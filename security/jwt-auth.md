# JWT(Json Web Token) 기반 인증 시스템 설계

## 1. JWT 토큰 관리

```java
@Service
public class JwtTokenService {
    
    @Value("${jwt.secret}")
    private String secretKey;
    
    private final long ACCESS_TOKEN_VALIDITY = 3600000; // 1시간
    private final long REFRESH_TOKEN_VALIDITY = 604800000; // 7일

    // 1. 토큰 생성
    public TokenPair generateTokenPair(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("username", userDetails.getUsername());
        claims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));

        String accessToken = createToken(claims, ACCESS_TOKEN_VALIDITY);
        String refreshToken = createRefreshToken(userDetails.getUsername());

        return new TokenPair(accessToken, refreshToken);
    }

    private String createToken(Map<String, Object> claims, long validity) {
        return Jwts.builder()
            .setClaims(claims)
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + validity))
            .signWith(SignatureAlgorithm.HS512, secretKey)
            .compact();
    }

    // 2. 토큰 검증
    public boolean validateToken(String token) {
        try {
            Jws<Claims> claims = Jwts.parser()
                .setSigningKey(secretKey)
                .parseClaimsJws(token);
                
            // 만료 시간 검증
            return !claims.getBody()
                .getExpiration()
                .before(new Date());
                
        } catch (JwtException | IllegalArgumentException e) {
            throw new InvalidTokenException("Invalid JWT token");
        }
    }

    // 3. 토큰에서 정보 추출
    public Authentication getAuthentication(String token) {
        Claims claims = extractClaims(token);
        
        Collection<? extends GrantedAuthority> authorities =
            Arrays.stream(claims.get("roles").toString().split(","))
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
                
        UserDetails principal = new User(
            claims.get("username").toString(), 
            "", 
            authorities);
            
        return new UsernamePasswordAuthenticationToken(
            principal, 
            "", 
            authorities);
    }
}
```

## 2. 토큰 보안 강화

```java
@Component
public class TokenSecurityEnhancer {

    private final RedisTemplate<String, String> redisTemplate;

    // 1. 토큰 블랙리스트 관리
    public class TokenBlacklistManager {
        
        public void blacklistToken(String token, Claims claims) {
            long expirationTime = 
                claims.getExpiration().getTime() - 
                System.currentTimeMillis();
                
            redisTemplate.opsForValue().set(
                "blacklist:" + token,
                "blocked",
                expirationTime,
                TimeUnit.MILLISECONDS
            );
        }

        public boolean isBlacklisted(String token) {
            return Boolean.TRUE.equals(redisTemplate.hasKey(
                "blacklist:" + token));
        }
    }

    // 2. 토큰 Rotation
    public class TokenRotationManager {
        
        public TokenPair rotateTokens(String refreshToken) {
            Claims claims = validateRefreshToken(refreshToken);
            
            // 이전 리프레시 토큰 무효화
            invalidateRefreshToken(refreshToken);
            
            // 새로운 토큰 쌍 생성
            return generateNewTokenPair(claims.getSubject());
        }

        private void invalidateRefreshToken(String refreshToken) {
            redisTemplate.delete("refresh_token:" + refreshToken);
        }
    }

    // 3. 토큰 지문(Fingerprint)
    public class TokenFingerprintManager {
        
        public TokenWithFingerprint generateTokenWithFingerprint(
            UserDetails user) {
            
            String fingerprint = generateSecureFingerprint();
            String fingerprintHash = hashFingerprint(fingerprint);
            
            // 토큰에 지문 해시 포함
            Map<String, Object> claims = new HashMap<>();
            claims.put("fph", fingerprintHash);
            
            String token = generateToken(claims);
            
            return new TokenWithFingerprint(token, fingerprint);
        }

        public boolean verifyFingerprint(
            String token, 
            String providedFingerprint) {
            
            Claims claims = extractClaims(token);
            String storedHash = claims.get("fph", String.class);
            String providedHash = hashFingerprint(providedFingerprint);
            
            return storedHash.equals(providedHash);
        }
    }
}
```

## 3. 보안 필터 구현

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtTokenService tokenService;
    private final TokenSecurityEnhancer securityEnhancer;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain) throws ServletException, IOException {
        
        try {
            String token = extractToken(request);
            
            if (token != null && tokenService.validateToken(token)) {
                // 토큰 블랙리스트 확인
                if (securityEnhancer.isBlacklisted(token)) {
                    throw new InvalidTokenException("Token is blacklisted");
                }
                
                // 토큰 지문 확인
                String fingerprint = extractFingerprint(request);
                if (!securityEnhancer.verifyFingerprint(token, fingerprint)) {
                    throw new InvalidTokenException(
                        "Invalid token fingerprint");
                }
                
                // 인증 정보 설정
                Authentication auth = tokenService.getAuthentication(token);
                SecurityContextHolder.getContext()
                    .setAuthentication(auth);
            }
            
        } catch (JwtException e) {
            SecurityContextHolder.clearContext();
            response.sendError(
                HttpServletResponse.SC_UNAUTHORIZED, 
                e.getMessage());
            return;
        }
        
        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && 
            bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

## 4. 토큰 갱신 및 예외 처리

```java
@RestController
@RequestMapping("/api/auth")
public class TokenController {

    private final JwtTokenService tokenService;
    private final TokenSecurityEnhancer securityEnhancer;

    // 1. 토큰 갱신
    @PostMapping("/refresh")
    public TokenResponse refreshToken(
        @RequestBody RefreshTokenRequest request) {
        
        try {
            // 리프레시 토큰 검증
            if (!tokenService.validateRefreshToken(
                request.getRefreshToken())) {
                throw new InvalidTokenException(
                    "Invalid refresh token");
            }
            
            // 새로운 토큰 쌍 생성
            TokenPair newTokens = securityEnhancer
                .rotateTokens(request.getRefreshToken());
                
            return new TokenResponse(newTokens);
            
        } catch (Exception e) {
            throw new TokenRefreshException(
                "Failed to refresh token", e);
        }
    }

    // 2. 토큰 무효화
    @PostMapping("/logout")
    public ResponseEntity<?> logout(
        @RequestHeader("Authorization") String token) {
        
        try {
            String actualToken = token.substring(7);  // "Bearer " 제거
            
            // 토큰 블랙리스트에 추가
            Claims claims = tokenService.extractClaims(actualToken);
            securityEnhancer.blacklistToken(actualToken, claims);
            
            // 리프레시 토큰 무효화
            securityEnhancer.invalidateRefreshToken(
                claims.getSubject());
                
            return ResponseEntity.ok()
                .body("Successfully logged out");
                
        } catch (Exception e) {
            return ResponseEntity.status(
                HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Logout failed");
        }
    }
}
```

이러한 JWT 기반 인증 시스템을 통해:

1. 토큰 관리
    - 안전한 토큰 생성
    - 유효성 검증
    - 정보 추출

2. 보안 강화
    - 토큰 블랙리스트
    - 토큰 회전
    - 토큰 지문

3. 인증 필터
    - 요청 인증
    - 토큰 검증
    - 보안 컨텍스트 관리

4. 토큰 갱신
    - 리프레시 토큰 처리
    - 안전한 로그아웃
    - 예외 처리

를 구현할 수 있습니다.

면접관: Refresh Token을 사용할 때의 보안 고려사항은 무엇인가요?

## Refresh Token 보안 전략

```java
@Service
public class RefreshTokenSecurityService {

    // 1. Refresh Token 저장 및 관리
    @Service
    public class RefreshTokenStore {
        private final RedisTemplate<String, RefreshTokenData> redisTemplate;
        private final UserDeviceRepository deviceRepository;

        public void storeRefreshToken(String userId, 
                                    RefreshTokenData tokenData) {
            String key = generateRefreshTokenKey(userId, tokenData.getDeviceId());
            
            // 디바이스별 토큰 관리
            redisTemplate.opsForHash()
                .put("refresh_tokens:" + userId,
                     tokenData.getDeviceId(),
                     tokenData);

            // 토큰 만료 설정
            redisTemplate.expire(
                "refresh_tokens:" + userId,
                30,
                TimeUnit.DAYS
            );

            // 디바이스 정보 저장
            saveDeviceInfo(userId, tokenData.getDeviceInfo());
        }

        public void validateDeviceAndToken(String userId, 
                                         String refreshToken,
                                         DeviceInfo currentDevice) {
            RefreshTokenData storedToken = getStoredToken(userId, 
                currentDevice.getDeviceId());

            if (storedToken == null || 
                !storedToken.getToken().equals(refreshToken)) {
                throw new InvalidRefreshTokenException(
                    "Invalid refresh token for device");
            }

            // 디바이스 지문 검증
            if (!verifyDeviceFingerprint(
                storedToken.getDeviceInfo(), 
                currentDevice)) {
                throw new DeviceMismatchException(
                    "Device fingerprint mismatch");
            }
        }
    }

    // 2. 회전(Rotation) 정책 구현
    @Service
    public class TokenRotationPolicy {
        private final RefreshTokenStore tokenStore;
        private final SecurityEventPublisher eventPublisher;

        public TokenPair rotateTokens(String userId, 
                                    String currentRefreshToken,
                                    DeviceInfo deviceInfo) {
            // 현재 토큰 검증
            tokenStore.validateDeviceAndToken(
                userId, 
                currentRefreshToken, 
                deviceInfo);

            // 재사용 감지
            if (isTokenReused(userId, currentRefreshToken)) {
                handleTokenReuse(userId);
                throw new TokenReuseException(
                    "Refresh token reuse detected");
            }

            // 새 토큰 쌍 생성
            TokenPair newTokens = generateNewTokenPair(userId);

            // Refresh Token 저장
            tokenStore.storeRefreshToken(userId, 
                RefreshTokenData.builder()
                    .token(newTokens.getRefreshToken())
                    .deviceId(deviceInfo.getDeviceId())
                    .deviceInfo(deviceInfo)
                    .issuedAt(Instant.now())
                    .build()
            );

            // 이전 토큰 무효화 이력 저장
            markTokenAsUsed(currentRefreshToken);

            return newTokens;
        }

        private boolean isTokenReused(String userId, String refreshToken) {
            return redisTemplate.opsForSet()
                .isMember("used_tokens:" + userId, refreshToken);
        }

        private void handleTokenReuse(String userId) {
            // 모든 리프레시 토큰 무효화
            tokenStore.revokeAllTokens(userId);
            
            // 보안 이벤트 발행
            eventPublisher.publishSecurityEvent(
                new TokenReuseEvent(userId));
        }
    }

    // 3. 토큰 추적 및 감사
    @Service
    public class TokenAuditService {
        private final AuditEventRepository auditRepository;

        @Async
        public void auditTokenUsage(TokenUsageEvent event) {
            AuditEvent auditEvent = AuditEvent.builder()
                .userId(event.getUserId())
                .deviceId(event.getDeviceInfo().getDeviceId())
                .action(event.getAction())
                .tokenId(event.getTokenId())
                .timestamp(Instant.now())
                .ipAddress(event.getIpAddress())
                .userAgent(event.getUserAgent())
                .build();

            auditRepository.save(auditEvent);
        }

        public void detectSuspiciousActivity(String userId) {
            List<AuditEvent> recentEvents = 
                auditRepository.findRecentByUserId(
                    userId, 
                    Duration.ofHours(24)
                );

            // 의심스러운 패턴 분석
            if (hasMultipleDeviceAccess(recentEvents) || 
                hasUnusualGeographicalAccess(recentEvents)) {
                notifySecurityTeam(userId, recentEvents);
            }
        }
    }

    // 4. 자동 토큰 정리
    @Service
    public class TokenCleanupService {
        
        @Scheduled(cron = "0 0 * * * *") // 매시간 실행
        public void cleanupExpiredTokens() {
            // 만료된 리프레시 토큰 삭제
            Set<String> expiredUserIds = findUsersWithExpiredTokens();
            
            for (String userId : expiredUserIds) {
                cleanupUserTokens(userId);
            }
        }

        @Scheduled(cron = "0 0 0 * * *") // 매일 실행
        public void cleanupUnusedTokens() {
            // 30일 이상 미사용 토큰 정리
            LocalDateTime cutoffDate = 
                LocalDateTime.now().minusDays(30);
                
            List<RefreshTokenData> unusedTokens = 
                findUnusedTokens(cutoffDate);
                
            for (RefreshTokenData token : unusedTokens) {
                revokeToken(token);
                notifyUser(token.getUserId(), 
                    "Unused refresh token revoked");
            }
        }
    }
}
```

이러한 Refresh Token 보안 전략을 통해:

1. 안전한 토큰 저장
    - Redis 기반 토큰 저장
    - 디바이스별 토큰 관리
    - 만료 시간 설정

2. 토큰 회전 정책
    - 재사용 감지
    - 자동 무효화
    - 디바이스 검증

3. 감사 및 모니터링
    - 사용 기록 추적
    - 의심 활동 감지
    - 보안 이벤트 처리

4. 자동화된 관리
    - 만료 토큰 정리
    - 미사용 토큰 제거
    - 사용자 알림

특히 중요한 보안 고려사항:
- 디바이스 바인딩
- 토큰 재사용 방지
- 이상 징후 감지
- 자동 클린업

이를 통해 Refresh Token의 안전한 관리와 보안 위협 대응이 가능합니다.

# JWT 토큰의 클라이언트 측 보안 전략

## 1. 토큰 저장 및 관리

```typescript
// TokenService.ts
export class TokenService {
    private static readonly ACCESS_TOKEN_KEY = 'access_token';
    private static readonly REFRESH_TOKEN_KEY = 'refresh_token';
    
    // 1. 메모리 기반 액세스 토큰 관리
    private static accessToken: string | null = null;

    // 2. HttpOnly 쿠키를 위한 인터페이스
    public static storeTokens(tokens: TokenResponse) {
        // Access Token은 메모리에 저장
        this.accessToken = tokens.accessToken;
        
        // Refresh Token은 HttpOnly 쿠키로 저장 (서버에 요청)
        this.storeRefreshToken(tokens.refreshToken);
    }

    private static storeRefreshToken(refreshToken: string) {
        // 서버에 쿠키 설정 요청
        axios.post('/api/auth/cookie', { refreshToken }, {
            withCredentials: true  // CORS 설정 필요
        });
    }

    // 3. 토큰 암호화 저장 (필요한 경우)
    private static encryptToken(token: string): string {
        const encoder = new TextEncoder();
        const data = encoder.encode(token);
        return window.btoa(String.fromCharCode(...new Uint8Array(data)));
    }
}
```

## 2. HTTP 요청 인터셉터

```typescript
// ApiInterceptor.ts
export class ApiInterceptor {
    private static instance: AxiosInstance;

    public static setup() {
        this.instance = axios.create({
            baseURL: process.env.API_BASE_URL,
            timeout: 10000,
            withCredentials: true
        });

        this.setupInterceptors();
    }

    private static setupInterceptors() {
        // 1. 요청 인터셉터
        this.instance.interceptors.request.use(
            async config => {
                const token = TokenService.getAccessToken();
                if (token) {
                    config.headers.Authorization = `Bearer ${token}`;
                }
                
                // CSRF 토큰 추가
                const csrfToken = CsrfTokenService.getToken();
                if (csrfToken) {
                    config.headers['X-CSRF-Token'] = csrfToken;
                }
                
                return config;
            },
            error => Promise.reject(error)
        );

        // 2. 응답 인터셉터
        this.instance.interceptors.response.use(
            response => response,
            async error => {
                const originalRequest = error.config;

                if (error.response?.status === 401 && 
                    !originalRequest._retry) {
                    originalRequest._retry = true;

                    try {
                        // 토큰 갱신 시도
                        const newTokens = 
                            await TokenService.refreshTokens();
                            
                        TokenService.storeTokens(newTokens);
                        
                        // 원래 요청 재시도
                        return this.instance(originalRequest);
                    } catch (refreshError) {
                        // 리프레시 실패 시 로그아웃
                        await AuthService.logout();
                        return Promise.reject(refreshError);
                    }
                }

                return Promise.reject(error);
            }
        );
    }
}
```

## 3. 보안 유틸리티

```typescript
// SecurityUtils.ts
export class SecurityUtils {
    // 1. XSS 방지
    public static sanitizeInput(input: string): string {
        const div = document.createElement('div');
        div.textContent = input;
        return div.innerHTML;
    }

    // 2. 디바이스 지문
    public static async generateDeviceFingerprint(): Promise<string> {
        const components = [
            navigator.userAgent,
            navigator.language,
            new Date().getTimezoneOffset(),
            screen.width,
            screen.height,
            navigator.hardwareConcurrency,
            // Canvas 지문
            await this.getCanvasFingerprint(),
            // WebGL 지문
            await this.getWebGLFingerprint()
        ];

        // 해시 생성
        const fingerprint = await crypto.subtle.digest(
            'SHA-256',
            new TextEncoder().encode(components.join(''))
        );

        return Array.from(new Uint8Array(fingerprint))
            .map(b => b.toString(16).padStart(2, '0'))
            .join('');
    }

    // 3. CSRF 보호
    public static setupCsrfProtection() {
        const token = document.querySelector(
            'meta[name="csrf-token"]'
        )?.getAttribute('content');

        if (token) {
            CsrfTokenService.setToken(token);
        }
    }
}
```

## 4. 인증 상태 관리 (예: React Context 사용)

```typescript
// AuthContext.tsx
interface AuthContextType {
    isAuthenticated: boolean;
    user: User | null;
    login: (credentials: LoginCredentials) => Promise<void>;
    logout: () => Promise<void>;
    refreshAuth: () => Promise<void>;
}

export const AuthContext = createContext<AuthContextType | null>(null);

export const AuthProvider: React.FC = ({ children }) => {
    const [isAuthenticated, setIsAuthenticated] = useState(false);
    const [user, setUser] = useState<User | null>(null);

    // 1. 인증 상태 초기화
    useEffect(() => {
        const initializeAuth = async () => {
            try {
                const token = TokenService.getAccessToken();
                if (token) {
                    await refreshAuth();
                }
            } catch (error) {
                await logout();
            }
        };

        initializeAuth();
    }, []);

    // 2. 로그인 처리
    const login = async (credentials: LoginCredentials) => {
        try {
            // 디바이스 지문 생성
            const deviceFingerprint = 
                await SecurityUtils.generateDeviceFingerprint();

            const response = await axios.post('/api/auth/login', {
                ...credentials,
                deviceFingerprint
            });

            TokenService.storeTokens(response.data.tokens);
            setUser(response.data.user);
            setIsAuthenticated(true);

        } catch (error) {
            console.error('Login failed:', error);
            throw error;
        }
    };

    // 3. 로그아웃 처리
    const logout = async () => {
        try {
            await axios.post('/api/auth/logout');
        } finally {
            TokenService.clearTokens();
            setUser(null);
            setIsAuthenticated(false);
        }
    };

    // 4. 자동 토큰 갱신
    const refreshAuth = async () => {
        try {
            const deviceFingerprint = 
                await SecurityUtils.generateDeviceFingerprint();
                
            const response = await axios.post('/api/auth/refresh', {
                deviceFingerprint
            }, {
                withCredentials: true
            });

            TokenService.storeTokens(response.data.tokens);
            setUser(response.data.user);
            setIsAuthenticated(true);
        } catch (error) {
            throw error;
        }
    };

    return (
        <AuthContext.Provider value={{
            isAuthenticated,
            user,
            login,
            logout,
            refreshAuth
        }}>
            {children}
        </AuthContext.Provider>
    );
};
```

이러한 클라이언트 측 보안 전략을 통해:

1. 토큰 보안 저장
    - Access Token은 메모리에 저장
    - Refresh Token은 HttpOnly 쿠키로 관리
    - 필요시 추가 암호화

2. 요청/응답 보안
    - 자동 토큰 갱신
    - CSRF 보호
    - 에러 처리

3. 디바이스 보안
    - 디바이스 지문 생성
    - XSS 방지
    - 입력 검증

4. 상태 관리
    - 안전한 인증 상태 관리
    - 자동 로그인/로그아웃
    - 토큰 만료 처리

를 구현할 수 있습니다.