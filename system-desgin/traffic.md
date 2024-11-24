# 대용량 트래픽 처리 시스템 설계 면접

면접관: "쇼핑몰 시스템을 설계해주세요. 기본적으로 100만 MAU(Monthly Active Users)를 처리해야 하며, 특히 대규모 프로모션 시에는 평소보다 10배 이상의 트래픽이 발생할 수 있습니다."

지원자: 요구사항을 명확히 하기 위해 몇 가지 질문을 드려도 될까요?

면접관: 네, 물론입니다.

지원자: 다음 사항들을 확인하고 싶습니다:
1. 전체 사용자 수와 동시접속자 수는 어느 정도로 예상하나요?
2. 데이터 정합성이나 일관성에 대한 요구사항은 어떻게 되나요?
3. 응답 시간에 대한 SLA(Service Level Agreement)는 어떻게 되나요?
4. 프로모션 시 주로 어떤 패턴의 트래픽이 발생하나요?

면접관: 좋은 질문들입니다.
1. MAU 100만 기준으로 일일 활성 사용자(DAU)는 약 30만 명이며, 피크 시간대 동시접속자는 평상시 5만 명, 프로모션 시 최대 50만 명까지 예상됩니다.
2. 주문 및 결제 관련해서는 강한 일관성이 필요하며, 상품 조회 등은 최종 일관성(eventual consistency)으로 충분합니다.
3. 응답시간은 99%의 요청에 대해 3초 이내, 평균 응답시간 1초 이내가 목표입니다.
4. 프로모션 시작 시점에 주문 트래픽이 급격히 증가하며, 특정 상품에 대한 집중적인 조회와 주문이 발생합니다.

지원자: 네, 이해했습니다. 그러면 전체적인 아키텍처를 설계해보겠습니다.

## 1. 전체 아키텍처 구성

```plaintext
[클라이언트] → [CDN] → [Load Balancer] → [API Gateway]
                                           ↓
[Cache Layer]  ←  [Application Servers] → [Message Queue]
     ↓                     ↓                    ↓
[Redis Cluster]   [Database Cluster]    [Background Workers]
```

이러한 구성에 대해 각 계층별로 상세 설명을 드려도 될까요?

면접관: 네, 각 계층별로 어떻게 설계하실 건지 설명해주세요.

## 2. 각 계층별 상세 설계

### 2.1 프론트엔드 계층
```java
// CDN 설정 예시
@Configuration
public class CdnConfig {
    @Value("${cdn.host}")
    private String cdnHost;
    
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addResourceHandlers(ResourceHandlerRegistry registry) {
                registry.addResourceHandler("/static/**")
                    .addResourceLocations("classpath:/static/")
                    .setCachePeriod(3600)
                    .resourceChain(true)
                    .addResolver(new VersionResourceResolver()
                        .addContentVersionStrategy("/**"));
            }
        };
    }
}
```

### 2.2 부하 분산 계층
```yaml
# Kubernetes Ingress 설정 예시
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rate-limit-connections: "100"
    nginx.ingress.kubernetes.io/limit-rps: "100"
spec:
  rules:
  - host: "shop.example.com"
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

면접관: 부하 분산 시 세션 관리는 어떻게 하실 건가요?

지원자: 세션 관리는 Redis를 사용한 중앙 집중식 세션 관리를 구현하겠습니다.

```java
@Configuration
@EnableRedisHttpSession
public class SessionConfig {
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration("redis-host", 6379));
    }
}
```

면접관: 프로모션 상황에서의 급격한 트래픽 증가는 어떻게 처리하실 건가요?

지원자: 다음과 같은 전략을 사용하겠습니다:

### 2.3 트래픽 처리 전략
```java
@Service
public class PromotionService {
    private final RedisTemplate<String, String> redisTemplate;
    private final RateLimiter rateLimiter;
    
    // 동시 접속 제어
    public boolean tryAccess(String userId) {
        String key = "promotion:access:" + userId;
        return redisTemplate.opsForValue()
            .setIfAbsent(key, "1", Duration.ofSeconds(5));
    }
    
    // 주문 큐잉
    @Async
    public CompletableFuture<OrderResult> processOrder(OrderRequest request) {
        if (!rateLimiter.tryAcquire()) {
            throw new TooManyRequestsException();
        }
        return orderQueue.enqueueOrder(request);
    }
}
```

면접관: 재고 관리는 어떻게 하실 건가요?

지원자: 재고는 Redis와 DB를 조합하여 관리하겠습니다:

### 2.4 재고 관리 시스템
```java
@Service
public class InventoryService {
    private final RedisTemplate<String, String> redisTemplate;
    private final InventoryRepository inventoryRepository;
    
    @Transactional
    public boolean checkAndDecrementStock(String productId, int quantity) {
        String key = "stock:" + productId;
        Long remaining = redisTemplate.opsForValue()
            .decrement(key, quantity);
            
        if (remaining < 0) {
            redisTemplate.opsForValue()
                .increment(key, quantity);
            return false;
        }
        
        // DB 동기화는 별도 배치로 처리
        return true;
    }
    
    @Scheduled(fixedRate = 1000)
    public void syncStockWithDb() {
        // Redis의 재고 정보를 DB에 동기화
    }
}
```

면접관: 시스템 장애 상황은 어떻게 대처하실 건가요?

지원자: Circuit Breaker 패턴과 Fallback 처리를 구현하겠습니다:

### 2.5 장애 대응 시스템
```java
@Service
public class ResilientService {
    private final CircuitBreaker circuitBreaker;
    
    public ProductResponse getProduct(String productId) {
        return circuitBreaker.run(
            () -> primaryProductService.getProduct(productId),
            throwable -> getProductFallback(productId)
        );
    }
    
    private ProductResponse getProductFallback(String productId) {
        // 캐시된 데이터 반환 또는 기본 응답 처리
        return cachedProductService.getProduct(productId);
    }
}
```

면접관: 모니터링과 알림은 어떻게 구성하실 건가요?

### 2.7 장애 감지 및 대응 전략

1. 헬스체크 시스템 구현
```java
@RestController
public class HealthCheckController {
    private final List<HealthIndicator> healthIndicators;
    
    @GetMapping("/health")
    public ResponseEntity<HealthStatus> checkHealth() {
        HealthStatus status = new HealthStatus();
        
        // 각 컴포넌트 상태 체크
        status.addComponent("db", checkDatabase());
        status.addComponent("cache", checkRedis());
        status.addComponent("queue", checkMessageQueue());
        
        return ResponseEntity.ok(status);
    }
    
    @Scheduled(fixedRate = 30000) // 30초마다 실행
    public void deepHealthCheck() {
        // 상세 헬스체크 로직
        checkDatabaseConnections();
        checkRedisClusterStatus();
        checkKafkaTopics();
    }
}
```

2. 자동 복구(Self-Healing) 시스템
```java
@Service
@Slf4j
public class AutoRecoveryService {
    
    @Retryable(
        value = {ServiceException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000))
    public void recoverService(String serviceId) {
        try {
            // 1. 서비스 재시작 시도
            restartService(serviceId);
            
            // 2. 리소스 정리
            cleanupResources(serviceId);
            
            // 3. 상태 확인
            verifyServiceHealth(serviceId);
            
        } catch (Exception e) {
            log.error("Recovery failed for service: " + serviceId, e);
            notifyOperations(serviceId, e);
            throw new ServiceException("Recovery failed", e);
        }
    }
}
```

면접관: Circuit Breaker 패턴을 좀 더 구체적으로 설명해주시겠어요?

3. Circuit Breaker 구현
```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        return CircuitBreakerRegistry.of(
            io.github.resilience4j.circuitbreaker.CircuitBreakerConfig.custom()
                .failureRateThreshold(50)        // 50% 실패율
                .waitDurationInOpenState(Duration.ofMillis(1000))
                .slidingWindowSize(10)           // 10번의 호출 기준
                .build());
    }
}

@Service
public class ResilientOrderService {
    private final CircuitBreaker circuitBreaker;
    private final OrderRepository orderRepository;
    
    public OrderResult processOrder(OrderRequest request) {
        return circuitBreaker.executeSupplier(() -> {
            try {
                return orderRepository.createOrder(request);
            } catch (Exception e) {
                // fallback 처리
                return handleOrderFailure(request);
            }
        });
    }
    
    private OrderResult handleOrderFailure(OrderRequest request) {
        // 1. 임시 저장
        saveToFailoverQueue(request);
        
        // 2. 사용자에게 알림
        notifyUserAboutDelay(request.getUserId());
        
        // 3. 관리자 알림
        alertOperations("Order processing failed", request);
        
        return OrderResult.delayed();
    }
}
```

4. 장애 전파 방지(Bulkhead Pattern) 구현
```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean
    public ThreadPoolTaskExecutor orderProcessingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(50);
        executor.setRejectedExecutionHandler(new CallerRunsPolicy());
        return executor;
    }
    
    @Bean
    public ThreadPoolTaskExecutor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        return executor;
    }
}

@Service
public class IsolatedOrderService {
    private final ThreadPoolTaskExecutor orderExecutor;
    private final ThreadPoolTaskExecutor notificationExecutor;
    
    public CompletableFuture<OrderResult> processOrderAsync(OrderRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // 주문 처리 로직
            OrderResult result = processOrder(request);
            
            // 비동기 알림 처리
            notificationExecutor.execute(() -> 
                sendNotification(result));
                
            return result;
        }, orderExecutor);
    }
}
```

면접관: 실제 장애가 발생했을 때의 복구 절차(Disaster Recovery)는 어떻게 계획하고 계신가요?

### 2.8 Disaster Recovery 계획

1. 데이터 백업 및 복구 전략
```java
@Configuration
public class BackupConfig {
    
    // 백업 종류별 구성
    @Bean
    public BackupStrategy multiLevelBackup() {
        return BackupStrategy.builder()
            .addLevel(
                // 1. 실시간 복제
                BackupLevel.REAL_TIME,
                new DatabaseReplicationConfig(
                    masterDb,
                    standbyDb,
                    ReplicationMode.SYNC
                )
            )
            .addLevel(
                // 2. 시점 백업
                BackupLevel.POINT_IN_TIME,
                new SnapshotConfig(
                    Duration.ofHours(1)  // 1시간 간격
                )
            )
            .addLevel(
                // 3. 전체 백업
                BackupLevel.FULL,
                new FullBackupConfig(
                    Duration.ofDays(1)   // 일간 백업
                )
            )
            .build();
    }
}
```

2. DR(Disaster Recovery) 사이트 구성
```yaml
# Kubernetes Multi-Region 구성
apiVersion: v1
kind: Service
metadata:
  name: shop-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  selector:
    app: shop
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfied: DoNotSchedule
          labelSelector:
            matchLabels:
              app: shop
```

3. 복구 절차 자동화
```java
@Service
@Slf4j
public class DisasterRecoveryService {
    
    @Autowired
    private KubernetesClient kubernetesClient;
    
    @Autowired
    private DatabaseFailoverService dbFailover;
    
    public void initiateFailover() {
        try {
            // 1. 트래픽 전환
            switchTraffic();
            
            // 2. DB 페일오버
            dbFailover.failoverToStandby();
            
            // 3. 캐시 워밍업
            warmupCache();
            
        } catch (Exception e) {
            log.error("Failover failed", e);
            rollbackFailover();
            throw new FailoverException("DR 전환 실패", e);
        }
    }
    
    private void switchTraffic() {
        // DNS 변경 또는 로드밸런서 설정 변경
        updateDNSRecords();
        // 또는
        updateLoadBalancerConfig();
    }
    
    private void warmupCache() {
        // 주요 데이터 캐시 프리로딩
        preloadFrequentlyAccessedData();
        preloadCriticalConfigurations();
    }
}
```

4. 복구 시간 목표(RTO) 및 복구 시점 목표(RPO) 관리
```java
@Component
public class RecoveryMetricsCollector {
    
    private final MeterRegistry registry;
    
    public void recordRecoveryMetrics(RecoveryEvent event) {
        // RTO 측정
        registry.timer("disaster.recovery.time")
            .record(event.getRecoveryDuration());
            
        // RPO 측정
        registry.gauge("disaster.recovery.data.loss",
            event.getDataLossInSeconds());
            
        // 복구 성공률
        registry.counter("disaster.recovery.success.rate")
            .increment(event.isSuccessful() ? 1 : 0);
    }
    
    @Scheduled(fixedRate = 3600000) // 1시간마다
    public void performRecoveryTest() {
        // 자동화된 복구 테스트 수행
        try {
            DisasterRecoveryTest test = new DisasterRecoveryTest();
            test.simulateFailure();
            test.verifyRecovery();
            recordRecoveryMetrics(test.getResults());
        } catch (Exception e) {
            notifyOperations("Recovery test failed", e);
        }
    }
}
```

5. 데이터 정합성 검증
```java
@Service
public class DataIntegrityService {
    
    @Scheduled(cron = "0 0 * * * *") // 매시간
    public void verifyDataIntegrity() {
        // 1. 마스터-슬레이브 데이터 비교
        compareDataBetweenRegions();
        
        // 2. 체크섬 검증
        validateChecksums();
        
        // 3. 트랜잭션 로그 검증
        verifyTransactionLogs();
    }
    
    private void compareDataBetweenRegions() {
        // 주요 테이블 데이터 비교
        Set<String> differences = findDataDifferences();
        if (!differences.isEmpty()) {
            // 불일치 데이터 복구
            reconcileData(differences);
            // 운영팀 통보
            notifyOperations("Data inconsistency detected", differences);
        }
    }
}
```