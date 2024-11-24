# 캐싱 전략 설계 면접

면접관: "대규모 이커머스 시스템에서 효율적인 캐싱 전략을 설계해주세요. 특히 상품 정보와 같이 자주 조회되는 데이터의 처리 방법에 대해 설명해주세요."

지원자: 네, 먼저 몇 가지 질문을 드려도 될까요?

면접관: 네, 말씀해주세요.

지원자: 다음 사항들을 확인하고 싶습니다:
1. 캐시해야 할 주요 데이터의 크기와 유형은 어떻게 되나요?
2. 데이터 갱신 빈도는 어느 정도인가요?
3. 캐시 히트율(Cache Hit Ratio)에 대한 목표가 있나요?
4. 데이터 일관성에 대한 요구사항은 어떻게 되나요?

면접관:
1. 상품 정보는 각각 약 1KB 크기이며, 총 100만 개의 상품이 있습니다. 상품 상세, 가격, 재고가 주요 정보입니다.
2. 상품 가격과 재고는 실시간성이 중요하며, 상품 상세 정보는 하루 평균 10% 정도가 갱신됩니다.
3. 캐시 히트율 95% 이상을 목표로 합니다.
4. 가격과 재고는 강한 일관성이, 상품 상세는 최대 5분의 지연이 허용됩니다.

지원자: 이해했습니다. 그러면 계층별 캐싱 전략을 설계해보겠습니다.

## 1. 멀티 레이어 캐싱 아키텍처

```java
@Service
public class CacheLayerService {
    
    private final LoadingCache<String, String> localCache;  // L1 Cache
    private final RedisTemplate<String, String> redisCache; // L2 Cache
    
    public CacheLayerService() {
        this.localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .build(key -> getFromRedis(key));
    }
    
    public Optional<Product> getProduct(String productId) {
        try {
            // 1. Local Cache 조회
            return Optional.ofNullable(localCache.get(productId));
        } catch (CacheLoadingException e) {
            // 2. Redis Cache 조회
            return Optional.ofNullable(redisCache.opsForValue().get(productId));
        }
    }
}
```

면접관: 가격과 재고 같은 실시간성이 중요한 데이터는 어떻게 처리하실 건가요?

지원자: Write-Through 전략과 캐시 무효화를 조합하여 구현하겠습니다.

```java
@Service
@Slf4j
public class ProductPriceService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final PriceRepository priceRepository;
    
    @Transactional
    public void updatePrice(String productId, BigDecimal newPrice) {
        try {
            // 1. DB 업데이트
            priceRepository.updatePrice(productId, newPrice);
            
            // 2. 캐시 업데이트 (Write-Through)
            String cacheKey = "price:" + productId;
            redisTemplate.opsForValue().set(cacheKey, newPrice.toString());
            
            // 3. 이벤트 발행 (다른 서비스에 통지)
            eventPublisher.publishPriceChange(productId, newPrice);
            
        } catch (Exception e) {
            // 4. 실패 시 캐시 무효화
            redisTemplate.delete("price:" + productId);
            throw e;
        }
    }
}
```

면접관: 캐시 일관성은 어떻게 보장하실 건가요?

### 2. 캐시 일관성 보장 전략

1. 분산 캐시 Lock 구현
```java
@Service
public class DistributedCacheLockService {
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean acquireLock(String key, long timeoutMs) {
        String lockKey = "lock:" + key;
        String lockValue = UUID.randomUUID().toString();
        
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, 
                Duration.ofMillis(timeoutMs));
                
        return Boolean.TRUE.equals(acquired);
    }
    
    @Async
    public void updateWithLock(String productId, Runnable updateAction) {
        String lockKey = "lock:" + productId;
        try {
            if (acquireLock(lockKey, 5000)) {  // 5초 타임아웃
                updateAction.run();
            } else {
                throw new CacheLockException("Lock acquisition failed");
            }
        } finally {
            releaseLock(lockKey);
        }
    }
}
```

2. TTL(Time To Live) 기반 캐시 갱신
```java
@Component
public class CacheRefreshStrategy {
    
    @Scheduled(fixedRate = 300000) // 5분마다 실행
    public void refreshExpiredCache() {
        // 1. TTL이 임박한 캐시 조회
        Set<String> expiringKeys = redisTemplate
            .keys("product:*")
            .stream()
            .filter(this::isExpiringSoon)
            .collect(Collectors.toSet());
            
        // 2. 백그라운드에서 갱신
        expiringKeys.forEach(key -> 
            CompletableFuture.runAsync(() -> refreshCache(key)));
    }
    
    private boolean isExpiringSoon(String key) {
        Long ttl = redisTemplate.getExpire(key);
        return ttl != null && ttl < 300; // 5분 미만 남은 캐시
    }
}
```

3. 이벤트 기반 캐시 동기화
```java
@Service
public class CacheEventHandler {
    
    @KafkaListener(topics = "cache-invalidation")
    public void handleCacheInvalidation(CacheInvalidationEvent event) {
        switch(event.getType()) {
            case PRODUCT_UPDATE:
                invalidateProductCache(event.getProductId());
                break;
            case PRICE_UPDATE:
                invalidatePriceCache(event.getProductId());
                break;
            case STOCK_UPDATE:
                invalidateStockCache(event.getProductId());
                break;
        }
    }
    
    private void invalidateProductCache(String productId) {
        // 로컬 캐시 무효화
        localCache.invalidate(productId);
        
        // Redis 캐시 무효화
        redisTemplate.delete("product:" + productId);
        
        // 관련 집계 캐시 무효화
        redisTemplate.delete("category:" + 
            productRepository.getCategoryId(productId));
    }
}
```

4. 캐시 정합성 검증
```java
@Service
public class CacheValidationService {
    
    @Scheduled(cron = "0 */10 * * * *") // 10분마다 실행
    public void validateCacheConsistency() {
        // 1. 샘플링된 캐시 키 선택
        List<String> sampleKeys = getSampleCacheKeys();
        
        // 2. DB와 캐시 데이터 비교
        Map<String, InconsistencyReport> inconsistencies = 
            checkConsistency(sampleKeys);
        
        // 3. 불일치 처리
        if (!inconsistencies.isEmpty()) {
            handleInconsistencies(inconsistencies);
        }
    }
    
    private Map<String, InconsistencyReport> checkConsistency(List<String> keys) {
        return keys.stream()
            .map(this::compareWithDatabase)
            .filter(InconsistencyReport::hasDiscrepancy)
            .collect(Collectors.toMap(
                InconsistencyReport::getKey,
                Function.identity()
            ));
    }
    
    private void handleInconsistencies(Map<String, InconsistencyReport> reports) {
        reports.forEach((key, report) -> {
            // 로그 기록
            log.warn("Cache inconsistency detected for key: {}", key);
            
            // 메트릭 기록
            meterRegistry.counter("cache.inconsistency").increment();
            
            // 캐시 재구성
            rebuildCache(key);
        });
    }
}
```

면접관: Hot Key 문제는 어떻게 해결하시겠습니까?

지원자: Hot Key 문제에 대해서는 다음과 같은 전략을 사용하겠습니다:

```java
@Service
public class HotKeyHandlingService {
    
    private final LoadingCache<String, String> localCache;
    private final RedisTemplate<String, String> redisTemplate;
    
    // Hot Key 샤딩 처리
    public String getWithSharding(String key, int shardCount) {
        int shard = Math.abs(key.hashCode() % shardCount);
        String shardedKey = key + ":shard:" + shard;
        
        // 로컬 캐시 확인
        String value = localCache.getIfPresent(key);
        if (value != null) {
            return value;
        }
        
        // Redis 캐시 확인
        return redisTemplate.opsForValue().get(shardedKey);
    }
    
    // Hot Key 모니터링
    @Scheduled(fixedRate = 1000)
    public void monitorHotKeys() {
        RedisCallback<Set<String>> callback = connection -> {
            // Redis INFO 명령어로 Hot Key 감지
            return connection.info("commandstats");
        };
        
        Set<String> hotKeys = redisTemplate.execute(callback);
        
        // Hot Key 발견 시 샤딩 처리
        hotKeys.forEach(this::handleHotKey);
    }
}
```

이러한 전략들을 통해 캐시의 일관성을 보장하면서도 성능을 최적화할 수 있습니다.