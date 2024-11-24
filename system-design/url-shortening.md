# URL 단축 서비스 설계

면접관: "bit.ly와 같은 URL 단축 서비스를 설계해주세요. 긴 URL을 짧은 URL로 변환하고, 짧은 URL을 통해 원래 URL로 리디렉션하는 기능이 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 일일 URL 생성 요청 수는 얼마나 예상되나요?
2. URL의 유효 기간이 있나요?
3. 커스텀 URL을 지원해야 하나요?
4. URL 접속 통계 기능이 필요한가요?

면접관:
1. 하루 100만 개의 새로운 URL 생성, 리디렉션은 그의 10배
2. 기본적으로 영구 보관이나, 유효기간 설정 가능해야 함
3. 프리미엄 사용자를 위한 커스텀 URL 지원 필요
4. 기본적인 접속 통계(클릭 수, 지역, 기기 등) 필요

지원자: 알겠습니다. 시스템을 설계해보겠습니다.

## 1. URL 단축 알고리즘

```java
@Service
public class UrlShortenerService {
    
    private static final String ALPHABET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    private static final int BASE = ALPHABET.length();
    
    // 1. Base64 인코딩 방식
    public String generateShortUrl(long id) {
        StringBuilder shortUrl = new StringBuilder();
        
        while (id > 0) {
            shortUrl.append(ALPHABET.charAt((int) (id % BASE)));
            id /= BASE;
        }
        
        return shortUrl.reverse().toString();
    }
    
    // 2. Counter 기반 ID 생성
    @Autowired
    private RedisTemplate<String, Long> redisTemplate;
    
    public long generateUniqueId() {
        return redisTemplate.opsForValue().increment("url:counter");
    }
}
```

## 2. URL 매핑 저장소

```java
@Entity
@Table(name = "url_mappings")
public class UrlMapping {
    @Id
    private String shortUrl;
    
    @Column(nullable = false)
    private String longUrl;
    
    @Column
    private LocalDateTime expiryDate;
    
    @Column
    private String userId;
    
    @Column
    private boolean isCustom;
    
    @CreationTimestamp
    private LocalDateTime createdAt;
}

@Repository
public class UrlRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public void saveUrl(UrlMapping mapping) {
        // 1. DB에 저장
        jdbcTemplate.update(
            "INSERT INTO url_mappings (short_url, long_url, expiry_date, user_id, is_custom) VALUES (?, ?, ?, ?, ?)",
            mapping.getShortUrl(),
            mapping.getLongUrl(),
            mapping.getExpiryDate(),
            mapping.getUserId(),
            mapping.isCustom()
        );
        
        // 2. 캐시에 저장
        if (mapping.getExpiryDate() != null) {
            redisTemplate.opsForValue().set(
                "url:" + mapping.getShortUrl(),
                mapping.getLongUrl(),
                Duration.between(LocalDateTime.now(), mapping.getExpiryDate())
            );
        } else {
            redisTemplate.opsForValue().set(
                "url:" + mapping.getShortUrl(),
                mapping.getLongUrl()
            );
        }
    }
}
```

## 3. URL 처리 및 리디렉션

```java
@RestController
@Slf4j
public class UrlController {
    
    @Autowired
    private UrlService urlService;
    
    @Autowired
    private AnalyticsService analyticsService;
    
    // 1. URL 단축
    @PostMapping("/shorten")
    public ResponseEntity<ShortenResponse> shortenUrl(@RequestBody ShortenRequest request) {
        try {
            // 입력 URL 검증
            validateUrl(request.getUrl());
            
            // URL 단축 처리
            String shortUrl = urlService.createShortUrl(request);
            
            return ResponseEntity.ok(new ShortenResponse(shortUrl));
        } catch (InvalidUrlException e) {
            return ResponseEntity.badRequest().build();
        }
    }
    
    // 2. 리디렉션 처리
    @GetMapping("/{shortUrl}")
    public ResponseEntity<Void> redirect(@PathVariable String shortUrl,
                                       HttpServletRequest request) {
        try {
            String longUrl = urlService.getLongUrl(shortUrl);
            
            // 비동기로 분석 데이터 수집
            CompletableFuture.runAsync(() -> 
                analyticsService.recordClick(shortUrl, request));
            
            return ResponseEntity.status(HttpStatus.MOVED_PERMANENTLY)
                .header(HttpHeaders.LOCATION, longUrl)
                .build();
                
        } catch (UrlNotFoundException e) {
            return ResponseEntity.notFound().build();
        }
    }
}
```

## 4. 분석 시스템

```java
@Service
public class AnalyticsService {
    
    @Autowired
    private KafkaTemplate<String, ClickEvent> kafkaTemplate;
    
    @Autowired
    private ClickstreamRepository clickstreamRepository;
    
    // 1. 클릭 이벤트 처리
    public void recordClick(String shortUrl, HttpServletRequest request) {
        ClickEvent event = ClickEvent.builder()
            .shortUrl(shortUrl)
            .timestamp(Instant.now())
            .ip(request.getRemoteAddr())
            .userAgent(request.getHeader("User-Agent"))
            .referer(request.getHeader("Referer"))
            .build();
            
        // Kafka로 이벤트 발행
        kafkaTemplate.send("url-clicks", event);
    }
    
    // 2. 집계 처리
    @Scheduled(fixedRate = 60000) // 1분마다 집계
    public void aggregateStats() {
        // 시간별 집계
        Map<String, UrlStats> hourlyStats = clickstreamRepository
            .aggregateByTimeWindow(Duration.ofHours(1));
            
        // 지역별 집계
        Map<String, GeoStats> geoStats = clickstreamRepository
            .aggregateByRegion();
            
        // 디바이스별 집계
        Map<String, DeviceStats> deviceStats = clickstreamRepository
            .aggregateByDevice();
            
        // Redis에 캐싱
        cacheStats(hourlyStats, geoStats, deviceStats);
    }
}
```

## 5. 캐싱 전략

```java
@Configuration
public class CacheConfig {
    
    // 1. 다층 캐싱 구성
    @Bean
    public CacheManager cacheManager() {
        return new LayeredCacheManager(
            new LocalCache(1000),    // L1 캐시
            new RedisCache()         // L2 캐시
        );
    }
    
    // 2. 캐시 정책
    @Bean
    public CachePolicy cachePolicy() {
        return CachePolicy.builder()
            .maxSize(10_000)
            .expireAfterWrite(Duration.ofHours(24))
            .evictionPolicy(EvictionPolicy.LRU)
            .build();
    }
}

@Service
public class UrlCacheService {
    
    @Autowired
    private CacheManager cacheManager;
    
    // 3. 캐시 워밍업
    @Scheduled(fixedRate = 3600000) // 1시간마다
    public void warmupCache() {
        List<String> popularUrls = analyticsService
            .getTopUrls(100);
            
        popularUrls.forEach(shortUrl -> {
            if (!cacheManager.exists(shortUrl)) {
                String longUrl = urlRepository.findLongUrl(shortUrl);
                cacheManager.put(shortUrl, longUrl);
            }
        });
    }
}
```

## 6. 부하 분산 및 확장성

```java
@Configuration
public class LoadBalancerConfig {
    
    // 1. 글로벌 로드 밸런싱
    @Bean
    public GlobalLoadBalancer globalLoadBalancer() {
        return GlobalLoadBalancer.builder()
            .withRegions(List.of("us-east", "eu-west", "ap-south"))
            .withDNSRouting(true)
            .withHealthCheck(healthChecker())
            .build();
    }
    
    // 2. 샤딩 전략
    @Bean
    public ShardingStrategy shardingStrategy() {
        return new ConsistentHashingStrategy(
            numShards = 128,
            replicationFactor = 3
        );
    }
}

@Service
public class ScalingService {
    
    // 3. 자동 스케일링
    public void adjustCapacity() {
        Metrics metrics = collectSystemMetrics();
        
        if (metrics.getCpuUsage() > 70 || 
            metrics.getRequestRate() > threshold) {
            scaleOut();
        }
        
        if (metrics.getCpuUsage() < 30 && 
            metrics.getRequestRate() < threshold / 2) {
            scaleIn();
        }
    }
}
```

이러한 설계를 통해:

1. 효율적인 URL 단축 및 리디렉션
2. 높은 확장성과 가용성
3. 빠른 응답 시간
4. 상세한 분석 기능
5. 효과적인 캐싱

을 제공할 수 있습니다.

면접관: 시스템의 보안 측면은 어떻게 고려하셨나요? 악의적인 사용을 어떻게 방지할 수 있을까요?

## 7. 보안 시스템 구현

### 7.1 악의적 URL 감지
```java
@Service
public class UrlSecurityService {
    
    private final PhishingDetector phishingDetector;
    private final MalwareScanner malwareScanner;
    private final RateLimiter rateLimiter;

    // 1. URL 검증
    public void validateUrl(String url) throws UnsafeUrlException {
        CompletableFuture<Boolean>[] checks = new CompletableFuture[] {
            // 피싱 사이트 체크
            CompletableFuture.supplyAsync(() -> 
                phishingDetector.check(url)),
            // 멀웨어 체크
            CompletableFuture.supplyAsync(() -> 
                malwareScanner.scan(url)),
            // 블랙리스트 체크
            CompletableFuture.supplyAsync(() -> 
                checkBlacklist(url))
        };

        // 모든 검증 완료 대기
        CompletableFuture.allOf(checks).join();
        
        if (Arrays.stream(checks).anyMatch(c -> c.join())) {
            throw new UnsafeUrlException("Malicious URL detected");
        }
    }

    // 2. 속도 제한
    @CircuitBreaker(name = "urlCreation")
    public boolean checkRateLimit(String userId) {
        return rateLimiter.tryAcquire(userId, new RateLimit(
            requests = 100,
            per = Duration.ofHours(1)
        ));
    }
}
```

### 7.2 사용자 인증 및 권한
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .and()
            .authorizeRequests()
                .antMatchers("/api/premium/**").authenticated()
                .antMatchers("/api/analytics/**").hasRole("ADMIN")
                .antMatchers("/{shortUrl}").permitAll()
            .and()
            .oauth2Login()
            .and()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 7.3 요청 검증 및 필터링
```java
@Component
public class RequestValidationFilter extends OncePerRequestFilter {
    
    private final XSSFilter xssFilter;
    private final SQLInjectionFilter sqlInjectionFilter;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain) {
        // 입력 검증
        validateRequest(request);
        
        // XSS 방지
        String cleanContent = xssFilter.clean(request.getParameter("url"));
        
        // SQL Injection 방지
        sqlInjectionFilter.validate(cleanContent);
        
        // CORS 설정
        response.setHeader("Access-Control-Allow-Origin", 
            getAllowedOrigins(request));
            
        filterChain.doFilter(request, response);
    }
}
```

### 7.4 보안 감사 및 모니터링
```java
@Service
@Slf4j
public class SecurityAuditService {
    
    private final AuditEventRepository auditRepository;
    private final AlertService alertService;

    // 1. 보안 이벤트 기록
    public void logSecurityEvent(SecurityEvent event) {
        AuditEvent auditEvent = AuditEvent.builder()
            .type(event.getType())
            .userId(event.getUserId())
            .action(event.getAction())
            .timestamp(Instant.now())
            .details(event.getDetails())
            .ipAddress(event.getIpAddress())
            .build();

        auditRepository.save(auditEvent);
    }

    // 2. 이상 행동 감지
    @Scheduled(fixedRate = 300000) // 5분마다
    public void detectAnomalies() {
        // IP별 요청 패턴 분석
        Map<String, RequestPattern> patterns = 
            analyzeRequestPatterns();

        // 이상 패턴 감지
        patterns.forEach((ip, pattern) -> {
            if (pattern.isAnomalous()) {
                handleAnomalousActivity(ip, pattern);
            }
        });
    }
}
```

### 7.5 데이터 암호화
```java
@Service
public class EncryptionService {
    
    @Value("${encryption.key}")
    private String encryptionKey;

    // 1. 민감 데이터 암호화
    public String encryptSensitiveData(String data) {
        return AES.encrypt(data, encryptionKey);
    }

    // 2. URL 파라미터 암호화
    public String encryptUrlParameters(Map<String, String> params) {
        String concatenated = params.entrySet().stream()
            .map(e -> e.getKey() + "=" + e.getValue())
            .collect(Collectors.joining("&"));
            
        return Base64.encode(concatenated);
    }
}
```

### 7.6 접근 제어 및 권한 관리
```java
@Service
public class AccessControlService {
    
    private final RoleHierarchy roleHierarchy;
    private final PermissionEvaluator permissionEvaluator;

    // 1. URL 접근 권한 검사
    @PreAuthorize("hasPermission(#shortUrl, 'URL', 'VIEW')")
    public void checkUrlAccess(String shortUrl, String userId) {
        UrlAccess access = urlAccessRepository
            .findByShortUrlAndUserId(shortUrl, userId);
            
        if (access == null || !access.hasValidAccess()) {
            throw new AccessDeniedException("Access denied to URL");
        }
    }

    // 2. 프리미엄 기능 접근 제어
    public boolean canAccessPremiumFeature(String userId, 
                                         PremiumFeature feature) {
        UserSubscription subscription = 
            subscriptionRepository.findByUserId(userId);
            
        return subscription != null && 
               subscription.hasFeature(feature) &&
               subscription.isActive();
    }
}
```

이러한 보안 구현을 통해:

1. 악의적인 URL 차단
2. 사용자 인증 및 권한 관리
3. 요청 검증 및 필터링
4. 보안 감사 및 모니터링
5. 데이터 암호화
6. 세분화된 접근 제어

를 제공할 수 있습니다. 또한, 정기적인 보안 감사와 모니터링을 통해 새로운 위협에 대응할 수 있습니다.

면접관: 실제 운영 환경에서 발생할 수 있는 장애 상황에 대한 대응 전략은 어떻게 되나요?

## 8. 장애 대응 전략

### 8.1 Circuit Breaker 패턴 구현
```java
@Service
public class ResiliencyService {
    
    private final CircuitBreakerFactory circuitBreakerFactory;

    // 1. 서비스별 Circuit Breaker 구성
    @Bean
    public Map<String, CircuitBreaker> circuitBreakers() {
        return Map.of(
            "urlRedirect", circuitBreakerFactory.create("urlRedirect",
                CircuitBreakerConfig.custom()
                    .failureRateThreshold(50)
                    .waitDurationInOpenState(Duration.ofSeconds(10))
                    .slidingWindowSize(10)
                    .build()),
                    
            "analytics", circuitBreakerFactory.create("analytics",
                CircuitBreakerConfig.custom()
                    .failureRateThreshold(30)
                    .waitDurationInOpenState(Duration.ofSeconds(5))
                    .build())
        );
    }

    // 2. Fallback 처리
    public String handleUrlRedirectFailure(String shortUrl, Exception e) {
        // Fallback 로직 (예: 캐시된 URL 반환 또는 임시 페이지로 리디렉션)
        return cacheService.getUrlFromBackupCache(shortUrl)
            .orElse("https://error-page.com");
    }
}
```

### 8.2 서비스 디그레이드
```java
@Service
public class DegradationService {
    
    private final HealthIndicator healthIndicator;

    // 1. 단계별 서비스 디그레이드
    public void degradeService(ServiceHealth health) {
        switch(health.getLevel()) {
            case CRITICAL:
                // 기본 URL 리디렉션만 유지
                disableAnalytics();
                disableUrlCreation();
                break;
                
            case SEVERE:
                // 필수 기능만 유지
                limitRateThreshold(50);
                disablePremiumFeatures();
                break;
                
            case WARNING:
                // 일부 기능 제한
                limitRateThreshold(80);
                break;
        }
    }

    // 2. 우선순위 기반 요청 처리
    public boolean shouldProcessRequest(Request request) {
        if (isSystemOverloaded()) {
            return request.getPriority() == Priority.HIGH;
        }
        return true;
    }
}
```

### 8.3 데이터 복구 전략
```java
@Service
public class DataRecoveryService {
    
    private final DatabaseBackupService backupService;
    private final RedisTemplate<String, String> redisTemplate;

    // 1. 데이터 백업
    @Scheduled(cron = "0 0 * * * *") // 매시간
    public void backupData() {
        // URL 매핑 데이터 백업
        backupService.backupUrlMappings();
        
        // 분석 데이터 백업
        backupService.backupAnalytics();
        
        // 캐시 스냅샷
        backupService.snapshotCache();
    }

    // 2. 데이터 복구 프로세스
    public void recoverData(RecoveryPoint point) {
        // 트랜잭션 로그 적용
        List<Transaction> transactions = 
            transactionLogger.getTransactionsAfter(point);
            
        // 데이터 복구
        transactions.forEach(tx -> {
            try {
                applyTransaction(tx);
            } catch (Exception e) {
                logRecoveryFailure(tx, e);
                notifyAdmins(tx, e);
            }
        });
    }
}
```

### 8.4 성능 모니터링 및 알림
```java
@Service
@Slf4j
public class MonitoringService {
    
    private final MeterRegistry registry;
    private final AlertService alertService;

    // 1. 핵심 메트릭 모니터링
    @Scheduled(fixedRate = 5000) // 5초마다
    public void monitorSystemHealth() {
        // 시스템 메트릭 수집
        SystemMetrics metrics = SystemMetrics.builder()
            .redirectLatency(measureRedirectLatency())
            .errorRate(calculateErrorRate())
            .cacheHitRate(getCacheHitRate())
            .systemLoad(getSystemLoad())
            .build();

        // 임계값 확인
        if (metrics.exceedsThresholds()) {
            handleAbnormalMetrics(metrics);
        }
    }

    // 2. 장애 감지 및 알림
    private void handleAbnormalMetrics(SystemMetrics metrics) {
        Alert alert = Alert.builder()
            .severity(determineSeverity(metrics))
            .message(createAlertMessage(metrics))
            .timestamp(Instant.now())
            .build();

        // 알림 발송
        alertService.sendAlert(alert);
        
        // 자동 조치 수행
        if (metrics.requiresAutomaticAction()) {
            performAutomaticRecovery(metrics);
        }
    }
}
```

### 8.5 분산 시스템 장애 대응
```java
@Service
public class DistributedSystemRecoveryService {
    
    private final NodeManager nodeManager;
    private final LoadBalancer loadBalancer;

    // 1. 노드 장애 처리
    public void handleNodeFailure(Node failedNode) {
        // 실패한 노드 제거
        nodeManager.removeNode(failedNode);
        
        // 트래픽 재분배
        loadBalancer.rebalance();
        
        // 데이터 재복제
        replicateData(failedNode.getData());
        
        // 새 노드 프로비저닝
        provisionNewNode(failedNode.getRole());
    }

    // 2. 네트워크 파티션 처리
    public void handleNetworkPartition(NetworkPartition partition) {
        // Quorum 기반 결정
        if (partition.hasQuorum()) {
            // 다수 파티션 선택
            promoteNewMaster(partition.getMajorityPartition());
            
            // 소수 파티션 격리
            isolateMinorityPartition(partition.getMinorityPartition());
        } else {
            // 수동 개입 필요
            enterReadOnlyMode();
            notifyOperators(partition);
        }
    }
}
```

이러한 장애 대응 전략을 통해:

1. 서비스 가용성 유지
2. 데이터 무결성 보장
3. 성능 저하 최소화
4. 신속한 장애 복구
5. 운영 투명성 확보

를 달성할 수 있습니다.

특히 중요한 점은:
- 단계별 디그레이드를 통한 핵심 기능 유지
- 자동화된 모니터링 및 알림
- 명확한 복구 프로세스
- 분산 시스템 특유의 장애 상황 대응

입니다.
