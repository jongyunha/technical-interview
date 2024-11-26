# 인증(Authentication)과 인가(Authorization) 기본 개념

## 주요 개념

### 1. 인증 (Authentication)
- "너가 누구인지 확인하는 과정"
- 신원을 증명하는 과정
- ex) 로그인

### 2. 인가 (Authorization)
- "너가 무엇을 할 수 있는지 확인하는 과정"
- 권한을 확인하는 과정
- ex) 관리자 페이지 접근 권한

```java
@Service
public class AuthenticationService {
    
    // 1. 기본적인 인증 처리
    public class AuthenticationManager {
        private final UserRepository userRepository;
        private final PasswordEncoder passwordEncoder;
        
        public AuthenticationResult authenticate(
            String username, String password) {
            
            // 1. 사용자 조회
            User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new AuthenticationException(
                    "User not found"));
            
            // 2. 비밀번호 검증
            if (!passwordEncoder.matches(password, user.getPassword())) {
                throw new AuthenticationException("Invalid password");
            }
            
            // 3. 인증 성공 처리
            return createAuthenticationResult(user);
        }
        
        private AuthenticationResult createAuthenticationResult(User user) {
            return AuthenticationResult.builder()
                .userId(user.getId())
                .username(user.getUsername())
                .roles(user.getRoles())
                .authorities(user.getAuthorities())
                .authenticated(true)
                .build();
        }
    }
    
    // 2. 권한 관리
    @Component
    public class AuthorizationManager {
        
        public boolean hasPermission(
            Authentication authentication, 
            String resource, 
            String action) {
            
            // 1. 사용자 권한 확인
            Set<String> authorities = authentication.getAuthorities()
                .stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toSet());
            
            // 2. 리소스에 대한 권한 검증
            Permission requiredPermission = 
                Permission.of(resource, action);
                
            return authorities.contains(requiredPermission.toString()) ||
                   hasRole(authentication, "ADMIN");
        }
        
        private boolean hasRole(
            Authentication authentication, 
            String role) {
            
            return authentication.getAuthorities()
                .stream()
                .anyMatch(auth -> 
                    auth.getAuthority().equals("ROLE_" + role));
        }
    }
}

// 3. Security Context 관리
@Component
public class SecurityContextHolder {
    private static final ThreadLocal<SecurityContext> contextHolder = 
        new ThreadLocal<>();
        
    public static void setContext(SecurityContext context) {
        contextHolder.set(context);
    }
    
    public static SecurityContext getContext() {
        SecurityContext ctx = contextHolder.get();
        if (ctx == null) {
            ctx = createEmptyContext();
            contextHolder.set(ctx);
        }
        return ctx;
    }
    
    public static void clearContext() {
        contextHolder.remove();
    }
}

// 4. 접근 제어 구현
@Aspect
@Component
public class SecurityAspect {
    
    private final AuthorizationManager authorizationManager;
    
    @Around("@annotation(secured)")
    public Object checkSecurity(
        ProceedingJoinPoint joinPoint, 
        Secured secured) throws Throwable {
        
        // 1. 현재 인증 정보 확인
        Authentication authentication = 
            SecurityContextHolder.getContext().getAuthentication();
            
        if (authentication == null || 
            !authentication.isAuthenticated()) {
            throw new AccessDeniedException("Authentication required");
        }
        
        // 2. 권한 확인
        if (!authorizationManager.hasPermission(
            authentication, 
            secured.resource(), 
            secured.action())) {
            throw new AccessDeniedException("Insufficient privileges");
        }
        
        return joinPoint.proceed();
    }
}

// 5. 사용 예시
@RestController
@RequestMapping("/api")
public class UserController {
    
    @Secured(resource = "user", action = "read")
    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        // 사용자 조회 로직
    }
    
    @Secured(resource = "user", action = "write")
    @PostMapping("/users")
    public UserResponse createUser(@RequestBody UserRequest request) {
        // 사용자 생성 로직
    }
}
```

이러한 기본 구조를 통해:

1. 인증(Authentication)
    - 사용자 신원 확인
    - 비밀번호 검증
    - 인증 상태 관리

2. 인가(Authorization)
    - 권한 검증
    - 리소스 접근 제어
    - 역할 기반 권한 관리

3. 보안 컨텍스트
    - 인증 정보 유지
    - 스레드별 컨텍스트 관리
    - 보안 정보 격리

4. 접근 제어
    - AOP 기반 보안 처리
    - 선언적 보안 설정
    - 세밀한 권한 제어

를 구현할 수 있습니다.

면접관: 실제 서비스에서 인증/인가 시스템을 구현할 때의 보안 고려사항은 무엇인가요?

## 실제 서비스의 인증/인가 보안 고려사항

### 1. 패스워드 보안
```java
@Service
public class PasswordSecurityService {
    
    private final PasswordEncoder passwordEncoder;
    private final PasswordValidator passwordValidator;

    // 1. 안전한 패스워드 해싱
    public String hashPassword(String rawPassword) {
        // BCrypt 사용 - 자동 솔트 생성 및 적용
        return passwordEncoder.encode(rawPassword);
    }

    // 2. 패스워드 정책 적용
    public void validatePassword(String password) {
        PasswordPolicy policy = PasswordPolicy.builder()
            .minLength(12)
            .requireUppercase(true)
            .requireLowercase(true)
            .requireNumbers(true)
            .requireSpecialChars(true)
            .preventCommonPasswords(true)
            .preventUserInfoInPassword(true)
            .build();

        List<PasswordViolation> violations = 
            passwordValidator.validate(password, policy);

        if (!violations.isEmpty()) {
            throw new PasswordPolicyViolationException(violations);
        }
    }

    // 3. 이전 패스워드 재사용 방지
    @Transactional
    public void changePassword(User user, String newPassword) {
        // 이전 패스워드 히스토리 확인
        if (isPasswordPreviouslyUsed(user, newPassword)) {
            throw new PasswordRecentlyUsedException(
                "Cannot reuse recent passwords");
        }

        // 패스워드 변경 및 히스토리 저장
        user.setPassword(hashPassword(newPassword));
        savePasswordHistory(user, newPassword);
    }
}
```

### 2. 세션 보안
```java
@Component
public class SessionSecurityManager {
    
    private final RedisTemplate<String, Object> redisTemplate;

    // 1. 세션 하이재킹 방지
    public HttpSession secureSession(HttpSession session) {
        // 세션 ID 재생성
        session.invalidate();
        HttpSession newSession = request.getSession(true);
        
        // 보안 속성 설정
        newSession.setAttribute("_csrf", generateCsrfToken());
        newSession.setAttribute("client_ip", request.getRemoteAddr());
        newSession.setAttribute("user_agent", request.getHeader("User-Agent"));

        return newSession;
    }

    // 2. 동시 세션 제어
    public void enforceSessionControl(String userId) {
        String sessionKey = "user_sessions:" + userId;
        
        // 최대 허용 세션 수 확인
        Long sessionCount = redisTemplate.opsForSet()
            .size(sessionKey);
            
        if (sessionCount >= MAX_SESSIONS_PER_USER) {
            // 가장 오래된 세션 종료
            String oldestSession = redisTemplate.opsForSet()
                .pop(sessionKey);
            invalidateSession(oldestSession);
        }
    }

    // 3. 세션 탈취 감지
    @Scheduled(fixedRate = 5000)
    public void detectSessionAnomaly() {
        activeSessions.forEach((sessionId, sessionInfo) -> {
            if (isAnomalousActivity(sessionInfo)) {
                terminateSession(sessionId);
                notifySecurityTeam(sessionInfo);
            }
        });
    }
}
```

### 3. 접근 제어 강화
```java
@Service
public class EnhancedAuthorizationService {

    // 1. 세분화된 권한 체크
    public boolean checkAccess(
        Authentication auth, 
        SecurityResource resource) {
        
        return Stream.of(
            checkResourceOwnership(auth, resource),
            checkRoleBasedAccess(auth, resource),
            checkTimeBasedAccess(auth, resource),
            checkLocationBasedAccess(auth, resource)
        ).allMatch(result -> result);
    }

    // 2. 동적 권한 관리
    public class DynamicPermissionManager {
        private final Cache<String, Set<Permission>> permissionCache;
        
        public Set<Permission> getEffectivePermissions(
            Authentication auth) {
            
            String cacheKey = auth.getName();
            
            return permissionCache.get(cacheKey, key -> {
                Set<Permission> permissions = new HashSet<>();
                
                // 기본 권한
                permissions.addAll(getBasePermissions(auth));
                
                // 상황별 권한
                permissions.addAll(
                    getContextualPermissions(auth));
                
                // 임시 권한
                permissions.addAll(
                    getTemporaryPermissions(auth));
                
                return permissions;
            });
        }
    }

    // 3. 감사 로깅
    @Aspect
    @Component
    public class SecurityAuditAspect {
        
        @Around("@annotation(secured)")
        public Object auditSecurityAccess(
            ProceedingJoinPoint joinPoint, 
            Secured secured) throws Throwable {
            
            AuditEvent auditEvent = AuditEvent.builder()
                .principal(getCurrentUser())
                .action(secured.action())
                .resource(secured.resource())
                .timestamp(Instant.now())
                .status("ATTEMPTED")
                .build();

            try {
                Object result = joinPoint.proceed();
                auditEvent.setStatus("SUCCESS");
                return result;
            } catch (SecurityException e) {
                auditEvent.setStatus("DENIED");
                auditEvent.setFailureReason(e.getMessage());
                throw e;
            } finally {
                auditLogger.log(auditEvent);
            }
        }
    }
}
```

### 4. 공격 방지
```java
@Component
public class SecurityDefenseSystem {

    // 1. 브루트포스 공격 방지
    @Service
    public class BruteForceProtection {
        private final RateLimiter rateLimiter;
        
        public void checkLoginAttempt(String username) {
            String key = "login_attempt:" + username;
            
            if (!rateLimiter.tryAcquire(key)) {
                throw new AccountLockedException(
                    "Too many login attempts");
            }
        }
        
        public void handleFailedLogin(String username) {
            String key = "failed_login:" + username;
            
            int attempts = incrementFailedAttempts(key);
            if (attempts >= MAX_FAILED_ATTEMPTS) {
                lockAccount(username);
            }
        }
    }

    // 2. 입력 검증
    public class InputValidator {
        
        public void validateInput(String input, InputType type) {
            Pattern pattern = getValidationPattern(type);
            
            if (!pattern.matcher(input).matches()) {
                throw new InvalidInputException(
                    "Invalid input format");
            }
            
            // XSS 방지
            input = sanitizeInput(input);
            
            // SQL Injection 방지
            input = escapeSqlCharacters(input);
        }
    }

    // 3. 보안 모니터링
    @Service
    public class SecurityMonitor {
        
        @Scheduled(fixedRate = 1000)
        public void monitorSecurityEvents() {
            // 비정상 패턴 감지
            detectAnomalousPatterns();
            
            // 동시 로그인 감지
            detectConcurrentLogins();
            
            // 권한 변경 감지
            detectPrivilegeChanges();
        }
        
        private void handleSecurityIncident(
            SecurityIncident incident) {
            
            // 즉시 대응
            immediateResponse(incident);
            
            // 보안팀 알림
            notifySecurityTeam(incident);
            
            // 증거 수집
            collectEvidence(incident);
        }
    }
}
```

이러한 보안 고려사항을 통해:

1. 패스워드 보안
    - 안전한 해싱
    - 강력한 정책
    - 이력 관리

2. 세션 보안
    - 하이재킹 방지
    - 동시 접속 제어
    - 이상 징후 감지

3. 접근 제어
    - 세밀한 권한 관리
    - 동적 권한 부여
    - 감사 로깅

4. 공격 방지
    - 브루트포스 방어
    - 입력 검증
    - 실시간 모니터링

을 구현하여 보안성을 강화할 수 있습니다.