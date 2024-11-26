# OAuth 2.0과 소셜 로그인 시스템 설계

## 1. OAuth 2.0 클라이언트 구현

```java
@Service
public class OAuth2ClientService {

    // 1. OAuth2 설정 관리
    @Configuration
    public class OAuth2Config {
        
        @Bean
        public Map<String, OAuth2ClientProperties> oauth2Clients() {
            return Map.of(
                "google", GoogleOAuth2Properties.builder()
                    .clientId(googleClientId)
                    .clientSecret(googleClientSecret)
                    .redirectUri("/oauth2/callback/google")
                    .scope("profile email")
                    .build(),
                "github", GithubOAuth2Properties.builder()
                    .clientId(githubClientId)
                    .clientSecret(githubClientSecret)
                    .redirectUri("/oauth2/callback/github")
                    .scope("user:email")
                    .build()
            );
        }
    }

    // 2. 인증 요청 처리
    @Service
    public class OAuth2AuthorizationService {
        private final OAuth2ClientProperties clientProperties;
        private final StateService stateService;

        public String generateAuthorizationUrl(String provider) {
            OAuth2ClientProperties props = oauth2Clients.get(provider);
            String state = stateService.generateState();

            return UriComponentsBuilder
                .fromUriString(props.getAuthorizationUri())
                .queryParam("client_id", props.getClientId())
                .queryParam("redirect_uri", props.getRedirectUri())
                .queryParam("scope", props.getScope())
                .queryParam("response_type", "code")
                .queryParam("state", state)
                .build()
                .toString();
        }

        // CSRF 방지를 위한 state 관리
        @Component
        public class StateService {
            private final RedisTemplate<String, String> redisTemplate;

            public String generateState() {
                String state = generateSecureRandomString();
                redisTemplate.opsForValue().set(
                    "oauth2_state:" + state,
                    state,
                    5,
                    TimeUnit.MINUTES
                );
                return state;
            }

            public boolean validateState(String state) {
                String storedState = redisTemplate.opsForValue()
                    .get("oauth2_state:" + state);
                return state.equals(storedState);
            }
        }
    }

    // 3. 콜백 처리
    @RestController
    @RequestMapping("/oauth2/callback")
    public class OAuth2CallbackController {
        
        private final OAuth2TokenService tokenService;
        private final UserService userService;

        @GetMapping("/{provider}")
        public ResponseEntity<AuthResponse> handleCallback(
            @PathVariable String provider,
            @RequestParam String code,
            @RequestParam String state) {

            // state 검증
            if (!stateService.validateState(state)) {
                throw new OAuth2AuthenticationException(
                    "Invalid state parameter");
            }

            // 액세스 토큰 획득
            OAuth2TokenResponse tokenResponse = 
                tokenService.getAccessToken(provider, code);

            // 사용자 정보 조회
            OAuth2UserInfo userInfo = 
                tokenService.getUserInfo(provider, tokenResponse);

            // 사용자 처리 (회원가입 또는 로그인)
            User user = userService
                .processOAuth2User(provider, userInfo);

            // 자체 토큰 발급
            AuthToken authToken = tokenService
                .createAuthToken(user);

            return ResponseEntity.ok(new AuthResponse(authToken));
        }
    }
}
```

## 2. OAuth2 토큰 및 사용자 정보 처리

```java
@Service
public class OAuth2TokenService {
    
    private final WebClient webClient;
    private final OAuth2ClientProperties clientProperties;

    // 1. 액세스 토큰 획득
    public OAuth2TokenResponse getAccessToken(
        String provider, 
        String authorizationCode) {
        
        OAuth2ClientProperties props = 
            clientProperties.get(provider);

        return webClient.post()
            .uri(props.getTokenUri())
            .header(HttpHeaders.CONTENT_TYPE, 
                MediaType.APPLICATION_FORM_URLENCODED_VALUE)
            .body(BodyInserters
                .fromFormData("client_id", props.getClientId())
                .with("client_secret", props.getClientSecret())
                .with("code", authorizationCode)
                .with("redirect_uri", props.getRedirectUri())
                .with("grant_type", "authorization_code"))
            .retrieve()
            .bodyToMono(OAuth2TokenResponse.class)
            .block();
    }

    // 2. 사용자 정보 조회
    public OAuth2UserInfo getUserInfo(
        String provider, 
        OAuth2TokenResponse tokenResponse) {
        
        OAuth2ClientProperties props = 
            clientProperties.get(provider);

        return webClient.get()
            .uri(props.getUserInfoUri())
            .header(HttpHeaders.AUTHORIZATION, 
                "Bearer " + tokenResponse.getAccessToken())
            .retrieve()
            .bodyToMono(getProviderUserInfoClass(provider))
            .map(userInfo -> convertToOAuth2UserInfo(provider, userInfo))
            .block();
    }

    // 3. 토큰 갱신
    @Scheduled(fixedRate = 3600000) // 1시간마다
    public void refreshExpiredTokens() {
        List<OAuth2Connection> expiredConnections = 
            oauth2ConnectionRepository
                .findExpiredConnections();

        for (OAuth2Connection connection : expiredConnections) {
            try {
                OAuth2TokenResponse newToken = refreshToken(
                    connection.getProvider(),
                    connection.getRefreshToken()
                );
                
                updateOAuth2Connection(connection, newToken);
            } catch (Exception e) {
                handleTokenRefreshError(connection, e);
            }
        }
    }
}
```

## 3. 사용자 프로필 통합 관리

```java
@Service
public class OAuth2UserService {

    private final UserRepository userRepository;
    private final OAuth2ConnectionRepository connectionRepository;

    // 1. OAuth2 사용자 처리
    @Transactional
    public User processOAuth2User(
        String provider, 
        OAuth2UserInfo userInfo) {

        // 기존 연동 확인
        OAuth2Connection connection = connectionRepository
            .findByProviderAndProviderId(
                provider, 
                userInfo.getId());

        if (connection != null) {
            // 기존 사용자 업데이트
            User existingUser = connection.getUser();
            updateExistingUser(existingUser, userInfo);
            return existingUser;
        }

        // 이메일로 기존 사용자 확인
        User existingUser = userRepository
            .findByEmail(userInfo.getEmail());

        if (existingUser != null) {
            // 기존 계정에 소셜 연동 추가
            createOAuth2Connection(existingUser, provider, userInfo);
            return existingUser;
        }

        // 새 사용자 생성
        return createNewUser(provider, userInfo);
    }

    // 2. 계정 연동
    public void linkOAuth2Account(
        User user, 
        String provider, 
        OAuth2UserInfo userInfo) {
        
        // 중복 연동 확인
        if (connectionRepository.existsByProviderAndProviderId(
            provider, userInfo.getId())) {
            throw new OAuth2AuthenticationException(
                "Account already linked");
        }

        // 연동 정보 저장
        createOAuth2Connection(user, provider, userInfo);
    }

    // 3. 연동 해제
    @Transactional
    public void unlinkOAuth2Account(User user, String provider) {
        OAuth2Connection connection = connectionRepository
            .findByUserAndProvider(user, provider)
            .orElseThrow(() -> new OAuth2AuthenticationException(
                "No linked account found"));

        // 마지막 로그인 방식 확인
        if (isLastLoginMethod(user, provider)) {
            throw new OAuth2AuthenticationException(
                "Cannot unlink last login method");
        }

        connectionRepository.delete(connection);
    }
}
```

## 4. OAuth 2.0 보안 및 예외 처리 전략

```java
@Service
public class OAuth2SecurityService {

    // 1. PKCE(Proof Key for Code Exchange) 구현
    @Service
    public class PkceService {
        private final RedisTemplate<String, String> redisTemplate;

        public PkceData generatePkceData() {
            String codeVerifier = generateSecureRandomString(64);
            String codeChallenge = generateCodeChallenge(codeVerifier);

            // Redis에 code verifier 저장
            String key = "pkce:" + codeChallenge;
            redisTemplate.opsForValue().set(key, codeVerifier, 10, TimeUnit.MINUTES);

            return new PkceData(codeVerifier, codeChallenge);
        }

        public String validateAndGetCodeVerifier(String codeChallenge) {
            String key = "pkce:" + codeChallenge;
            String codeVerifier = redisTemplate.opsForValue().get(key);
            
            if (codeVerifier == null) {
                throw new OAuth2AuthenticationException("Invalid PKCE challenge");
            }
            
            redisTemplate.delete(key);
            return codeVerifier;
        }

        private String generateCodeChallenge(String codeVerifier) {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(codeVerifier.getBytes());
            return Base64URL.encode(hash);
        }
    }

    // 2. 토큰 보안 관리
    @Service
    public class TokenSecurityManager {
        private final OAuth2TokenRepository tokenRepository;
        private final SecurityEventPublisher eventPublisher;

        public void validateTokenUsage(OAuth2TokenResponse tokenResponse) {
            // 토큰 재사용 감지
            if (isTokenReused(tokenResponse.getAccessToken())) {
                handleTokenReuse(tokenResponse);
                throw new OAuth2SecurityException("Token reuse detected");
            }

            // IP 기반 검증
            if (!isValidIPForToken(tokenResponse.getAccessToken())) {
                eventPublisher.publishSecurityEvent(
                    new SuspiciousTokenUsageEvent(tokenResponse));
                throw new OAuth2SecurityException("Invalid token usage location");
            }
        }

        @Scheduled(fixedRate = 3600000) // 매시간 실행
        public void auditTokenUsage() {
            List<OAuth2TokenUsage> suspiciousUsages = 
                findSuspiciousTokenUsage();
                
            for (OAuth2TokenUsage usage : suspiciousUsages) {
                handleSuspiciousTokenUsage(usage);
            }
        }
    }

    // 3. OAuth 예외 처리
    @ControllerAdvice
    public class OAuth2ExceptionHandler {
        private final ErrorResponseBuilder errorResponseBuilder;
        private final SecurityEventPublisher eventPublisher;

        @ExceptionHandler(OAuth2AuthenticationException.class)
        public ResponseEntity<ErrorResponse> handleOAuth2AuthenticationException(
            OAuth2AuthenticationException ex) {

            logAuthenticationFailure(ex);
            eventPublisher.publishSecurityEvent(
                new OAuth2AuthenticationFailureEvent(ex));

            return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(errorResponseBuilder.build(ex));
        }

        @ExceptionHandler(OAuth2SecurityException.class)
        public ResponseEntity<ErrorResponse> handleOAuth2SecurityException(
            OAuth2SecurityException ex) {

            notifySecurityTeam(ex);
            eventPublisher.publishSecurityEvent(
                new OAuth2SecurityIncidentEvent(ex));

            return ResponseEntity
                .status(HttpStatus.FORBIDDEN)
                .body(errorResponseBuilder.build(ex));
        }
    }

    // 4. 로깅 및 감사
    @Service
    @Slf4j
    public class OAuth2AuditService {
        private final AuditEventRepository auditRepository;

        public void auditOAuth2Event(OAuth2AuditEvent event) {
            AuditLog log = AuditLog.builder()
                .eventType(event.getType())
                .provider(event.getProvider())
                .userId(event.getUserId())
                .ipAddress(event.getIpAddress())
                .userAgent(event.getUserAgent())
                .status(event.getStatus())
                .errorDetails(event.getErrorDetails())
                .timestamp(Instant.now())
                .build();

            auditRepository.save(log);

            if (event.isSecurity()) {
                notifySecurityTeam(event);
            }
        }

        @Scheduled(cron = "0 0 * * * *") // 매시간
        public void analyzeOAuth2Patterns() {
            // 비정상 패턴 분석
            List<AuditLog> recentLogs = 
                auditRepository.findRecentLogs(Duration.ofHours(1));
                
            PatternAnalysisResult analysis = 
                analyzePatterns(recentLogs);
                
            if (analysis.hasAnomalies()) {
                handleAnomalies(analysis);
            }
        }
    }

    // 5. Rate Limiting
    @Component
    public class OAuth2RateLimiter {
        private final RateLimiter rateLimiter;
        private final SecurityConfig securityConfig;

        public void checkRateLimit(String provider, String ipAddress) {
            String key = String.format("oauth2_rate:%s:%s", provider, ipAddress);
            
            if (!rateLimiter.tryAcquire(key, 
                securityConfig.getOAuth2RateLimit())) {
                
                throw new OAuth2SecurityException(
                    "Rate limit exceeded for OAuth2 requests");
            }
        }

        // IP 기반 차단
        public void blockSuspiciousIP(String ipAddress, Duration duration) {
            String key = "oauth2_blocked:" + ipAddress;
            redisTemplate.opsForValue().set(key, "blocked", duration);
        }
    }
}
```

이러한 보안 및 예외 처리 전략을 통해:

1. PKCE 구현
    - 코드 인젝션 공격 방지
    - 인증 코드 인터셉트 방지
    - 보안 강화

2. 토큰 보안
    - 재사용 감지
    - IP 기반 검증
    - 사용 패턴 분석

3. 예외 처리
    - 세분화된 예외 처리
    - 보안 이벤트 발행
    - 에러 응답 표준화

4. 감사 및 모니터링
    - 상세 로깅
    - 패턴 분석
    - 보안팀 알림

5. 접근 제어
    - 비율 제한
    - IP 차단
    - 이상 행위 감지

특히 중요한 보안 고려사항:
- 토큰 노출 방지
- 상태 관리
- 사용자 검증
- 감사 추적

이를 통해 안전한 OAuth 2.0 인증을 구현할 수 있습니다.