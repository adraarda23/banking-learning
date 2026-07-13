# Topic 6.7 — Saga Pattern

## Hedef

Distributed transaction problemini Saga pattern ile çözmek. 2PC'nin neden production banking için yetersiz olduğunu anlamak. Orchestration vs choreography seçimi, compensating action design, banking cross-bank transfer örneği, Spring State Machine ile orchestrator implementation.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 6.1-6.6 bitti (Kafka, producer, consumer, Streams, Outbox)
- Phase 2 (transactions) + Phase 4 (locking) — ACID kavramları
- Phase 7'de microservice decomposition gelecek — burası temelini atar

---

## Kavramlar

### 1. Distributed transaction problemi

**Senaryo:** Cross-bank transfer (TR Bank A → German Bank B).
1. Bank A'da debit (TRY hesap)
2. FX conversion
3. Bank B'de credit (EUR hesap)
4. SWIFT confirmation
5. Audit log

Tek DB'de transaction: `BEGIN; ... COMMIT` — atomic. Kolay.

**Microservice'lerde:**
- Bank A → kendi DB
- FX service → kendi state
- Bank B → uzak DB (başka kurum)
- SWIFT → external network
- Audit → kendi DB

5 farklı transactional boundary. Atomicity nasıl?

### 2. 2PC (Two-Phase Commit) — neden banking için yasak

```
Phase 1 (Prepare):
  Coordinator → Participant 1: "hazır mısın?"
  Coordinator → Participant 2: "hazır mısın?"
  Coordinator → Participant 3: "hazır mısın?"
  Coordinator: hepsi "evet" derse → Phase 2'ye geç

Phase 2 (Commit):
  Coordinator → Hepsi: "COMMIT"
  Veya birinden "hayır" → Hepsi: "ROLLBACK"
```

**XA distributed transaction** ile implement edilir.

**Sorunlar (banking için fatal):**

1. **Blocking:** Phase 1'de tüm participant'lar **kaynakları kilitler** (DB row lock). Phase 2 commit gelene kadar **bekler**.

2. **Coordinator SPOF:** Coordinator Phase 1 sonrası **çöker**. Participant'lar **uncertain state**'te kalır. Manuel müdahale.

3. **Performance:** Network roundtrip × N participant. Banking transfer'ı 5 saniye değil 30 saniye.

4. **Vendor lock-in:** XA destekleyen DB + transaction manager şart. Cloud-native değil.

5. **Scalability:** Coordinator throughput bottleneck.

**Banking pratiği:** 2PC modern banking'de **YASAK**. Eski legacy sistemlerde var, yenileri Saga.

### 3. Saga pattern — sequence of local transactions

**Prensip:**
- Distributed transaction'ı **N adıma böl**
- Her adım **local transaction** (atomic, fast)
- Bir adım fail → **compensating action** ile öncekileri **geri al**

**Banking — cross-bank transfer:**

```
Step 1: Bank A'da debit                         (local tx, ~50ms)
  ↓ success
Step 2: FX conversion (TCMB rate uygula)         (~10ms)
  ↓ success
Step 3: Bank B'de credit                         (~50ms via SWIFT)
  ↓ success
Step 4: Audit log                                (~10ms)
  ↓ success
Step 5: Notification                             (~20ms)

= Total ~140ms, no blocking lock
```

**Failure senaryosu — Step 3 fail:**

```
Step 1: Bank A'da debit                         ✓
Step 2: FX conversion                            ✓
Step 3: Bank B'de credit                         ✗ FAIL
  ↓ compensate
Compensate Step 2: FX cancel (no-op or logged)
Compensate Step 1: Bank A'ya geri credit (reverse debit)
```

Net effect: Sistem **eski state'e döndü**. Audit trail var (Step 1 + reversal).

### 4. Orchestration — central coordinator

```
[Saga Orchestrator] → Bank A: debit
[Saga Orchestrator] ← debit OK
[Saga Orchestrator] → FX: convert
[Saga Orchestrator] ← FX rate
[Saga Orchestrator] → Bank B: credit
[Saga Orchestrator] ← Bank B FAIL
[Saga Orchestrator] → FX: cancel
[Saga Orchestrator] → Bank A: reverse debit
[Saga Orchestrator] → Audit: log compensated
```

**State machine** — her adım state, her transition action.

```
STARTED → DEBITED → CONVERTED → CREDITED → COMPLETED
                     ↓ fail
                  COMPENSATING → COMPENSATED
```

#### Avantajlar

- **Visibility:** State machine'i UI'da görür, debug kolay
- **Recovery:** State DB'de, orchestrator crash → restart kaldığı yerden
- **Centralized logic:** Saga flow tek yerde

#### Dezavantajlar

- **Coupling:** Orchestrator tüm service'leri bilmek zorunda
- **SPOF:** Orchestrator down → saga durur (HA replica gerekli)
- **Stateful:** State persistence gerekli

### 5. Spring State Machine ile orchestration

```java
public enum TransferState {
    STARTED, DEBITED, CONVERTED, CREDITED, COMPLETED,
    COMPENSATING, COMPENSATED, FAILED
}

public enum TransferEvent {
    DEBIT_OK, DEBIT_FAILED,
    CONVERT_OK, CONVERT_FAILED,
    CREDIT_OK, CREDIT_FAILED,
    COMPLETE, FAIL,
    COMPENSATE_OK
}

@Configuration
@EnableStateMachineFactory
public class TransferSagaConfig extends StateMachineConfigurerAdapter<TransferState, TransferEvent> {
    
    @Override
    public void configure(StateMachineStateConfigurer<TransferState, TransferEvent> states) throws Exception {
        states.withStates()
            .initial(TransferState.STARTED)
            .state(TransferState.DEBITED, debitedAction())
            .state(TransferState.CONVERTED, convertedAction())
            .state(TransferState.CREDITED, creditedAction())
            .state(TransferState.COMPENSATING, compensateAction())
            .end(TransferState.COMPLETED)
            .end(TransferState.COMPENSATED)
            .end(TransferState.FAILED);
    }
    
    @Override
    public void configure(StateMachineTransitionConfigurer<TransferState, TransferEvent> transitions) throws Exception {
        transitions
            // Happy path
            .withExternal()
                .source(TransferState.STARTED).target(TransferState.DEBITED).event(TransferEvent.DEBIT_OK)
            .and().withExternal()
                .source(TransferState.DEBITED).target(TransferState.CONVERTED).event(TransferEvent.CONVERT_OK)
            .and().withExternal()
                .source(TransferState.CONVERTED).target(TransferState.CREDITED).event(TransferEvent.CREDIT_OK)
            .and().withExternal()
                .source(TransferState.CREDITED).target(TransferState.COMPLETED).event(TransferEvent.COMPLETE)
            
            // Compensation paths
            .and().withExternal()
                .source(TransferState.STARTED).target(TransferState.FAILED).event(TransferEvent.DEBIT_FAILED)
            .and().withExternal()
                .source(TransferState.DEBITED).target(TransferState.COMPENSATING).event(TransferEvent.CONVERT_FAILED)
            .and().withExternal()
                .source(TransferState.CONVERTED).target(TransferState.COMPENSATING).event(TransferEvent.CREDIT_FAILED)
            .and().withExternal()
                .source(TransferState.COMPENSATING).target(TransferState.COMPENSATED).event(TransferEvent.COMPENSATE_OK);
    }
    
    @Bean
    public Action<TransferState, TransferEvent> debitedAction() {
        return context -> {
            TransferContext ctx = context.getExtendedState().get("ctx", TransferContext.class);
            log.info("Saga DEBITED for transfer {}", ctx.getTransferId());
            // Next step: FX conversion request
            fxService.requestConversionAsync(ctx);
        };
    }
    
    @Bean
    public Action<TransferState, TransferEvent> compensateAction() {
        return context -> {
            TransferContext ctx = context.getExtendedState().get("ctx", TransferContext.class);
            
            // Determine compensation steps based on current state
            switch (ctx.getCurrentState()) {
                case CREDITED:
                    // Step 3 fail değil ama subsequent fail. Step 3 başarılı, geri al
                    bankBService.reverseCredit(ctx.getBankBTransactionId());
                    fxService.cancelConversion(ctx.getFxConversionId());
                    bankAService.reverseDebit(ctx.getBankATransactionId());
                    break;
                case CONVERTED:
                    fxService.cancelConversion(ctx.getFxConversionId());
                    bankAService.reverseDebit(ctx.getBankATransactionId());
                    break;
                case DEBITED:
                    bankAService.reverseDebit(ctx.getBankATransactionId());
                    break;
            }
            
            sagaStateRepo.markCompensated(ctx.getSagaId());
        };
    }
}
```

### 6. Saga state persistence

State machine state DB'de saklanmalı. Orchestrator crash → restart → state'ten kaldığı yerden.

```sql
CREATE TABLE saga_states (
    saga_id UUID PRIMARY KEY,
    saga_type VARCHAR(50) NOT NULL,
    current_state VARCHAR(50) NOT NULL,
    transfer_id UUID,
    context JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    
    CONSTRAINT chk_saga_state CHECK (current_state IN (
        'STARTED', 'DEBITED', 'CONVERTED', 'CREDITED', 'COMPLETED',
        'COMPENSATING', 'COMPENSATED', 'FAILED'
    ))
);

CREATE INDEX idx_saga_pending ON saga_states(current_state, updated_at) 
    WHERE current_state NOT IN ('COMPLETED', 'COMPENSATED', 'FAILED');

CREATE INDEX idx_saga_stuck ON saga_states(updated_at) 
    WHERE current_state NOT IN ('COMPLETED', 'COMPENSATED', 'FAILED');
```

`stuck` index — recovery scheduler için (Section 9).

### 7. Choreography — event-driven

Her servis event yayınlar, diğerleri dinler, kendi adımını yapar, event yayınlar.

```
Bank A: TransferRequested al
       → debit
       → DebitedEvent yayınla (Kafka)

FX:    DebitedEvent al
       → convert
       → ConvertedEvent yayınla

Bank B: ConvertedEvent al
       → credit
       → CreditedEvent (success) veya CreditFailedEvent (fail) yayınla

Bank A: CreditFailedEvent al
       → reverseDebit
       → DebitReversedEvent yayınla
```

#### Avantajlar

- **Decoupling:** Orchestrator yok, servisler bağımsız
- **Scalability:** Linear
- **No SPOF:** Coordinator yok

#### Dezavantajlar

- **Visibility:** Flow dağınık (event'leri trace etmek zor)
- **Debugging:** Tracing/observability şart
- **Cyclic complexity:** Event'ler arası bağımlılık karmaşıklaşır

### 8. Orchestration vs Choreography — karar matrisi

| Kriter | Orchestration | Choreography |
|---|---|---|
| Flow visibility | Yüksek (state machine) | Düşük (dağınık event'ler) |
| Service coupling | Orchestrator central | Düşük |
| Recovery | State machine'den | Event sourcing veya replay |
| Debugging | Kolay | Distributed tracing şart |
| SPOF | Orchestrator | Yok |
| Scalability | Coordinator bottleneck | Linear |
| Banking adoption | TR bankalarında daha yaygın | Modern startup'larda |

**Banking pratiği:**
- **Mission-critical** cross-bank transfer → orchestration (visibility kritik)
- **Internal events** (audit, notification) → choreography (decoupling)

Mixed approach yaygın.

### 9. Compensating action — kurallar

**Compensating action 5 kuralı:**

#### 1. Idempotent

```java
public void reverseDebit(UUID transactionId) {
    // İdempotent — 2 kez çağrılsa aynı sonuç
    if (reversedTxRepo.existsByOriginalTxId(transactionId)) {
        log.warn("Already reversed: {}", transactionId);
        return;
    }
    
    BankATransaction original = bankATxRepo.findById(transactionId).orElseThrow();
    bankAService.credit(original.getAccountId(), original.getAmount());
    reversedTxRepo.save(new ReversedTransaction(transactionId, Instant.now()));
}
```

Network retry, partial failure → 2x compensate → state bozulmasın.

#### 2. Commutative (mümkünse)

Compensation order esnek olmalı. Strict sıra şart değilse cleanup paralel olabilir.

#### 3. Local atomic

Her compensation kendi tx'inde. Compensation'ın kendisi fail edebilir → outer compensation gerekirse.

#### 4. Audit trail

```java
public void reverseDebit(UUID transactionId) {
    // ...
    auditService.log(AuditEvent.builder()
        .action("DEBIT_REVERSED")
        .resourceId(transactionId)
        .reason("Saga compensation")
        .build());
}
```

Compensation'ı sil **etme**. Audit için kayıt.

#### 5. Semantic compensation

Bazen "geri alma" tam mümkün değil. **Semantic compensation**: yeni adım at, eski'yi telafi et.

Banking örnek: SMS gönderildi. SMS'i "geri al" mümkün değil. **Compensation:** ikinci SMS gönder — "Önceki SMS hatalıydı, transfer iptal edildi."

### 10. Banking örnek — Cross-bank Transfer Saga

```java
@Service
@Slf4j
public class CrossBankTransferSaga {
    
    private final KafkaTemplate<String, Object> kafka;
    private final SagaStateRepository sagaRepo;
    private final TransferContextSerializer serializer;
    
    // Entry point — orchestrator init
    public UUID initiate(CrossBankTransferRequest req) {
        UUID sagaId = UUID.randomUUID();
        
        TransferContext context = TransferContext.builder()
            .sagaId(sagaId)
            .transferRequest(req)
            .build();
        
        SagaState state = SagaState.builder()
            .sagaId(sagaId)
            .sagaType("CROSS_BANK_TRANSFER")
            .currentState("STARTED")
            .context(serializer.toJson(context))
            .createdAt(Instant.now())
            .updatedAt(Instant.now())
            .build();
        sagaRepo.save(state);
        
        // Step 1: Bank A debit
        kafka.send("bank-a.debit-requested", sagaId.toString(),
            new DebitRequest(sagaId, req.getFromAccount(), req.getAmount()));
        
        log.info("Saga initiated: id={}, from={}, to={}, amount={}",
            sagaId, req.getFromAccount(), req.getToAccount(), req.getAmount());
        
        return sagaId;
    }
    
    @KafkaListener(topics = "bank-a.debit-completed", groupId = "saga-orchestrator")
    @Transactional
    public void onDebitCompleted(DebitCompletedEvent event, Acknowledgment ack) {
        SagaState state = sagaRepo.findById(event.getSagaId()).orElseThrow();
        TransferContext ctx = serializer.fromJson(state.getContext());
        
        ctx.setBankATransactionId(event.getTransactionId());
        state.setContext(serializer.toJson(ctx));
        state.setCurrentState("DEBITED");
        state.setUpdatedAt(Instant.now());
        sagaRepo.save(state);
        
        // Step 2: FX conversion
        kafka.send("fx.convert-requested", event.getSagaId().toString(),
            new ConvertRequest(event.getSagaId(), ctx.getTransferRequest().getCurrency(), 
                              "EUR", ctx.getTransferRequest().getAmount()));
        
        ack.acknowledge();
    }
    
    @KafkaListener(topics = "bank-a.debit-failed", groupId = "saga-orchestrator")
    @Transactional
    public void onDebitFailed(DebitFailedEvent event, Acknowledgment ack) {
        SagaState state = sagaRepo.findById(event.getSagaId()).orElseThrow();
        state.setCurrentState("FAILED");
        state.setUpdatedAt(Instant.now());
        state.setCompletedAt(Instant.now());
        sagaRepo.save(state);
        
        // No compensation — Step 1 zaten failed, geri alacak bir şey yok
        
        // Müşteri bilgilendir
        notifyCustomer(event.getSagaId(), "DEBIT_FAILED", event.getReason());
        
        ack.acknowledge();
    }
    
    @KafkaListener(topics = "fx.convert-completed", groupId = "saga-orchestrator")
    @Transactional
    public void onConvertCompleted(ConvertCompletedEvent event, Acknowledgment ack) {
        SagaState state = sagaRepo.findById(event.getSagaId()).orElseThrow();
        TransferContext ctx = serializer.fromJson(state.getContext());
        
        ctx.setFxConversionId(event.getConversionId());
        ctx.setConvertedAmount(event.getConvertedAmount());
        state.setContext(serializer.toJson(ctx));
        state.setCurrentState("CONVERTED");
        state.setUpdatedAt(Instant.now());
        sagaRepo.save(state);
        
        // Step 3: Bank B credit
        kafka.send("bank-b.credit-requested", event.getSagaId().toString(),
            new CreditRequest(event.getSagaId(), ctx.getTransferRequest().getToAccount(), 
                            event.getConvertedAmount(), "EUR"));
        
        ack.acknowledge();
    }
    
    @KafkaListener(topics = "fx.convert-failed", groupId = "saga-orchestrator")
    @Transactional
    public void onConvertFailed(ConvertFailedEvent event, Acknowledgment ack) {
        SagaState state = sagaRepo.findById(event.getSagaId()).orElseThrow();
        TransferContext ctx = serializer.fromJson(state.getContext());
        
        state.setCurrentState("COMPENSATING");
        state.setUpdatedAt(Instant.now());
        sagaRepo.save(state);
        
        // Compensate Step 1
        kafka.send("bank-a.reverse-debit-requested", event.getSagaId().toString(),
            new ReverseDebitRequest(event.getSagaId(), ctx.getBankATransactionId()));
        
        ack.acknowledge();
    }
    
    @KafkaListener(topics = "bank-b.credit-completed", groupId = "saga-orchestrator")
    @Transactional
    public void onCreditCompleted(CreditCompletedEvent event, Acknowledgment ack) {
        SagaState state = sagaRepo.findById(event.getSagaId()).orElseThrow();
        TransferContext ctx = serializer.fromJson(state.getContext());
        
        ctx.setBankBTransactionId(event.getTransactionId());
        state.setContext(serializer.toJson(ctx));
        state.setCurrentState("COMPLETED");
        state.setUpdatedAt(Instant.now());
        state.setCompletedAt(Instant.now());
        sagaRepo.save(state);
        
        // Async: audit + notification (no need to wait)
        kafka.send("audit.log-requested", event.getSagaId().toString(),
            new AuditLogRequest("CROSS_BANK_TRANSFER_COMPLETED", ctx));
        
        kafka.send("notification.send-requested", event.getSagaId().toString(),
            new NotificationRequest(ctx.getTransferRequest().getCustomerId(), 
                                   "Transfer completed", ctx));
        
        log.info("Saga completed: id={}", event.getSagaId());
        ack.acknowledge();
    }
    
    @KafkaListener(topics = "bank-b.credit-failed", groupId = "saga-orchestrator")
    @Transactional
    public void onCreditFailed(CreditFailedEvent event, Acknowledgment ack) {
        SagaState state = sagaRepo.findById(event.getSagaId()).orElseThrow();
        TransferContext ctx = serializer.fromJson(state.getContext());
        
        state.setCurrentState("COMPENSATING");
        state.setUpdatedAt(Instant.now());
        sagaRepo.save(state);
        
        log.warn("Credit failed, starting compensation: saga={}, reason={}",
            event.getSagaId(), event.getReason());
        
        // Compensate: Step 2 + Step 1
        kafka.send("fx.cancel-requested", event.getSagaId().toString(),
            new CancelConversionRequest(event.getSagaId(), ctx.getFxConversionId()));
        
        kafka.send("bank-a.reverse-debit-requested", event.getSagaId().toString(),
            new ReverseDebitRequest(event.getSagaId(), ctx.getBankATransactionId()));
        
        ack.acknowledge();
    }
    
    @KafkaListener(topics = "bank-a.reverse-debit-completed", groupId = "saga-orchestrator")
    @Transactional
    public void onReverseDebitCompleted(ReverseDebitCompletedEvent event, Acknowledgment ack) {
        SagaState state = sagaRepo.findById(event.getSagaId()).orElseThrow();
        state.setCurrentState("COMPENSATED");
        state.setUpdatedAt(Instant.now());
        state.setCompletedAt(Instant.now());
        sagaRepo.save(state);
        
        notifyCustomer(event.getSagaId(), "COMPENSATED", "Transfer iptal edildi, paranız iade edildi.");
        
        log.info("Saga compensated: id={}", event.getSagaId());
        ack.acknowledge();
    }
}
```

### 11. Timeout handling — stuck saga recovery

Bir step bekleniyor ama event gelmiyor → timeout.

```java
@Scheduled(fixedDelay = 60000)   // her dakika
@SchedulerLock(name = "stuckSagaCheck")
public void checkStuckSagas() {
    Duration timeout = Duration.ofMinutes(5);
    Instant cutoff = Instant.now().minus(timeout);
    
    List<SagaState> stuck = sagaRepo.findStuck(cutoff);
    
    for (SagaState saga : stuck) {
        log.warn("Stuck saga: id={}, state={}, age={}min", 
            saga.getSagaId(), saga.getCurrentState(),
            Duration.between(saga.getUpdatedAt(), Instant.now()).toMinutes());
        
        // Trigger compensation
        triggerCompensation(saga);
    }
}

private void triggerCompensation(SagaState saga) {
    TransferContext ctx = serializer.fromJson(saga.getContext());
    
    // Force compensation event
    switch (saga.getCurrentState()) {
        case "DEBITED":
            kafka.send("bank-a.reverse-debit-requested", saga.getSagaId().toString(),
                new ReverseDebitRequest(saga.getSagaId(), ctx.getBankATransactionId()));
            break;
        case "CONVERTED":
            kafka.send("fx.cancel-requested", ...);
            kafka.send("bank-a.reverse-debit-requested", ...);
            break;
        case "CREDITED":
            kafka.send("bank-b.reverse-credit-requested", ...);
            kafka.send("fx.cancel-requested", ...);
            kafka.send("bank-a.reverse-debit-requested", ...);
            break;
    }
    
    saga.setCurrentState("COMPENSATING");
    saga.setUpdatedAt(Instant.now());
    sagaRepo.save(saga);
    
    alerts.notifyOps("Stuck saga compensation triggered: " + saga.getSagaId());
}
```

Banking pratiği: Stuck timeout 5-15 dakika. SLA'nına göre ayarla.

### 12. TCC (Try-Confirm-Cancel) — alternatif pattern

3-phase:
1. **Try:** Kaynak reserve et (hold)
2. **Confirm:** Reserved'ı finalize
3. **Cancel:** Reserved'ı release

```java
// Try
String holdId = accountService.holdBalance(fromId, amount);

// Confirm (saga step OK)
accountService.confirmHold(holdId);

// Cancel (saga step FAIL)
accountService.releaseHold(holdId);
```

Banking örnek — kart authorization:

```
Authorization (Try): Kartta amount hold
Capture (Confirm): Held → actual debit
Void (Cancel): Held → release
```

TCC sıkı consistency. Saga'dan **daha rigid**, ama daha karmaşık.

Banking adoption: Booking/reservation pattern (otel, uçak) → TCC. Banking transfer → Saga.

### 13. Saga monitoring + observability

Banking için saga'nın **görünür** olması kritik.

```java
@Component
public class SagaMetrics {
    
    private final MeterRegistry registry;
    
    public void recordTransition(String sagaType, String fromState, String toState) {
        registry.counter("saga.transition",
            "type", sagaType,
            "from", fromState,
            "to", toState).increment();
    }
    
    public void recordDuration(String sagaType, String finalState, Duration duration) {
        registry.timer("saga.duration",
            "type", sagaType,
            "outcome", finalState).record(duration);
    }
}
```

Grafana dashboard:
- Saga success rate
- Saga p99 duration
- State distribution (kaç saga her state'te)
- Stuck saga count

OpenTelemetry trace ile end-to-end visibility (Phase 7 + 9).

### 14. Banking anti-pattern'leri

**Anti-pattern 1: 2PC kullanmak**

```java
@Transactional("xaTransactionManager")   // ❌ XA distributed tx
public void transfer() {
    bankAService.debit(...);   // XA participant
    bankBService.credit(...);   // XA participant
}
```

Blocking, SPOF, performans katil. Banking için **YASAK**. Saga kullan.

**Anti-pattern 2: Saga state in-memory**

```java
private Map<UUID, SagaState> states = new ConcurrentHashMap<>();   // ❌
```

App restart → saga kayıp. DB-persistent şart.

**Anti-pattern 3: Compensation idempotent değil**

```java
public void reverseDebit(UUID txId) {
    BankATransaction tx = bankATxRepo.findById(txId);
    bankAService.credit(tx.getAccount(), tx.getAmount());   // duplicate compensation → double credit
}
```

Duplicate compensation → state bozulur. Idempotent şart.

**Anti-pattern 4: Saga timeout yok**

Bir step takıldı → saga sonsuza kadar PENDING. Stuck monitoring + auto-compensation.

**Anti-pattern 5: Saga'yı sync HTTP yapmak**

```java
public Transfer execute(...) {
    bankAService.debit(...);   // 1 saniye blocking call
    fxService.convert(...);    // 1 saniye
    bankBService.credit(...);  // 1 saniye
    auditService.log(...);     // 500ms
    notifService.send(...);    // 1 saniye
    return transfer;           // 4.5+ saniye sync — kullanıcı bekliyor
}
```

User ekranda 5 saniye bekler, timeout riski. Saga **async** olmalı. HTTP endpoint:

```java
@PostMapping("/cross-bank-transfers")
public ResponseEntity<SagaCreatedResponse> initiate(@RequestBody CrossBankTransferRequest req) {
    UUID sagaId = crossBankSaga.initiate(req);
    return ResponseEntity.accepted()   // 202 Accepted
        .body(new SagaCreatedResponse(sagaId, "/sagas/" + sagaId));
}

@GetMapping("/sagas/{id}")
public SagaStatusResponse getStatus(@PathVariable UUID id) {
    SagaState state = sagaRepo.findById(id).orElseThrow();
    return new SagaStatusResponse(state.getCurrentState(), ...);
}
```

Async pattern + status polling. Banking standard.

**Anti-pattern 6: Compensation order yanlış**

Step 3 fail → compensate Step 1, sonra Step 2.

**Yanlış sıra.** Step 2 (FX) önce iptal, sonra Step 1 (debit). Reverse order: son'dan ilk'e doğru.

**Anti-pattern 7: Saga event'leri non-idempotent topic'e**

Event 2 kez geldi → handler duplicate işlem. Idempotent consumer pattern (Topic 6.3).

---

## Önemli olabilecek araştırma kaynakları

- "Microservices Patterns" (Chris Richardson) — Saga chapter
- "Designing Data-Intensive Applications" (Kleppmann) — distributed transaction
- Vaughn Vernon — implementing DDD with saga
- Eventuate Tram framework documentation
- Spring State Machine reference

---

## Mini task'ler

### Task 6.7.1 — Saga state DB schema (15 dk)

`saga_states` migration. Yukarıdaki schema.

### Task 6.7.2 — Cross-bank transfer orchestrator (90 dk)

`CrossBankTransferSaga` class'ı:
- `initiate()` entry point
- 6 @KafkaListener (debit OK, debit fail, convert OK, convert fail, credit OK, credit fail)
- Compensation listener'lar
- State machine geçişleri

Test happy path:
1. `initiate()` çağır
2. `bank-a.debit-completed` event simulate et
3. `fx.convert-completed` event simulate et
4. `bank-b.credit-completed` event simulate et
5. SagaState COMPLETED olmalı

### Task 6.7.3 — Compensation flow (60 dk)

Test compensation:
1. `initiate()`
2. `bank-a.debit-completed` ✓
3. `bank-b.credit-failed` ✗
4. Beklenenler:
   - `fx.cancel-requested` event yayınlandı
   - `bank-a.reverse-debit-requested` event yayınlandı
   - SagaState COMPENSATING → COMPENSATED

### Task 6.7.4 — Stuck saga recovery (45 dk)

5 dakika eski PENDING saga → recovery scheduler trigger compensation. Test: saga'yı eski timestamp ile DB'ye koy, scheduler çalıştır, COMPENSATING'e geçtiğini doğrula.

### Task 6.7.5 — TCC pattern denemesi (60 dk)

`account.holdBalance(amount)` → holdId. `confirmHold(holdId)` veya `releaseHold(holdId)`. Saga step → TCC try → confirm/cancel.

### Task 6.7.6 — Spring State Machine ile (60 dk)

Yukarıdaki state machine config'i ile orchestrator. Event'leri state machine'e gönder, transition'ları action'lar tetikle.

### Task 6.7.7 — Saga monitoring + Grafana (30 dk)

`SagaMetrics` ile transition + duration metric'leri. Grafana panel: success rate, p99 duration, stuck count.

---

## Test yazma rehberi

### Test 6.7.1 — Happy path

```java
@SpringBootTest
@Testcontainers
class CrossBankTransferSagaIT {
    
    @Container @ServiceConnection static PostgreSQLContainer<?> postgres = ...;
    @Container static KafkaContainer kafka = ...;
    
    @Autowired CrossBankTransferSaga saga;
    @Autowired SagaStateRepository sagaRepo;
    @Autowired KafkaTemplate<String, Object> template;
    
    @Test
    void shouldCompleteSuccessfulSaga() throws Exception {
        CrossBankTransferRequest req = createValidRequest();
        UUID sagaId = saga.initiate(req);
        
        // Simulate Bank A debit success
        template.send("bank-a.debit-completed", sagaId.toString(),
            new DebitCompletedEvent(sagaId, UUID.randomUUID()));
        
        await().atMost(5, SECONDS).untilAsserted(() -> {
            SagaState state = sagaRepo.findById(sagaId).orElseThrow();
            assertThat(state.getCurrentState()).isEqualTo("DEBITED");
        });
        
        // Simulate FX convert success
        template.send("fx.convert-completed", sagaId.toString(),
            new ConvertCompletedEvent(sagaId, UUID.randomUUID(), new BigDecimal("250.00")));
        
        await().atMost(5, SECONDS).untilAsserted(() -> {
            SagaState state = sagaRepo.findById(sagaId).orElseThrow();
            assertThat(state.getCurrentState()).isEqualTo("CONVERTED");
        });
        
        // Simulate Bank B credit success
        template.send("bank-b.credit-completed", sagaId.toString(),
            new CreditCompletedEvent(sagaId, UUID.randomUUID()));
        
        await().atMost(5, SECONDS).untilAsserted(() -> {
            SagaState state = sagaRepo.findById(sagaId).orElseThrow();
            assertThat(state.getCurrentState()).isEqualTo("COMPLETED");
        });
    }
}
```

### Test 6.7.2 — Compensation

```java
@Test
void shouldCompensateOnCreditFailure() throws Exception {
    UUID sagaId = saga.initiate(createValidRequest());
    
    // Debit + Convert success
    template.send("bank-a.debit-completed", sagaId.toString(), new DebitCompletedEvent(...));
    template.send("fx.convert-completed", sagaId.toString(), new ConvertCompletedEvent(...));
    
    await().atMost(5, SECONDS).untilAsserted(() ->
        assertThat(sagaRepo.findById(sagaId).orElseThrow().getCurrentState()).isEqualTo("CONVERTED"));
    
    // Credit fail
    template.send("bank-b.credit-failed", sagaId.toString(),
        new CreditFailedEvent(sagaId, "Insufficient funds at Bank B"));
    
    await().atMost(5, SECONDS).untilAsserted(() ->
        assertThat(sagaRepo.findById(sagaId).orElseThrow().getCurrentState()).isEqualTo("COMPENSATING"));
    
    // Compensation completes
    template.send("bank-a.reverse-debit-completed", sagaId.toString(),
        new ReverseDebitCompletedEvent(sagaId));
    
    await().atMost(5, SECONDS).untilAsserted(() -> {
        SagaState state = sagaRepo.findById(sagaId).orElseThrow();
        assertThat(state.getCurrentState()).isEqualTo("COMPENSATED");
        assertThat(state.getCompletedAt()).isNotNull();
    });
}
```

### Test 6.7.3 — Stuck saga recovery

```java
@Test
void stuckSagaShouldTriggerCompensation() {
    SagaState stuck = SagaState.builder()
        .sagaId(UUID.randomUUID())
        .sagaType("CROSS_BANK_TRANSFER")
        .currentState("DEBITED")
        .updatedAt(Instant.now().minus(10, ChronoUnit.MINUTES))   // 10 dk eski
        .build();
    sagaRepo.save(stuck);
    
    stuckSagaScheduler.checkStuckSagas();
    
    await().atMost(5, SECONDS).untilAsserted(() -> {
        SagaState updated = sagaRepo.findById(stuck.getSagaId()).orElseThrow();
        assertThat(updated.getCurrentState()).isEqualTo("COMPENSATING");
    });
}
```

### Test 6.7.4 — Idempotent compensation

```java
@Test
void duplicateCompensationShouldNotDoubleReverse() {
    UUID txId = setupDebitedTransaction();
    
    bankAService.reverseDebit(txId);
    bankAService.reverseDebit(txId);   // duplicate call
    
    // Account balance sadece bir kez reversed
    BigDecimal finalBalance = accountRepo.getBalance(testAccountId);
    assertThat(finalBalance).isEqualByComparingTo(initialBalance);   // not 2x
}
```

---

## Claude-verify prompt

```
Saga pattern kodumu banking-grade kriterlere göre değerlendir:

1. 2PC kullanılmamış mı?
   - XA transaction manager YOK?
   - Saga pattern tercih edilmiş?

2. Saga state persistence:
   - saga_states tablosu DB'de?
   - State machine değişimleri DB'ye yazılıyor mu?
   - Restart sonrası state recovery mümkün mü?

3. Orchestration vs choreography:
   - Mission-critical (cross-bank) için orchestration tercih edilmiş mi?
   - Internal events için choreography uygun mu?

4. Compensation:
   - Tüm compensation action idempotent mi?
   - Audit trail compensation log'lanıyor mu?
   - Compensation order doğru (reverse order)?

5. Timeout handling:
   - Stuck saga detection scheduler var mı?
   - Auto-compensation trigger?
   - Ops alert?

6. Async pattern:
   - HTTP endpoint 202 Accepted dönüyor mu?
   - Status polling endpoint var mı?
   - Sync HTTP saga (5+ saniye) anti-pattern'i YOK?

7. Banking specific:
   - Cross-bank transfer saga 4-5 step orchestrated mi?
   - FX conversion middle step?
   - Bank A debit, Bank B credit compensation order doğru?

8. Spring State Machine veya custom orchestrator:
   - State transitions explicit?
   - Action'lar her transition'da?

9. Monitoring:
   - SagaMetrics counter + timer?
   - Stuck saga gauge?
   - Grafana dashboard?

10. Anti-pattern:
    - In-memory saga state?
    - Compensation idempotent değil?
    - Saga timeout yok?
    - Sync HTTP saga?
    - 2PC kullanılmış?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] saga_states tablosu migration
- [ ] CrossBankTransferSaga orchestrator (6 listener)
- [ ] Happy path test geçiyor
- [ ] Compensation path test geçiyor
- [ ] Stuck saga recovery scheduler
- [ ] Idempotent compensation actions
- [ ] Async API (202 Accepted + status polling)
- [ ] SagaMetrics + Grafana
- [ ] TCC pattern denemesi yapıldı
- [ ] Spring State Machine örneği

---

## Defter notları (10 madde)

1. "2PC banking için yetersiz olmasının 5 sebebi: ____."
2. "Saga vs 2PC karar matrisi: ____."
3. "Orchestration vs choreography karar kriterleri: ____."
4. "Compensating action 5 kuralı (idempotent, audit, vb.): ____."
5. "Semantic compensation banking örneği (SMS): ____."
6. "Cross-bank transfer saga state machine: ____."
7. "Saga state persistence neden DB-backed zorunlu: ____."
8. "Stuck saga timeout banking SLA için: ____."
9. "TCC vs Saga banking adoption (booking vs transfer): ____."
10. "Async HTTP 202 + status polling pattern banking için: ____."
