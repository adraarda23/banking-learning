# Topic 5.6 — Scheduling: @Scheduled, Quartz, ShedLock, K8s CronJob

## Hedef

Banking EOD job tetikleme stratejileri banking-grade derinlikte: Spring `@Scheduled` (basit), Quartz Scheduler (cluster-aware), ShedLock (multi-instance tek run), K8s CronJob (cloud-native), banking holiday calendar integration, time zone handling, failure recovery, observability.

## Süre

Okuma: 1.5 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 5.1-5.5 bitti
- Cron syntax
- TR business calendar (Topic 10.4)

---

## Kavramlar

### 1. Scheduling options matrix

| Option | Cluster-aware | Persistence | Banking use |
|---|---|---|---|
| `@Scheduled` | ❌ no | ❌ in-memory | Single instance only |
| `@Scheduled` + ShedLock | ✓ via DB lock | ❌ | Multi-instance Spring Boot |
| Quartz | ✓ cluster | ✓ JDBC store | Complex schedules, persistent |
| K8s CronJob | ✓ K8s | ✓ K8s API | Container-native |
| Airflow / Dagster | ✓ | ✓ | Complex DAG, banking ETL |

### 2. Spring @Scheduled — basic

```java
@Configuration
@EnableScheduling
public class BatchSchedulingConfig {
    
    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setThreadNamePrefix("banking-scheduler-");
        scheduler.initialize();
        return scheduler;
    }
}

@Component
public class BankingScheduler {
    
    private final JobLauncher jobLauncher;
    private final Job eodMasterJob;
    private final TrBusinessCalendar calendar;
    
    @Scheduled(cron = "0 30 23 * * *", zone = "Europe/Istanbul")
    public void runEodMasterJob() throws Exception {
        LocalDate today = LocalDate.now(ZoneId.of("Europe/Istanbul"));
        
        if (!calendar.isBusinessDay(today)) {
            log.info("Today {} is not a business day; skipping EOD", today);
            return;
        }
        
        JobParameters params = BankingJobParameters.forEodOnDate(today);
        JobExecution exec = jobLauncher.run(eodMasterJob, params);
        
        log.info("EOD master job {}: {}", exec.getId(), exec.getStatus());
    }
    
    @Scheduled(cron = "0 0 5 * * MON-FRI", zone = "Europe/Istanbul")
    public void checkFailedJobs() {
        // Recovery scheduler (Topic 5.3)
        scheduler.resumeStalledJobs();
    }
    
    @Scheduled(cron = "0 0 6 * * *", zone = "Europe/Istanbul")
    public void refreshSanctionsLists() {
        sanctionsService.downloadAndRefresh();
    }
    
    @Scheduled(cron = "0 0 7 1 * *", zone = "Europe/Istanbul")   // 1. of month, 07:00
    public void monthlyClose() throws Exception {
        jobLauncher.run(monthlyCloseJob, BankingJobParameters.forMonthEnd(LocalDate.now()));
    }
}
```

**Cron format (Spring):** `second minute hour day month day-of-week`

Examples:
- `0 30 23 * * *` — Every day 23:30
- `0 0 5 * * MON-FRI` — Weekdays 05:00
- `0 0 7 1 * *` — 1st of every month 07:00
- `0 */15 * * * *` — Every 15 minutes
- `0 0 22 ? * SAT` — Saturday 22:00 only

**Time zone critical for banking:**

```java
@Scheduled(cron = "0 30 23 * * *", zone = "Europe/Istanbul")
```

Banking working hours TR zone. UTC offset değişir (DST → no, TR yıl boyunca UTC+3). Always specify zone.

### 3. ShedLock — multi-instance safety

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.10.2</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.10.2</version>
</dependency>
```

```sql
CREATE TABLE shedlock(
    name VARCHAR(64) NOT NULL,
    lock_until TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NOT NULL,
    locked_by VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "PT6H")
public class ShedLockConfig {
    
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .withTableName("shedlock")
                .usingDbTime()   // Use DB time (not app time)
                .build());
    }
}

@Component
public class BankingScheduler {
    
    @Scheduled(cron = "0 30 23 * * *", zone = "Europe/Istanbul")
    @SchedulerLock(name = "eodMasterJob", 
                   lockAtMostFor = "PT6H",       // Max lock 6 hours
                   lockAtLeastFor = "PT1H")      // Min lock 1 hour
    public void runEodMasterJob() throws Exception {
        // ... only one instance executes
    }
}
```

**ShedLock behavior:**
- 3 pods schedule at 23:30 simultaneously
- All try DB lock acquisition
- First pod wins
- Others see "already locked" → skip
- `lockAtMostFor` → safety net (if lock-holder crashes, lock auto-expires)
- `lockAtLeastFor` → prevent rapid re-execution (clock skew defense)

**Banking pratiği:** Spring Boot K8s multi-replica → ShedLock şart.

### 4. Quartz Scheduler — complex needs

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

```yaml
spring:
  quartz:
    job-store-type: jdbc
    properties:
      org:
        quartz:
          scheduler:
            instanceName: BankingScheduler
            instanceId: AUTO
          jobStore:
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            driverDelegateClass: org.quartz.impl.jdbcjobstore.PostgreSQLDelegate
            tablePrefix: QRTZ_
            isClustered: true
            clusterCheckinInterval: 10000
          threadPool:
            threadCount: 10
```

```java
@Configuration
public class QuartzConfig {
    
    @Bean
    public JobDetail eodJobDetail() {
        return JobBuilder.newJob(EodQuartzJob.class)
            .withIdentity("eodMasterJob")
            .storeDurably()
            .build();
    }
    
    @Bean
    public Trigger eodTrigger(JobDetail eodJobDetail) {
        return TriggerBuilder.newTrigger()
            .forJob(eodJobDetail)
            .withIdentity("eodTrigger")
            .withSchedule(CronScheduleBuilder
                .cronSchedule("0 30 23 * * ?")
                .inTimeZone(TimeZone.getTimeZone("Europe/Istanbul"))
                .withMisfireHandlingInstructionFireAndProceed())
            .build();
    }
}

@DisallowConcurrentExecution
@PersistJobDataAfterExecution
public class EodQuartzJob extends QuartzJobBean {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job eodMasterJob;
    
    @Override
    protected void executeInternal(JobExecutionContext context) {
        try {
            JobParameters params = BankingJobParameters.forEodOnDate(LocalDate.now());
            jobLauncher.run(eodMasterJob, params);
        } catch (Exception e) {
            throw new JobExecutionException(e);
        }
    }
}
```

**Quartz strengths over ShedLock:**
- Persistent triggers (survive restart)
- Misfire handling (job time missed → catch-up)
- Listener API
- Calendar (exclude dates)
- Pause / resume API

**Banking için:** Complex orchestration → Quartz. Simple cron → @Scheduled + ShedLock.

### 5. Misfire handling — Quartz

Pod down at 23:30. Scheduled job missed.

```java
.withMisfireHandlingInstructionFireAndProceed()    // Run ASAP when back
// OR
.withMisfireHandlingInstructionDoNothing()         // Skip; next schedule
// OR
.withMisfireHandlingInstructionIgnoreMisfires()    // Stack up missed
```

Banking EOD: typically `DoNothing` (manual recovery preferred over auto-misfire).

### 6. K8s CronJob — cloud-native

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: banking-eod
  namespace: banking-batch
spec:
  schedule: "30 23 * * *"
  timeZone: "Europe/Istanbul"
  concurrencyPolicy: Forbid   # No overlap
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  startingDeadlineSeconds: 600   # Try for 10 min if controller down
  jobTemplate:
    spec:
      backoffLimit: 0    # No retry — banking idempotency concern
      activeDeadlineSeconds: 21600   # 6 hour timeout
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: Never
          serviceAccountName: banking-batch
          containers:
            - name: eod
              image: banking/eod-job:1.0.0
              command: ["java", "-jar", "/app.jar", "eodMasterJob"]
              env:
                - name: JOB_DATE
                  value: "$(date -d 'today' +%Y-%m-%d)"
              resources:
                requests:
                  memory: 2Gi
                  cpu: 1
                limits:
                  memory: 4Gi
                  cpu: 4
```

**K8s CronJob features:**
- Native to K8s
- No app-level scheduler infrastructure
- Resource isolated per execution
- Cluster-aware (only one Job created per schedule)
- History retention

**Banking trade-off:**
- K8s CronJob simpler — but no Spring Batch native
- Spring Batch features (restart, listeners) → Spring Boot app + ShedLock
- Banking common: K8s CronJob runs Spring Boot one-shot job

### 7. Banking calendar integration

```java
@Component
public class CalendarAwareScheduler {
    
    @Scheduled(cron = "0 30 23 * * *", zone = "Europe/Istanbul")
    @SchedulerLock(name = "eodMasterJob", lockAtMostFor = "PT6H")
    public void runIfBusinessDay() throws Exception {
        LocalDate today = LocalDate.now(ZoneId.of("Europe/Istanbul"));
        
        if (!calendar.isBusinessDay(today)) {
            log.info("Skip EOD — non-business day: {}", today);
            return;
        }
        
        jobLauncher.run(eodMasterJob, BankingJobParameters.forEodOnDate(today));
    }
}
```

Alternative: Quartz Calendar exclusion:

```java
@Bean
public Calendar bankHolidayCalendar() {
    HolidayCalendar cal = new HolidayCalendar();
    trCalendar.allHolidays(2024, 2025).forEach(date -> 
        cal.addExcludedDate(Date.from(date.atStartOfDay(zone).toInstant())));
    return cal;
}

@Bean
public Trigger eodTrigger(JobDetail eodJobDetail, Calendar bankHolidayCalendar) {
    return TriggerBuilder.newTrigger()
        .forJob(eodJobDetail)
        .modifiedByCalendar("bankHolidayCalendar")   // Skip holidays
        .withSchedule(CronScheduleBuilder.cronSchedule("0 30 23 * * ?"))
        .build();
}
```

### 8. Misfire + recovery patterns

```
23:30: Scheduled
23:30 (real): Pod evicted, no job started

Recovery options:
1. Misfire fire-and-proceed (Quartz): Run when pod back up
2. Morning recovery scheduler: 05:00 check pending dates
3. Manual operator runbook
```

Banking pratiği:
- Critical EOD → manual confirm + run
- Sanctions refresh → auto-misfire OK
- Statement gen → catch-up next morning

### 9. Observability scheduling

```java
@Component
public class SchedulerMetrics {
    
    private final MeterRegistry registry;
    
    public void recordJobTrigger(String jobName, LocalDate date, boolean executed, String reason) {
        registry.counter("banking.scheduler.triggers",
            "job", jobName,
            "executed", String.valueOf(executed),
            "skip_reason", reason != null ? reason : "")
            .increment();
    }
    
    public void recordJobDuration(String jobName, Duration duration, String status) {
        registry.timer("banking.scheduler.job.duration",
            "job", jobName,
            "status", status)
            .record(duration);
    }
}
```

Alert rules:
- Scheduled job not run for X hours → alert
- Failed scheduler retries threshold
- Holiday calendar staleness

### 10. Banking — scheduling anti-pattern'leri

**Anti-pattern 1: @Scheduled multi-instance without ShedLock**

3 pods → 3x execution = duplicate side effects.

**Anti-pattern 2: No time zone**

```java
@Scheduled(cron = "0 30 23 * * *")   // ❌ UTC default
```

EOD runs UTC 23:30 = TR 02:30 — wrong day boundary.

**Anti-pattern 3: Hardcoded holiday calendar**

```java
if (date.equals(LocalDate.of(2024, 1, 1))) skip();   // ❌
```

Dynamic calendar (Topic 10.4).

**Anti-pattern 4: lockAtMostFor too short**

```java
@SchedulerLock(name = "...", lockAtMostFor = "PT5M")
```

Job runs 6 hours, lock expires after 5 min → second pod also starts.

**Anti-pattern 5: lockAtLeastFor unset**

Clock skew: pod A runs, finishes 23:35. Pod B clock 23:30 → re-runs same.

**Anti-pattern 6: K8s concurrencyPolicy: Allow**

Overlap if previous didn't finish.

**Anti-pattern 7: No failure alerting**

Job failed silently. Monitor scheduler heartbeat + duration metric.

**Anti-pattern 8: Quartz tablePrefix conflict**

Multiple apps same DB → table prefix per app.

**Anti-pattern 9: @Scheduled fixedRate without @Async**

Scheduled methods on single thread. Long-running job blocks others. Async or ThreadPoolTaskScheduler.

**Anti-pattern 10: Trigger non-business day no skip**

EOD runs Sunday → empty data, but ops alert + audit pollution.

---

## Önemli olabilecek araştırma kaynakları

- Spring Boot @Scheduled docs
- ShedLock GitHub
- Quartz Scheduler docs
- K8s CronJob reference
- Apache Airflow (advanced DAG)

---

## Mini task'ler

### Task 5.6.1 — Spring @Scheduled banking job (30 dk)

EOD 23:30 cron. Europe/Istanbul zone. Job launcher trigger.

### Task 5.6.2 — Calendar-aware skip (30 dk)

isBusinessDay check before run. Test holiday skip.

### Task 5.6.3 — ShedLock multi-instance (60 dk)

2 pod simulation. shedlock table. Only one executes.

### Task 5.6.4 — Quartz JDBC cluster (60 dk)

Quartz config + JDBC store. Misfire handling.

### Task 5.6.5 — Quartz Calendar exclusion (45 dk)

TR holidays as Quartz Calendar.

### Task 5.6.6 — K8s CronJob (45 dk)

CronJob manifest. concurrencyPolicy: Forbid. backoffLimit: 0.

### Task 5.6.7 — Morning recovery scheduler (45 dk)

05:00 check failed/stopped jobs from yesterday → resume.

### Task 5.6.8 — Scheduler metrics (30 dk)

triggers + duration + skip metric. Grafana panel.

### Task 5.6.9 — Failure alert (30 dk)

Job failed → Slack notify. Grafana alert rule.

### Task 5.6.10 — Concurrent run prevention test (45 dk)

Try trigger same job twice → second waits / skips / fails.

---

## Test yazma rehberi

```java
@SpringBootTest
@ImportAutoConfiguration(SchedulingAutoConfiguration.class)
class SchedulerTest {
    
    @MockBean JobLauncher jobLauncher;
    @MockBean TrBusinessCalendar calendar;
    
    @Test
    void shouldRunOnBusinessDay() throws Exception {
        when(calendar.isBusinessDay(any())).thenReturn(true);
        
        scheduler.runEodMasterJob();
        
        verify(jobLauncher).run(any(), any());
    }
    
    @Test
    void shouldSkipOnHoliday() throws Exception {
        when(calendar.isBusinessDay(any())).thenReturn(false);
        
        scheduler.runEodMasterJob();
        
        verify(jobLauncher, never()).run(any(), any());
    }
    
    @Test
    void shedLockShouldPreventConcurrentExecution() throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(3);
        AtomicInteger executions = new AtomicInteger();
        
        for (int i = 0; i < 3; i++) {
            pool.submit(() -> {
                scheduler.runEodMasterJob();
                executions.incrementAndGet();
            });
        }
        
        pool.shutdown();
        pool.awaitTermination(60, TimeUnit.SECONDS);
        
        // Only one execution (others see lock + skip)
        assertThat(executions.get()).isLessThanOrEqualTo(1);
    }
}
```

---

## Claude-verify prompt

```
Scheduling setup'ımı banking-grade kriterlere göre değerlendir:

1. Option selection:
   - @Scheduled + ShedLock simple banking?
   - Quartz complex orchestration?
   - K8s CronJob cloud-native?

2. Time zone:
   - Europe/Istanbul explicit?
   - No UTC implicit?

3. Cluster safety:
   - ShedLock @SchedulerLock?
   - K8s CronJob concurrencyPolicy: Forbid?
   - Quartz isClustered: true?

4. Banking calendar:
   - isBusinessDay check?
   - Holiday skip?
   - Dynamic calendar (Topic 10.4)?

5. Lock config:
   - lockAtMostFor banking realistic (6 hours)?
   - lockAtLeastFor clock skew defense?

6. Misfire handling:
   - Misfire policy explicit?
   - Recovery scheduler morning?
   - Manual operator workflow?

7. Observability:
   - Scheduler trigger metric?
   - Duration metric?
   - Failure alert?
   - Grafana panel?

8. K8s native:
   - CronJob backoffLimit: 0 (banking idempotency)?
   - activeDeadlineSeconds set?
   - History retention?

9. Quartz specific:
   - JDBC job store?
   - Calendar exclusion banking holidays?
   - Listener API?

10. Anti-pattern:
    - Multi-instance without ShedLock YOK?
    - No time zone YOK?
    - Hardcoded holiday YOK?
    - Short lockAtMostFor YOK?
    - K8s concurrencyPolicy: Allow YOK?
    - Silent failure YOK?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] Spring @Scheduled cron banking
- [ ] Europe/Istanbul time zone
- [ ] Calendar-aware skip
- [ ] ShedLock multi-instance test
- [ ] Quartz JDBC cluster mode
- [ ] Quartz Calendar exclusion holidays
- [ ] K8s CronJob manifest
- [ ] Morning recovery scheduler
- [ ] Metrics + alert
- [ ] 6+ integration test

---

## Defter notları (10 madde)

1. "Scheduling options (@Scheduled / Quartz / K8s CronJob) banking selection matrix: ____."
2. "Spring @Scheduled cron + zone=Europe/Istanbul critical: ____."
3. "ShedLock @SchedulerLock multi-instance duplicate-prevention: ____."
4. "Quartz JDBC store cluster persistent misfire handling: ____."
5. "K8s CronJob concurrencyPolicy + backoffLimit + activeDeadlineSeconds banking: ____."
6. "Banking calendar isBusinessDay check (TR holidays + bayram): ____."
7. "lockAtMostFor + lockAtLeastFor clock skew + crash defense: ____."
8. "Morning recovery scheduler (05:00) failed/stopped job resume: ____."
9. "Misfire handling (fire-and-proceed vs DoNothing) banking pratik: ____."
10. "Anti-pattern (multi-instance no ShedLock + no zone + hardcoded holiday): ____."
