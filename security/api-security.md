# API 보안 시스템 설계

## 1. API 인증과 인가 시스템

```java
@Configuration
public class ApiSecurityConfig {

    // 1. API 키 관리
    @Service
    public class ApiKeyManager {
        private final ApiKeyRepository apiKeyRepository;
        private final RedisTemplate<String, ApiKeyDetails> redisTemplate;

        public ApiKey generateApiKey(String clientId, ApiKeySpec spec) {
            String apiKey = generateSecureRandomKey();
            String hashedKey = hashApiKey(apiKey);

            // API 키 저장
            ApiKey key = ApiKey.builder()
                .clientId(clientId)
                .hashedKey(hashedKey)
                .permissions(spec.getPermissions())
                .rateLimit(spec.getRateLimit())
                .expiryDate(spec.getExpiryDate())
                .build();

            apiKeyRepository.save(key);
            cacheApiKey(apiKey, key);

            return key;
        }

        private void cacheApiKey(String apiKey, ApiKey details) {
            String cacheKey = "api_key:" + hashApiKey(apiKey);
            redisTemplate.opsForValue().set(
                cacheKey,
                details,
                Duration.ofHours(24)
            );
        }

        public ApiKeyDetails validateApiKey(String apiKey) {
            String hashedKey = hashApiKey(apiKey);
            ApiKeyDetails details = getFromCache(hashedKey);

            if (details == null) {
                details = apiKeyRepository.findByHashedKey(hashedKey)
                    .orElseThrow(() -> new InvalidApiKeyException(
                        "Invalid API key"));
                cacheApiKey(apiKey, details);
            }

            if (isApiKeyExpired(details)) {
                throw new ApiKeyExpiredException("API key expired");
            }

            return details;
        }
    }

    // 2. 요청 인증 필터
    @Component
    public class ApiAuthenticationFilter extends OncePerRequestFilter {
        
        private final ApiKeyManager apiKeyManager;
        private final RateLimiter rateLimiter;

        @Override
        protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

            try {
                String apiKey = extractApiKey(request);
                ApiKeyDetails keyDetails = apiKeyManager.validateApiKey(apiKey);

                // 비율 제한 확인
                if (!rateLimiter.tryAcquire(keyDetails.getClientId())) {
                    throw new RateLimitExceededException(
                        "Rate limit exceeded");
                }

                // 보안 컨텍스트 설정
                SecurityContextHolder.getContext()
                    .setAuthentication(createAuthentication(keyDetails));

                // 요청 메타데이터 기록
                auditRequest(request, keyDetails);

            } catch (ApiSecurityException e) {
                handleSecurityException(response, e);
                return;
            }

            chain.doFilter(request, response);
        }

        private String extractApiKey(HttpServletRequest request) {
            String apiKey = request.getHeader("X-API-Key");
            if (apiKey == null) {
                apiKey = request.getParameter("api_key");
            }
            if (apiKey == null) {
                throw new MissingApiKeyException("API key is required");
            }
            return apiKey;
        }
    }
}
```

## 2. 요청 검증 및 보안 처리

```java
@Service
public class ApiSecurityService {

    // 1. 요청 검증
    @Component
    public class RequestValidator {
        private final SignatureVerifier signatureVerifier;
        private final RequestSanitizer requestSanitizer;

        public void validateRequest(HttpServletRequest request) {
            // 1. 요청 서명 검증
            verifyRequestSignature(request);

            // 2. 입력 값 검증 및 살균
            sanitizeRequest(request);

            // 3. 요청 타임스탬프 검증
            validateTimestamp(request);
        }

        private void verifyRequestSignature(HttpServletRequest request) {
            String signature = request.getHeader("X-Signature");
            String timestamp = request.getHeader("X-Timestamp");
            String payload = extractRequestPayload(request);

            if (!signatureVerifier.verify(payload, signature, timestamp)) {
                throw new InvalidSignatureException(
                    "Invalid request signature");
            }
        }

        private void validateTimestamp(HttpServletRequest request) {
            String timestamp = request.getHeader("X-Timestamp");
            long requestTime = Long.parseLong(timestamp);
            long currentTime = System.currentTimeMillis();

            if (Math.abs(currentTime - requestTime) > 
                TimeUnit.MINUTES.toMillis(5)) {
                throw new RequestTimestampException(
                    "Request timestamp too old");
            }
        }
    }

    // 2. 입력 검증 및 살균
    @Component
    public class RequestSanitizer {
        
        public String sanitizeInput(String input) {
            // XSS 방지
            input = stripXSS(input);
            
            // SQL Injection 방지
            input = escapeSqlCharacters(input);
            
            // 문자 인코딩 정규화
            input = normalizeString(input);
            
            return input;
        }

        private String stripXSS(String input) {
            // HTML 태그 제거
            input = input.replaceAll("<[^>]*>", "");
            
            // 스크립트 이벤트 제거
            input = input.replaceAll("javascript:", "");
            input = input.replaceAll("on\\w+\\s*=", "");
            
            return input;
        }

        private String escapeSqlCharacters(String input) {
            return input.replace("'", "''")
                       .replace("\\", "\\\\")
                       .replace("%", "\\%")
                       .replace("_", "\\_");
        }
    }
}

## 3. API 모니터링 및 보안 감사

```java
@Service
public class ApiMonitoringService {

    // 1. 요청 모니터링
    @Component
    public class RequestMonitor {
        private final MeterRegistry meterRegistry;
        private final AnomalyDetector anomalyDetector;

        @Async
        public void monitorRequest(
            ApiRequest request, 
            ApiResponse response) {
            
            // 응답 시간 기록
            recordResponseTime(request, response);
            
            // 오류율 모니터링
            if (response.isError()) {
                recordError(request, response);
            }
            
            // 비정상 패턴 감지
            if (anomalyDetector.detectAnomaly(request)) {
                handleAnomaly(request);
            }
        }

        private void recordResponseTime(
            ApiRequest request, 
            ApiResponse response) {
            
            meterRegistry.timer("api.response.time",
                "endpoint", request.getEndpoint(),
                "client", request.getClientId())
                .record(response.getResponseTime());
        }
    }

    // 2. 보안 감사
    @Service
    public class SecurityAuditService {
        private final AuditEventRepository auditRepository;
        private final AlertService alertService;

        public void auditSecurityEvent(SecurityEvent event) {
            // 감사 로그 저장
            AuditLog log = AuditLog.builder()
                .eventType(event.getType())
                .clientId(event.getClientId())
                .endpoint(event.getEndpoint())
                .ipAddress(event.getIpAddress())
                .timestamp(Instant.now())
                .details(event.getDetails())
                .build();

            auditRepository.save(log);

            // 심각도에 따른 알림
            if (event.getSeverity().isHigh()) {
                alertService.sendSecurityAlert(event);
            }
        }

        @Scheduled(fixedRate = 300000) // 5분마다
        public void analyzeSecurityPatterns() {
            List<AuditLog> recentLogs = 
                auditRepository.findRecentLogs(Duration.ofMinutes(5));
                
            SecurityAnalysis analysis = 
                analyzeSecurityPatterns(recentLogs);
                
            if (analysis.hasThreats()) {
                handleSecurityThreats(analysis.getThreats());
            }
        }
    }
}
```

이러한 API 보안 시스템을 통해:

1. 인증/인가
    - API 키 관리
    - 요청 검증
    - 접근 제어

2. 요청 보안
    - 서명 검증
    - 입력 검증
    - XSS/SQL Injection 방지

3. 모니터링/감사
    - 실시간 모니터링
    - 보안 감사
    - 위협 감지

특히 중요한 보안 고려사항:
- API 키 안전성
- 요청 무결성
- 입력 데이터 검증
- 이상 행위 감지

면접관: 실제 서비스에서 API 보안을 구현할 때의 성능과 보안 사이의 트레이드오프는 어떻게 처리하시겠습니까?

# API 보안의 성능 최적화 전략

## 1. 캐싱 및 메모리 최적화

```java
@Service
public class ApiSecurityOptimizer {

    // 1. 다층 캐싱 전략
    @Service
    public class SecurityCacheManager {
        private final LoadingCache<String, ApiKeyDetails> localCache;
        private final RedisTemplate<String, ApiKeyDetails> redisCache;

        public SecurityCacheManager() {
            this.localCache = Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(5))
                .recordStats() // 캐시 성능 모니터링
                .build(key -> loadFromRedis(key));
        }

        public ApiKeyDetails getApiKeyDetails(String apiKey) {
            String cacheKey = "api_key:" + hashApiKey(apiKey);
            
            try {
                // L1 캐시 (로컬 메모리)
                return localCache.get(cacheKey);
            } catch (Exception e) {
                // L2 캐시 (Redis)
                ApiKeyDetails details = redisCache.opsForValue().get(cacheKey);
                if (details != null) {
                    localCache.put(cacheKey, details);
                    return details;
                }
                // DB 조회
                return loadFromDatabase(apiKey);
            }
        }

        // 캐시 성능 모니터링
        @Scheduled(fixedRate = 60000)
        public void monitorCachePerformance() {
            CacheStats stats = localCache.stats();
            
            // 히트율이 낮은 경우 캐시 크기 조정
            if (stats.hitRate() < 0.7) {
                adjustCacheSize(stats);
            }
        }
    }

    // 2. 병렬 처리 최적화
    @Service
    public class ParallelSecurityProcessor {
        private final ExecutorService executorService;

        public CompletableFuture<SecurityValidationResult> validateRequestAsync(
            HttpServletRequest request) {
            
            return CompletableFuture.supplyAsync(() -> {
                List<CompletableFuture<ValidationResult>> futures = Arrays.asList(
                    validateSignature(request),
                    validateApiKey(request),
                    validateRateLimit(request)
                );

                return futures.stream()
                    .map(CompletableFuture::join)
                    .reduce(ValidationResult::combine)
                    .orElseThrow();
            }, executorService);
        }
    }
}
```

## 2. 요청 처리 최적화

```java
@Service
public class OptimizedRequestProcessor {

    // 1. 경량화된 보안 체크
    @Component
    public class LightweightSecurityChecker {
        private final BloomFilter<String> blacklistedTokens;
        private final RateLimiter rateLimiter;

        public ValidationResult quickSecurityCheck(HttpServletRequest request) {
            // 블룸 필터를 사용한 빠른 블랙리스트 체크
            String token = request.getHeader("Authorization");
            if (blacklistedTokens.mightContain(token)) {
                return ValidationResult.failed("Potentially blacklisted token");
            }

            // 토큰 구조 빠른 검증
            if (!isValidTokenFormat(token)) {
                return ValidationResult.failed("Invalid token format");
            }

            return ValidationResult.success();
        }
    }

    // 2. 우선순위 기반 처리
    @Component
    public class PriorityRequestHandler {
        
        public void processRequest(HttpServletRequest request) {
            SecurityLevel securityLevel = determineSecurityLevel(request);
            
            switch (securityLevel) {
                case HIGH:
                    applyFullSecurityChecks(request);
                    break;
                case MEDIUM:
                    applyStandardSecurityChecks(request);
                    break;
                case LOW:
                    applyBasicSecurityChecks(request);
                    break;
            }
        }

        private SecurityLevel determineSecurityLevel(
            HttpServletRequest request) {
            
            return SecurityLevel.valueOf(
                Optional.ofNullable(request.getHeader("X-Security-Level"))
                    .orElse(DEFAULT_SECURITY_LEVEL)
            );
        }
    }
}
```

## 3. 리소스 사용 최적화

```java
@Service
public class ResourceOptimizer {

    // 1. 동적 스레드 풀 관리
    @Component
    public class DynamicThreadPoolManager {
        private final ThreadPoolExecutor executor;
        private final LoadMonitor loadMonitor;

        @Scheduled(fixedRate = 1000)
        public void adjustThreadPool() {
            SystemLoad currentLoad = loadMonitor.getCurrentLoad();
            
            int optimalThreads = calculateOptimalThreads(currentLoad);
            
            executor.setCorePoolSize(optimalThreads);
            executor.setMaximumPoolSize(optimalThreads * 2);
        }

        private int calculateOptimalThreads(SystemLoad load) {
            int processors = Runtime.getRuntime().availableProcessors();
            double loadFactor = Math.min(load.getCpuUsage(), 0.8);
            
            return (int) (processors * loadFactor);
        }
    }

    // 2. 메모리 사용 최적화
    @Component
    public class MemoryOptimizer {
        
        public void optimizeRequestProcessing(HttpServletRequest request) {
            // 요청 데이터 스트리밍 처리
            if (isLargePayload(request)) {
                processLargePayloadInChunks(request);
            } else {
                processNormalPayload(request);
            }
        }

        private void processLargePayloadInChunks(HttpServletRequest request) {
            try (BufferedReader reader = request.getReader()) {
                StringBuilder chunk = new StringBuilder();
                char[] buffer = new char[8192];
                int bytesRead;
                
                while ((bytesRead = reader.read(buffer)) != -1) {
                    chunk.append(buffer, 0, bytesRead);
                    
                    if (chunk.length() >= CHUNK_SIZE) {
                        processChunk(chunk.toString());
                        chunk.setLength(0);
                    }
                }
                
                if (chunk.length() > 0) {
                    processChunk(chunk.toString());
                }
            }
        }
    }
}
```

## 4. 모니터링 및 최적화 피드백

```java
@Service
public class PerformanceMonitor {

    // 1. 성능 메트릭 수집
    @Component
    public class SecurityMetricsCollector {
        private final MeterRegistry registry;

        public void recordSecurityCheck(String checkType, long duration) {
            registry.timer("security.check.duration",
                "type", checkType)
                .record(duration, TimeUnit.MILLISECONDS);
        }

        @Scheduled(fixedRate = 60000)
        public void analyzePerformance() {
            Map<String, Double> averageDurations = 
                calculateAverageDurations();
                
            // 성능 임계값 초과 검사
            averageDurations.forEach((checkType, avgDuration) -> {
                if (avgDuration > getThreshold(checkType)) {
                    optimizeSecurityCheck(checkType);
                }
            });
        }
    }

    // 2. 자동 최적화
    @Service
    public class AutoOptimizer {
        
        public void optimizeBasedOnMetrics() {
            SecurityMetrics metrics = collectCurrentMetrics();
            
            // 캐시 설정 최적화
            if (metrics.getCacheHitRate() < 0.8) {
                adjustCacheSettings(metrics);
            }

            // 스레드 풀 최적화
            if (metrics.getThreadPoolUtilization() > 0.9) {
                adjustThreadPool(metrics);
            }

            // 타임아웃 설정 최적화
            if (metrics.getTimeoutRate() > 0.01) {
                adjustTimeouts(metrics);
            }
        }

        private void adjustCacheSettings(SecurityMetrics metrics) {
            int optimalSize = calculateOptimalCacheSize(
                metrics.getCacheStats());
                
            cacheManager.resize(optimalSize);
        }
    }
}
```

이러한 성능 최적화 전략을 통해:

1. 캐싱 최적화
    - 다층 캐싱
    - 메모리 효율성
    - 캐시 성능 모니터링

2. 처리 최적화
    - 병렬 처리
    - 우선순위 기반 처리
    - 경량화된 검증

3. 리소스 관리
    - 동적 스레드풀
    - 메모리 사용 최적화
    - 청크 처리

4. 모니터링
    - 실시간 성능 추적
    - 자동 최적화
    - 피드백 루프

이를 통해 보안성을 유지하면서도 높은 성능을 달성할 수 있습니다.