# Topic 8.2 — Authentication: UserDetails, Password Hashing, Policy, MFA

## Hedef

Spring Security 6 authentication mekanizmalarını banking-grade seviyede öğrenmek. BCrypt vs Argon2, password policy (BDDK uyumlu), brute force protection, MFA (TOTP + SMS OTP), session management, credential storage best practices. Banking için **müşteri authentication** ve **internal employee authentication** akışlarını tasarlayabilmek.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 8.1 (Spring Security Architecture) bitti — filter chain, AuthenticationManager
- Phase 1'in user/account model'i hazır
- Phase 7 (microservices) — auth-service Keycloak alternative düşünüldü

---

## Kavramlar

### 1. Authentication (AuthN) vs Authorization (AuthZ)

| Kavram | Anlamı | Banking örneği |
|---|---|---|
| **AuthN** | "Sen kimsin?" | Kullanıcı login (username + password + MFA) |
| **AuthZ** | "Ne yapabilirsin?" | Sadece kendi hesabını görebilir, başkasının hesabına eremez |

Bu topic AuthN'a odaklı. AuthZ Topic 8.4 + 8.5'te.

### 2. Spring Security authentication flow

```
1. Client → AuthenticationFilter (e.g. UsernamePasswordAuthenticationFilter)
2. Filter → AuthenticationManager
3. AuthenticationManager → AuthenticationProvider(s)
   - Provider tries each registered AuthenticationProvider
4. Provider → UserDetailsService.loadUserByUsername(...)
5. UserDetailsService → DB lookup → UserDetails (with hashed password)
6. Provider → PasswordEncoder.matches(rawPassword, hashedPassword)
7. Match → Authentication object (with authorities) → SecurityContext
   No match → AuthenticationException → 401
```

### 3. UserDetailsService — DB integration

```java
@Service
public class BankingUserDetailsService implements UserDetailsService {
    
    private final UserRepository userRepo;
    private final LoginAttemptService loginAttemptService;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // Banking: username = email veya phone veya TC No
        User user = userRepo.findByUsernameOrEmailOrPhone(username, username, username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));
        
        // Account status checks (banking critical)
        if (loginAttemptService.isBlocked(username)) {
            throw new LockedException("Account is locked due to too many failed attempts");
        }
        
        if (user.getStatus() == UserStatus.SUSPENDED) {
            throw new DisabledException("Account is suspended");
        }
        
        if (user.getPasswordExpiresAt() != null && user.getPasswordExpiresAt().isBefore(Instant.now())) {
            throw new CredentialsExpiredException("Password expired, must reset");
        }
        
        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPasswordHash())
            .authorities(mapAuthorities(user.getRoles(), user.getPermissions()))
            .accountExpired(false)
            .accountLocked(loginAttemptService.isBlocked(username))
            .credentialsExpired(false)
            .disabled(!user.isEnabled())
            .build();
    }
    
    private Collection<? extends GrantedAuthority> mapAuthorities(Set<Role> roles, Set<Permission> permissions) {
        Set<GrantedAuthority> authorities = new HashSet<>();
        for (Role role : roles) {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));
        }
        for (Permission perm : permissions) {
            authorities.add(new SimpleGrantedAuthority(perm.getName()));
        }
        return authorities;
    }
}
```

Banking detayları:
- `username` = email/phone/TC No (esnek)
- Status checks (locked, suspended, password expired)
- Role + Permission birleştirme (ROLE_customer, ACCOUNT_READ)

### 4. Password hashing — BCrypt (default)

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);
}
```

**BCrypt detayları:**
- Built-in salt (her hash unique)
- Adaptive work factor (`strength` parameter)
- 60-character hash (storage)

**Strength selection:**

| Strength | Hash time (modern CPU) | Banking |
|---|---|---|
| 4 | <1ms | ❌ çok hızlı (brute force easy) |
| 10 | ~100ms | OK (default) |
| 12 | ~250ms | ✓ Banking standard |
| 14 | ~1s | Login UX kötü |
| 16+ | ~5s | DoS riski |

**Banking pratiği:** Strength 12. Login bekleme süresi 200-300ms acceptable.

#### Storage

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash CHAR(60) NOT NULL,   -- BCrypt fixed-length
    password_changed_at TIMESTAMPTZ,
    password_expires_at TIMESTAMPTZ,
    ...
);
```

### 5. Argon2 — modern alternative

OWASP 2023+ önerisi: Argon2id (memory-hard, GPU-resistant).

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new Argon2PasswordEncoder(
        16,    // saltLength bytes
        32,    // hashLength bytes
        1,     // parallelism (threads)
        4096,  // memory KB (4 MB)
        3      // iterations
    );
}
```

**Parametre ayarı:** Banking laptop'larda ~250-500ms hash time hedef. Üretim sunucularında ölç.

#### BCrypt vs Argon2

| Kriter | BCrypt | Argon2 |
|---|---|---|
| Yıl | 1999 | 2015 (PHC winner) |
| GPU resistance | Düşük | Yüksek |
| Memory-hard | No | Yes |
| Battle-tested | ✓✓✓ | ✓✓ |
| Banking adoption | Yaygın (legacy) | Yeni projeler |

### 6. DelegatingPasswordEncoder — migration safe

Mevcut BCrypt'ten Argon2'ye geçiş:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put("bcrypt", new BCryptPasswordEncoder(12));
    encoders.put("argon2", new Argon2PasswordEncoder(16, 32, 1, 4096, 3));
    
    DelegatingPasswordEncoder encoder = new DelegatingPasswordEncoder("argon2", encoders);
    encoder.setDefaultPasswordEncoderForMatches(encoders.get("bcrypt"));   // legacy compat
    return encoder;
}
```

Hash format: `{algorithmId}hash`

```
{bcrypt}$2a$12$abcdefgh...
{argon2}$argon2id$v=19$m=4096,t=3,p=1$abc...
```

Yeni password'lar Argon2 ile hash. Login sırasında eski BCrypt match'lenir. **Rehash on login** ile kademeli migration:

```java
@Component
public class PasswordRehashListener {
    
    @EventListener
    public void onLoginSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        User user = userRepo.findByUsername(username).orElseThrow();
        
        // Check if password is BCrypt (legacy)
        if (user.getPasswordHash().startsWith("{bcrypt}")) {
            // Get raw password from authentication (only available in login event)
            String rawPassword = ...;   // implementation specific
            
            // Rehash with Argon2
            String newHash = passwordEncoder.encode(rawPassword);   // uses default argon2
            user.setPasswordHash(newHash);
            userRepo.save(user);
            
            log.info("Password rehashed for user: {}", username);
        }
    }
}
```

Banking pratiği: 6-12 ay rehash window. Sonra tüm hash'ler Argon2.

### 7. Password policy — BDDK uyumlu

Banking için **strong password policy**:

```java
@Component
public class BankingPasswordPolicy {
    
    private static final int MIN_LENGTH = 12;
    private static final int MAX_LENGTH = 128;
    private static final int MIN_UPPERCASE = 1;
    private static final int MIN_LOWERCASE = 1;
    private static final int MIN_DIGIT = 1;
    private static final int MIN_SYMBOL = 1;
    
    private final HaveIBeenPwnedClient pwnedClient;
    private final PasswordHistoryRepository historyRepo;
    
    public void validate(String password, UUID userId) {
        if (password.length() < MIN_LENGTH || password.length() > MAX_LENGTH) {
            throw new WeakPasswordException(
                "Password length must be between " + MIN_LENGTH + " and " + MAX_LENGTH);
        }
        
        if (countMatches(password, "[A-Z]") < MIN_UPPERCASE) {
            throw new WeakPasswordException("At least " + MIN_UPPERCASE + " uppercase required");
        }
        if (countMatches(password, "[a-z]") < MIN_LOWERCASE) {
            throw new WeakPasswordException("At least " + MIN_LOWERCASE + " lowercase required");
        }
        if (countMatches(password, "[0-9]") < MIN_DIGIT) {
            throw new WeakPasswordException("At least " + MIN_DIGIT + " digit required");
        }
        if (countMatches(password, "[^a-zA-Z0-9]") < MIN_SYMBOL) {
            throw new WeakPasswordException("At least " + MIN_SYMBOL + " symbol required");
        }
        
        // Common password check (top 1M passwords)
        if (CommonPasswords.contains(password)) {
            throw new WeakPasswordException("Password is too common");
        }
        
        // HaveIBeenPwned check (k-anonymity)
        if (pwnedClient.isCompromised(password)) {
            throw new WeakPasswordException("This password has been found in data breaches");
        }
        
        // History check — last 5 passwords
        List<String> recentHashes = historyRepo.findLast5ByUserId(userId);
        for (String oldHash : recentHashes) {
            if (passwordEncoder.matches(password, oldHash)) {
                throw new WeakPasswordException("Cannot reuse last 5 passwords");
            }
        }
        
        // Common patterns
        if (isSequential(password) || isRepetitive(password)) {
            throw new WeakPasswordException("Password contains predictable pattern");
        }
    }
    
    private int countMatches(String input, String regex) {
        return (int) input.chars().filter(c -> String.valueOf((char) c).matches(regex)).count();
    }
    
    private boolean isSequential(String password) {
        // "12345", "abcde" → sequential
        for (int i = 0; i < password.length() - 4; i++) {
            char c = password.charAt(i);
            if (password.charAt(i+1) == c + 1 && 
                password.charAt(i+2) == c + 2 &&
                password.charAt(i+3) == c + 3 &&
                password.charAt(i+4) == c + 4) {
                return true;
            }
        }
        return false;
    }
    
    private boolean isRepetitive(String password) {
        // "aaaaaa", "11111" → repetitive
        for (int i = 0; i < password.length() - 4; i++) {
            char c = password.charAt(i);
            if (password.charAt(i+1) == c && password.charAt(i+2) == c && 
                password.charAt(i+3) == c && password.charAt(i+4) == c) {
                return true;
            }
        }
        return false;
    }
}
```

**BDDK requirements (yaklaşık):**
- Minimum 8 character
- Complexity (uppercase, lowercase, digit, symbol)
- Expiration 90 days
- History 5 (cannot reuse)
- No common patterns

### 8. HaveIBeenPwned API — k-anonymity

`Have I Been Pwned?` (HIBP) — milyarlarca leaked password database.

**K-anonymity:** Password'i göndermek yerine SHA-1 hash'in **ilk 5 karakteri** gönderilir. Server o prefix ile başlayan tüm hash'lerin **suffix'lerini** döner. Client local'de match arar.

```java
@Component
public class HaveIBeenPwnedClient {
    
    private final RestTemplate restTemplate;
    
    public boolean isCompromised(String password) {
        String sha1 = sha1Hex(password).toUpperCase();
        String prefix = sha1.substring(0, 5);
        String suffix = sha1.substring(5);
        
        String response = restTemplate.getForObject(
            "https://api.pwnedpasswords.com/range/" + prefix,
            String.class
        );
        
        return response.lines().anyMatch(line -> line.startsWith(suffix + ":"));
    }
    
    private String sha1Hex(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-1");
            byte[] bytes = md.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder sb = new StringBuilder();
            for (byte b : bytes) {
                sb.append(String.format("%02x", b));
            }
            return sb.toString();
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

**Privacy guarantee:** API gerçek password'i bilmez, sadece SHA-1 prefix'i (10^5 = 100,000 possibility).

Banking pratiği: Password set/change'inde mutlaka HIBP check. Compromised password reject.

### 9. Brute force protection — account lockout

Banking için **şart**:

```java
@Component
public class LoginAttemptService {
    
    private static final int MAX_ATTEMPTS = 5;
    private static final Duration LOCK_DURATION = Duration.ofMinutes(15);
    
    private final Map<String, AttemptRecord> attempts = new ConcurrentHashMap<>();
    
    public void loginFailed(String username, String ip) {
        AttemptRecord record = attempts.computeIfAbsent(username, k -> new AttemptRecord());
        
        int count = record.count.incrementAndGet();
        record.lastAttemptAt = Instant.now();
        record.failedIps.add(ip);
        
        if (count >= MAX_ATTEMPTS) {
            record.lockedUntil = Instant.now().plus(LOCK_DURATION);
            
            // Notify user (suspicious activity)
            notificationService.sendSecurityAlert(username, 
                "Multiple failed login attempts detected. Account locked for 15 minutes.");
            
            // Audit log
            auditLog.log(SecurityEvent.builder()
                .type("ACCOUNT_LOCKED")
                .username(username)
                .ip(ip)
                .reason("MAX_FAILED_ATTEMPTS")
                .build());
        }
    }
    
    public void loginSucceeded(String username) {
        attempts.remove(username);
    }
    
    public boolean isBlocked(String username) {
        AttemptRecord record = attempts.get(username);
        if (record == null) return false;
        
        if (record.lockedUntil != null && record.lockedUntil.isAfter(Instant.now())) {
            return true;
        }
        
        // Lock duration expired
        if (record.lockedUntil != null) {
            attempts.remove(username);
        }
        return false;
    }
    
    private static class AttemptRecord {
        AtomicInteger count = new AtomicInteger();
        Instant lastAttemptAt;
        Instant lockedUntil;
        Set<String> failedIps = ConcurrentHashMap.newKeySet();
    }
}
```

**Banking pratiği:** 5 fail → 15 dk lock. Distributed multi-instance → **Redis-backed** (in-memory yetmez).

```java
@Component
public class RedisLoginAttemptService {
    
    private final StringRedisTemplate redis;
    
    public void loginFailed(String username) {
        String key = "login:attempts:" + username;
        Long count = redis.opsForValue().increment(key);
        redis.expire(key, Duration.ofMinutes(15));
        
        if (count >= 5) {
            redis.opsForValue().set("login:locked:" + username, "true", Duration.ofMinutes(15));
        }
    }
    
    public boolean isBlocked(String username) {
        return Boolean.TRUE.equals(redis.hasKey("login:locked:" + username));
    }
    
    public void loginSucceeded(String username) {
        redis.delete("login:attempts:" + username);
        redis.delete("login:locked:" + username);
    }
}
```

### 10. IP-based protection

Username + IP kombinasyonu — distributed brute force korunması.

```java
public void loginFailed(String username, String ip) {
    // Per-user
    redis.opsForValue().increment("login:attempts:user:" + username);
    redis.expire("login:attempts:user:" + username, Duration.ofMinutes(15));
    
    // Per-IP (multiple users from same IP attacking)
    Long ipCount = redis.opsForValue().increment("login:attempts:ip:" + ip);
    redis.expire("login:attempts:ip:" + ip, Duration.ofMinutes(15));
    
    if (ipCount >= 50) {   // 50 fail from one IP → block IP
        redis.opsForValue().set("login:blocked:ip:" + ip, "true", Duration.ofHours(1));
        alerts.notifySecurityOps("Suspicious IP: " + ip);
    }
}
```

Banking ips: IP block + alert. CAPTCHA enforcement.

### 11. MFA (Multi-Factor Authentication)

Banking için **şart** (BDDK requirement):

3 factor:
1. **Knowledge** — password (something you know)
2. **Possession** — phone (SMS OTP), security key (FIDO2)
3. **Inherence** — biometric (fingerprint, face)

#### TOTP (Time-based One-Time Password)

Google Authenticator, Authy gibi app'lerle. RFC 6238.

```java
@Component
public class TotpService {
    
    private final GoogleAuthenticator googleAuth = new GoogleAuthenticator();
    
    public String generateSecret() {
        return googleAuth.createCredentials().getKey();
    }
    
    public String generateQrCodeUrl(String username, String secret) {
        return GoogleAuthenticatorQRGenerator.getOtpAuthURL(
            "MaviBank", username, new GoogleAuthenticatorKey.Builder(secret).build()
        );
    }
    
    public boolean validate(String secret, int totpCode) {
        return googleAuth.authorize(secret, totpCode);
    }
}
```

#### SMS OTP

```java
@Component
public class SmsOtpService {
    
    private final SmsGatewayClient smsGateway;
    private final RedisTemplate<String, String> redis;
    
    public void sendOtp(String phoneNumber) {
        String otp = generateOtp();
        redis.opsForValue().set("otp:" + phoneNumber, otp, Duration.ofMinutes(5));
        
        smsGateway.send(phoneNumber, 
            "MaviBank OTP: " + otp + ". Geçerlilik 5 dakika.");
        
        auditLog.log("SMS_OTP_SENT", phoneNumber);
    }
    
    public boolean validate(String phoneNumber, String otp) {
        String stored = redis.opsForValue().get("otp:" + phoneNumber);
        if (stored == null) return false;
        
        boolean valid = MessageDigest.isEqual(
            stored.getBytes(StandardCharsets.UTF_8),
            otp.getBytes(StandardCharsets.UTF_8)   // timing-safe comparison
        );
        
        if (valid) {
            redis.delete("otp:" + phoneNumber);
        }
        return valid;
    }
    
    private String generateOtp() {
        SecureRandom random = new SecureRandom();
        return String.format("%06d", random.nextInt(1000000));   // 6-digit
    }
}
```

**Banking pratiği:** SMS OTP backup, TOTP primary. Modern: WhatsApp Business OTP de var.

#### MFA login flow

```java
@RestController
public class AuthController {
    
    @PostMapping("/auth/login")
    public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest req) {
        // Step 1: Username + password
        User user = authenticate(req.username(), req.password());
        
        // Step 2: MFA challenge
        if (user.isMfaEnabled()) {
            String challengeId = UUID.randomUUID().toString();
            
            redisTemplate.opsForValue().set(
                "mfa:challenge:" + challengeId,
                user.getId().toString(),
                Duration.ofMinutes(5)
            );
            
            if (user.getMfaMethod() == MfaMethod.SMS) {
                smsOtpService.sendOtp(user.getPhone());
                return ResponseEntity.ok(LoginResponse.mfaRequired(
                    challengeId, "SMS_OTP", "Kod " + user.getMaskedPhone() + "'a gönderildi"));
            } else if (user.getMfaMethod() == MfaMethod.TOTP) {
                return ResponseEntity.ok(LoginResponse.mfaRequired(
                    challengeId, "TOTP", "Lütfen authenticator app'inizdeki kodu girin"));
            }
        }
        
        // No MFA — issue token
        return ResponseEntity.ok(LoginResponse.success(tokenService.issueTokens(user)));
    }
    
    @PostMapping("/auth/mfa/verify")
    public ResponseEntity<LoginResponse> verifyMfa(@RequestBody MfaVerifyRequest req) {
        String userIdStr = redisTemplate.opsForValue().get("mfa:challenge:" + req.challengeId());
        if (userIdStr == null) {
            return ResponseEntity.status(401).body(LoginResponse.error("Challenge expired"));
        }
        
        User user = userRepo.findById(UUID.fromString(userIdStr)).orElseThrow();
        
        boolean valid = switch (user.getMfaMethod()) {
            case SMS -> smsOtpService.validate(user.getPhone(), req.code());
            case TOTP -> totpService.validate(user.getMfaSecret(), Integer.parseInt(req.code()));
        };
        
        if (!valid) {
            return ResponseEntity.status(401).body(LoginResponse.error("Invalid MFA code"));
        }
        
        redisTemplate.delete("mfa:challenge:" + req.challengeId());
        
        return ResponseEntity.ok(LoginResponse.success(tokenService.issueTokens(user)));
    }
}
```

### 12. FIDO2 / WebAuthn — hardware key

Modern banking için **physical security key** (YubiKey, Touch ID).

`webauthn4j` library Spring Security entegre. Banking ileri (Phase 8 sonrası).

### 13. Session management

Spring Security session policies:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(s -> s
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)   // banking JWT
            .maximumSessions(1)   // tek aktif device
            .maxSessionsPreventsLogin(false)   // yeni login eski'yi kovacak
            .expiredUrl("/login?expired"))
        ...
    return http.build();
}
```

**Banking stateless vs stateful:**

- **Stateful session:** Eski model. Server `JSESSIONID` cookie. Sticky session veya distributed (Redis).
- **Stateless JWT:** Modern. Server state'siz. Token expiry ile kontrol.

Banking modern: **Stateless JWT** (Topic 8.3).

### 14. Concurrent session limit

```yaml
spring:
  security:
    session:
      max-concurrent: 1
```

Banking: Tek device login. Yeni login → eski oturum invalidate.

### 15. Session fixation protection

```java
.sessionManagement(s -> s
    .sessionFixation().migrateSession())   // login sonrası yeni session ID
```

Default Spring Security'de aktif. Eski session ID'leri attack için kullanılamaz.

### 16. Banking örnek — full auth flow

```java
@Configuration
public class BankingSecurityConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new DelegatingPasswordEncoder("argon2", Map.of(
            "bcrypt", new BCryptPasswordEncoder(12),
            "argon2", new Argon2PasswordEncoder(16, 32, 1, 4096, 3)
        ));
    }
    
    @Bean
    public AuthenticationProvider daoAuthenticationProvider(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        provider.setHideUserNotFoundExceptions(true);   // username enumeration prevention
        return provider;
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
                                            LoginAttemptFilter attemptFilter,
                                            JwtAuthenticationFilter jwtFilter) throws Exception {
        http
            .csrf(c -> c.disable())   // stateless API
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/auth/**", "/v1/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("admin")
                .anyRequest().authenticated())
            .addFilterBefore(attemptFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}

@Component
public class LoginAttemptFilter extends OncePerRequestFilter {
    
    private final LoginAttemptService attemptService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        if (req.getRequestURI().equals("/auth/login")) {
            String username = extractUsername(req);
            if (username != null && attemptService.isBlocked(username)) {
                res.setStatus(HttpStatus.LOCKED.value());
                res.getWriter().write("{\"error\":\"Account locked\"}");
                return;
            }
        }
        chain.doFilter(req, res);
    }
}
```

### 17. Authentication events + audit

```java
@Component
public class AuthenticationEventListener {
    
    @EventListener
    public void onSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        loginAttemptService.loginSucceeded(username);
        
        auditLog.log(SecurityEvent.builder()
            .type("LOGIN_SUCCESS")
            .username(username)
            .ip(getCurrentIp())
            .userAgent(getCurrentUserAgent())
            .build());
    }
    
    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        loginAttemptService.loginFailed(username, getCurrentIp());
        
        auditLog.log(SecurityEvent.builder()
            .type("LOGIN_FAILURE")
            .username(username)
            .ip(getCurrentIp())
            .reason(event.getException().getMessage())
            .build());
    }
}
```

Banking pratiği: **Tüm authentication event'leri immutable audit log**'a yazılır. KVKK + regulatory.

### 18. Banking anti-pattern'leri

**Anti-pattern 1: Plain MD5/SHA-1 password hash**

```java
password_hash = sha1(password);   // ❌ Banking için yasak
```

Hızlı = saldırgan da hızlı brute force. **BCrypt veya Argon2.**

**Anti-pattern 2: BCrypt strength 4-8**

Çok hızlı hash → brute force kolay. Banking strength **12+**.

**Anti-pattern 3: Salt elle ekleme**

```java
String hash = bcrypt(password + customSalt);   // ❌
```

BCrypt salt'ı kendi yapıyor. Custom salt → bug + standart dışı.

**Anti-pattern 4: Brute force protection yok**

```java
@PostMapping("/auth/login")
public LoginResponse login(LoginRequest req) {
    User user = userRepo.findByUsername(req.username()).orElseThrow();
    if (passwordEncoder.matches(req.password(), user.getPasswordHash())) {
        return success(...);
    }
    return failure();
    // ❌ Unlimited attempts → brute force open
}
```

Banking için **5 fail → 15 dk lock + alert**.

**Anti-pattern 5: MFA opsiyonel**

Banking için MFA **zorunlu** olmalı. Mobile banking app: TOTP + biometric.

**Anti-pattern 6: SMS OTP plain text log'a**

```java
log.info("Sending OTP {} to {}", otp, phone);   // ❌ secret in log
```

OTP **asla** log'lanmaz. Sadece "OTP sent" event.

**Anti-pattern 7: Username enumeration**

```java
catch (UsernameNotFoundException e) {
    return "User not found";   // ❌ farklı message → username olduğunu açığa çıkar
}
catch (BadCredentialsException e) {
    return "Wrong password";   // ❌
}
```

Banking: **Tek generic mesaj** — "Invalid credentials". User exist mi belli etme.

`hideUserNotFoundExceptions(true)` Spring Security'de.

**Anti-pattern 8: Password expiry yok**

90 gün rotation BDDK gereksinimi. `password_expires_at` field.

**Anti-pattern 9: Password history yok**

User same password 5 kez = anti-pattern. Last 5 hash karşılaştırması.

**Anti-pattern 10: Reset password URL'de token + no expiry**

```
https://banking.com/reset?token=abc123
```

Token expiry yok → forever valid. 15 dakika TTL + one-time use.

---

## Önemli olabilecek araştırma kaynakları

- OWASP Password Storage Cheat Sheet (2023)
- "Spring Security in Action" (Laurentiu Spilca)
- Argon2 RFC 9106
- BCrypt original paper (Niels Provos)
- HaveIBeenPwned API documentation
- BDDK güvenlik regülasyonları (TR specific)
- WebAuthn / FIDO2 specs
- Google Authenticator (TOTP) RFC 6238

---

## Mini task'ler

### Task 8.2.1 — UserDetailsService + DB lookup (45 dk)

`BankingUserDetailsService` implement et. Username/email/phone flexible lookup. Status checks (locked, suspended, expired).

### Task 8.2.2 — BCrypt strength 12 (15 dk)

`PasswordEncoder` bean. Test hash time ~250ms.

### Task 8.2.3 — DelegatingPasswordEncoder + migration (60 dk)

BCrypt + Argon2 dual encoder. Existing BCrypt hash matchle + new Argon2 ile yarat.

Login event listener'da rehash on success pattern.

### Task 8.2.4 — Password policy (60 dk)

`BankingPasswordPolicy.validate()` — length, complexity, history, breach check, pattern.

Test: 10 farklı weak password reject case.

### Task 8.2.5 — HaveIBeenPwned k-anonymity (30 dk)

`HaveIBeenPwnedClient.isCompromised(password)` — SHA-1 prefix lookup.

Test: bilinen leaked password (örnek: "Password123!") compromised dönmeli.

### Task 8.2.6 — Brute force protection Redis-backed (60 dk)

`RedisLoginAttemptService`. 5 fail/15 dk lock. Per-user + per-IP.

Test:
- 5 fail → 6. attempt 423 Locked
- 15 dk sonra unlock
- Different user same IP — bağımsız

### Task 8.2.7 — TOTP MFA (60 dk)

GoogleAuthenticator library. Secret generate + QR code URL. Validate.

Test: Authenticator app'le QR scan → 6-digit kod doğrulanmalı.

### Task 8.2.8 — SMS OTP (45 dk)

Mock SMS gateway. 6-digit OTP generate, Redis 5 dk TTL, timing-safe compare.

### Task 8.2.9 — Full MFA login flow (60 dk)

`/auth/login` + `/auth/mfa/verify` endpoint'leri. Challenge ID Redis'te.

Test: 2-step flow end-to-end.

### Task 8.2.10 — Authentication event audit (30 dk)

`AuthenticationEventListener` ile success/failure event → audit log.

---

## Test yazma rehberi

```java
@SpringBootTest
@AutoConfigureMockMvc
class AuthenticationIntegrationTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Autowired MockMvc mockMvc;
    @Autowired UserRepository userRepo;
    @Autowired PasswordEncoder passwordEncoder;
    
    @Test
    void shouldRejectWeakPassword() throws Exception {
        mockMvc.perform(post("/auth/register")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"username": "test", "password": "12345678"}
                """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.error").value(containsString("complexity")));
    }
    
    @Test
    void shouldHashPasswordWithArgon2() {
        String raw = "MyStr0ngP@ssw0rd!";
        String hash = passwordEncoder.encode(raw);
        
        assertThat(hash).startsWith("{argon2}");
        assertThat(passwordEncoder.matches(raw, hash)).isTrue();
        assertThat(passwordEncoder.matches("wrong", hash)).isFalse();
    }
    
    @Test
    void shouldLockAccountAfter5FailedAttempts() throws Exception {
        createTestUser("ali", "CorrectPassword!");
        
        for (int i = 0; i < 5; i++) {
            mockMvc.perform(post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"username": "ali", "password": "WrongPassword!"}
                    """))
                .andExpect(status().isUnauthorized());
        }
        
        // 6th attempt → locked
        mockMvc.perform(post("/auth/login")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"username": "ali", "password": "CorrectPassword!"}
                """))
            .andExpect(status().isLocked());
    }
    
    @Test
    void shouldNotRevealUsernameExistence() throws Exception {
        // Non-existent user
        mockMvc.perform(post("/auth/login")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"username": "nonexistent", "password": "any"}
                """))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.error").value("Invalid credentials"));   // generic
        
        // Existing user, wrong password
        createTestUser("ali", "CorrectPassword!");
        mockMvc.perform(post("/auth/login")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"username": "ali", "password": "wrong"}
                """))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.error").value("Invalid credentials"));   // same message
    }
    
    @Test
    void shouldRequireMfaWhenEnabled() throws Exception {
        User user = createUserWithMfa("ali", "CorrectPassword!", MfaMethod.TOTP);
        
        // Step 1: username + password
        MvcResult result = mockMvc.perform(post("/auth/login")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"username": "ali", "password": "CorrectPassword!"}
                """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.mfaRequired").value(true))
            .andExpect(jsonPath("$.challengeId").exists())
            .andReturn();
        
        String challengeId = JsonPath.read(result.getResponse().getContentAsString(), "$.challengeId");
        
        // Step 2: TOTP code
        String totpCode = generateValidTotpCode(user.getMfaSecret());
        
        mockMvc.perform(post("/auth/mfa/verify")
            .contentType(MediaType.APPLICATION_JSON)
            .content(String.format("""
                {"challengeId": "%s", "code": "%s"}
                """, challengeId, totpCode)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.accessToken").exists());
    }
}
```

---

## Claude-verify prompt

```
Authentication implementation'ımı banking-grade kriterlere göre değerlendir:

1. Password hashing:
   - BCrypt strength 12+ veya Argon2?
   - DelegatingPasswordEncoder migration safe?
   - Rehash on login pattern var mı?

2. UserDetailsService:
   - Username + email + phone flexible lookup?
   - Status checks (locked, suspended, password expired)?
   - hideUserNotFoundExceptions=true username enumeration prevention?

3. Password policy:
   - Min 12 char + complexity (4 group)?
   - History 5 (no reuse)?
   - HaveIBeenPwned k-anonymity check?
   - Common pattern (sequential, repetitive) reject?

4. Brute force protection:
   - 5 fail → 15 dk lock?
   - Redis-backed (distributed)?
   - Per-user + per-IP tracking?
   - Audit log + alert?

5. MFA:
   - TOTP + SMS OTP support?
   - Challenge ID Redis 5 dk TTL?
   - Timing-safe OTP comparison?
   - OTP **asla** log'lanmaz?

6. Session management:
   - Stateless JWT (banking modern)?
   - Concurrent session limit 1?
   - Session fixation protection?

7. Audit:
   - AuthenticationEventListener?
   - Login success + failure event audit log?
   - Immutable storage?

8. Banking-specific:
   - BDDK password policy compliance?
   - Reset password 15 dk one-time token?
   - PII (TC No, phone) PII protection?

9. Anti-pattern:
   - MD5/SHA-1 password?
   - Strength < 10?
   - Username enumeration?
   - MFA opsiyonel?
   - OTP log'da?
   - Reset token expiry yok?

10. Test:
    - Weak password reject test?
    - Account lock after 5 fail test?
    - Generic error message (no enumeration) test?
    - MFA full flow test?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] UserDetailsService DB integration + status checks
- [ ] DelegatingPasswordEncoder (BCrypt + Argon2)
- [ ] BankingPasswordPolicy (length, complexity, history, breach, pattern)
- [ ] HaveIBeenPwned k-anonymity check
- [ ] RedisLoginAttemptService (5 fail/15 dk)
- [ ] Per-IP block 50 fail/1h
- [ ] TOTP MFA + QR code
- [ ] SMS OTP MFA
- [ ] Full MFA login flow endpoint'leri
- [ ] AuthenticationEventListener audit
- [ ] Session stateless + concurrent limit
- [ ] 5+ integration test

---

## Defter notları (10 madde)

1. "Authentication vs Authorization farkı + banking örneği: ____."
2. "Spring Security authentication flow (Filter → Manager → Provider → UserDetailsService): ____."
3. "BCrypt vs Argon2 banking için seçim + work factor: ____."
4. "DelegatingPasswordEncoder migration pattern: ____."
5. "HaveIBeenPwned k-anonymity privacy guarantee: ____."
6. "Brute force protection 5/15 + per-IP block: ____."
7. "MFA 3 factor (knowledge/possession/inherence) banking örneği: ____."
8. "TOTP vs SMS OTP + WhatsApp OTP modern alternative: ____."
9. "Username enumeration prevention (generic error + hideUserNotFoundExceptions): ____."
10. "BDDK password policy compliance (length, complexity, expiration, history): ____."
