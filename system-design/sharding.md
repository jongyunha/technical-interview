# 데이터 샤딩(Sharding) 설계 면접

면접관: "대규모 이커머스 플랫폼에서 데이터베이스 샤딩 전략을 설계해주세요. 특히 주문 데이터와 상품 데이터에 대한 샤딩 방안을 설명해주세요."

지원자: 네, 먼저 몇 가지 질문을 드려도 될까요?

면접관: 네, 말씀해주세요.

지원자: 다음 사항들을 확인하고 싶습니다:
1. 현재 데이터의 규모와 증가율은 어떻게 되나요?
2. 주요 쿼리 패턴은 어떻게 되나요? (예: 조회, 집계 등)
3. 데이터 일관성에 대한 요구사항은 어떻게 되나요?
4. 장애 상황에서의 가용성 요구사항은 어떻게 되나요?

면접관:
1. 주문 데이터는 월 1000만 건이 발생하며, 연간 50% 증가 추세입니다. 상품 데이터는 100만 개 정도입니다.
2. 주문은 사용자별 조회가 많고, 상품은 카테고리별 조회와 재고 수정이 빈번합니다.
3. 주문 데이터는 강한 일관성이 필요하고, 상품 정보는 약간의 지연이 허용됩니다.
4. 일부 샤드 장애 시에도 서비스는 계속 운영되어야 합니다.

지원자: 이해했습니다. 샤딩 전략을 설계해보겠습니다.

## 1. 샤딩 키 설계

```java
public class ShardingStrategy {
    
    // 주문 데이터 샤딩 - userId 기반
    public int calculateOrderShard(String userId) {
        return Math.abs(userId.hashCode() % TOTAL_ORDER_SHARDS);
    }
    
    // 상품 데이터 샤딩 - 카테고리 기반
    public int calculateProductShard(String categoryId) {
        return Math.abs(categoryId.hashCode() % TOTAL_PRODUCT_SHARDS);
    }
    
    // 샤드 라우팅 정보
    public ShardInfo getShardInfo(ShardingKey key) {
        return ShardInfo.builder()
            .shardId(calculateShard(key))
            .replicaNodes(getReplicaNodes(key))
            .build();
    }
}
```

## 2. 데이터베이스 라우팅 구현

```java
@Component
public class ShardingDataSource extends AbstractRoutingDataSource {
    
    private final ShardingStrategy shardingStrategy;
    
    @Override
    protected Object determineCurrentLookupKey() {
        ShardingKey key = ShardingContextHolder.getShardingKey();
        return shardingStrategy.calculateShard(key);
    }
    
    // 트랜잭션 매니저 설정
    @Bean
    public PlatformTransactionManager transactionManager() {
        return new ChainedTransactionManager(
            getOrderTransactionManager(),
            getProductTransactionManager()
        );
    }
}
```

면접관: Cross-Shard 쿼리는 어떻게 처리하실 건가요?

## 3. Cross-Shard 쿼리 처리

### 3.1 분산 쿼리 실행기
```java
@Service
public class DistributedQueryExecutor {
    
    private final ExecutorService queryExecutor = 
        Executors.newFixedThreadPool(10);
    
    // 병렬 쿼리 실행
    public <T> List<T> executeAcrossShards(ShardedQuery<T> query) {
        List<CompletableFuture<List<T>>> futures = new ArrayList<>();
        
        // 각 샤드별 쿼리 실행
        for (int shardId = 0; shardId < TOTAL_SHARDS; shardId++) {
            final int shard = shardId;
            CompletableFuture<List<T>> future = CompletableFuture
                .supplyAsync(() -> query.execute(shard), queryExecutor);
            futures.add(future);
        }
        
        // 결과 취합
        return futures.stream()
            .map(CompletableFuture::join)
            .flatMap(List::stream)
            .collect(Collectors.toList());
    }
    
    // 조건부 샤드 쿼리
    public <T> List<T> executeWithShardPredicate(
            ShardedQuery<T> query, 
            ShardPredicate predicate) {
        return IntStream.range(0, TOTAL_SHARDS)
            .filter(predicate::test)
            .mapToObj(shard -> query.execute(shard))
            .flatMap(List::stream)
            .collect(Collectors.toList());
    }
}
```

### 3.2 결과 집계 처리
```java
@Service
public class QueryAggregator {
    
    // 페이징 처리를 위한 집계
    public <T> PagedResult<T> aggregateWithPaging(
            List<ShardResult<T>> results, 
            int page, 
            int size) {
        
        // 전체 결과 정렬
        List<T> sortedResults = results.stream()
            .flatMap(ShardResult::stream)
            .sorted()
            .skip((long) page * size)
            .limit(size)
            .collect(Collectors.toList());
            
        // 전체 카운트 계산
        long totalCount = results.stream()
            .mapToLong(ShardResult::getCount)
            .sum();
            
        return new PagedResult<>(sortedResults, page, size, totalCount);
    }
    
    // 집계 함수 처리
    public <T> AggregateResult aggregateResults(
            List<ShardResult<T>> results, 
            AggregateFunction function) {
        
        return switch (function) {
            case COUNT -> results.stream()
                .mapToLong(ShardResult::getCount)
                .sum();
                
            case SUM -> results.stream()
                .mapToDouble(ShardResult::getSum)
                .sum();
                
            case AVG -> {
                double sum = results.stream()
                    .mapToDouble(ShardResult::getSum)
                    .sum();
                long count = results.stream()
                    .mapToLong(ShardResult::getCount)
                    .sum();
                yield sum / count;
            }
        };
    }
}
```

### 3.3 분산 트랜잭션 처리
```java
@Service
public class DistributedTransactionManager {
    
    private final TransactionTemplate transactionTemplate;
    private final ShardingStrategy shardingStrategy;
    
    // 2PC (Two-Phase Commit) 구현
    public void executeInDistributedTransaction(
            List<ShardedOperation> operations) {
            
        // 1. Prepare Phase
        Map<Integer, Boolean> prepareResults = operations.stream()
            .map(op -> prepareTransaction(op))
            .collect(Collectors.toMap(
                PrepareResult::getShardId,
                PrepareResult::isSuccess
            ));
            
        // 2. Commit/Rollback Phase
        if (prepareResults.values().stream().allMatch(result -> result)) {
            // 모든 샤드가 준비되면 커밋
            operations.forEach(this::commitTransaction);
        } else {
            // 하나라도 실패하면 롤백
            operations.forEach(this::rollbackTransaction);
            throw new DistributedTransactionException("Transaction failed");
        }
    }
    
    // 보상 트랜잭션 처리
    public void executeWithCompensation(
            ShardedOperation operation, 
            CompensatingOperation compensation) {
            
        try {
            operation.execute();
        } catch (Exception e) {
            // 실패 시 보상 트랜잭션 실행
            compensation.execute();
            throw e;
        }
    }
}
```

### 3.4 샤드 리밸런싱
```java
@Service
public class ShardRebalancer {
    
    private final ShardingStrategy shardingStrategy;
    private final DataMigrator dataMigrator;
    
    @Scheduled(cron = "0 0 2 * * *")  // 매일 새벽 2시
    public void checkAndRebalance() {
        Map<Integer, ShardMetrics> metrics = collectShardMetrics();
        
        if (needsRebalancing(metrics)) {
            executeRebalancing(metrics);
        }
    }
    
    private void executeRebalancing(Map<Integer, ShardMetrics> metrics) {
        // 1. 새로운 샤드 맵 계산
        Map<Integer, Integer> rebalanceMap = calculateRebalanceMap(metrics);
        
        // 2. 점진적 데이터 마이그레이션
        rebalanceMap.forEach((sourceShardId, targetShardId) -> {
            dataMigrator.migrateData(
                sourceShardId, 
                targetShardId,
                batchSize = 1000
            );
        });
        
        // 3. 라우팅 테이블 업데이트
        updateRoutingTable(rebalanceMap);
    }
}
```

면접관: 데이터 정합성은 어떻게 보장하실 건가요? 특히 리밸런싱 중에 발생할 수 있는 문제는요?

### 3.5 데이터 정합성 보장 전략

```java
@Service
@Slf4j
public class DataConsistencyManager {

    private final RedisTemplate<String, String> redisTemplate;
    private final ShardingStrategy shardingStrategy;

    // 1. 이중 쓰기(Dual Write) 처리
    @Transactional
    public void executeWithDualWrite(WriteOperation operation) {
        String operationId = UUID.randomUUID().toString();
        try {
            // 쓰기 작업 로그 기록
            logOperation(operationId, operation);
            
            // 신규 샤드에 쓰기
            int newShardId = shardingStrategy.getNewShardId(operation.getKey());
            operation.executeOn(newShardId);
            
            // 구 샤드에 쓰기
            int oldShardId = shardingStrategy.getOldShardId(operation.getKey());
            operation.executeOn(oldShardId);
            
            // 작업 완료 마킹
            markOperationComplete(operationId);
        } catch (Exception e) {
            // 실패 시 복구 프로세스 시작
            initiateRecoveryProcess(operationId);
            throw e;
        }
    }

    // 2. 정합성 검증
    @Scheduled(fixedRate = 5000) // 5초마다 실행
    public void verifyConsistency() {
        Set<String> inconsistentKeys = new HashSet<>();
        
        getAllShardKeys().forEach(key -> {
            List<ShardData> shardDataList = getAllShardData(key);
            if (!isConsistent(shardDataList)) {
                inconsistentKeys.add(key);
                resolveInconsistency(key, shardDataList);
            }
        });
    }

    // 3. 리밸런싱 중 정합성 보장
    public class RebalancingConsistencyGuard {
        private final ConcurrentMap<String, LockInfo> lockMap = new ConcurrentHashMap<>();
        
        public void beginRebalancing(String key) {
            // 리밸런싱 시작 시 락 획득
            LockInfo lock = new LockInfo(
                System.currentTimeMillis(),
                RebalancingState.IN_PROGRESS
            );
            lockMap.put(key, lock);
        }
        
        public void processRequest(String key, Operation operation) {
            LockInfo lock = lockMap.get(key);
            if (lock != null) {
                // 리밸런싱 중인 경우
                if (lock.getState() == RebalancingState.IN_PROGRESS) {
                    // 양쪽 샤드 모두에 쓰기
                    executeOnBothShards(key, operation);
                } else {
                    // 새로운 샤드에만 쓰기
                    executeOnNewShard(key, operation);
                }
            }
        }
    }

    // 4. 복구 프로세스
    private void initiateRecoveryProcess(String operationId) {
        try {
            OperationLog log = getOperationLog(operationId);
            if (!isOperationComplete(operationId)) {
                // 미완료 작업 재실행
                retryOperation(log);
                
                // 불일치 데이터 수정
                reconcileData(log.getKey());
            }
        } catch (Exception e) {
            log.error("Recovery failed for operation: " + operationId, e);
            alertOperations("Recovery failed", operationId);
        }
    }

    // 5. 버전 관리
    private class VersionedData {
        private final long version;
        private final Object data;
        private final Timestamp timestamp;
        
        public boolean isNewer(VersionedData other) {
            if (this.version != other.version) {
                return this.version > other.version;
            }
            return this.timestamp.after(other.timestamp);
        }
    }

    // 6. 충돌 해결
    private void resolveConflict(String key, List<VersionedData> versions) {
        // 가장 최신 버전 찾기
        VersionedData latest = versions.stream()
            .max(Comparator.comparing(VersionedData::getVersion))
            .orElseThrow();
            
        // 다른 모든 샤드 업데이트
        versions.stream()
            .filter(v -> !v.equals(latest))
            .forEach(v -> updateShardData(key, latest));
            
        // 충돌 로그 기록
        logConflictResolution(key, versions, latest);
    }

    // 7. 모니터링 및 알림
    @Scheduled(fixedRate = 1000)
    public void monitorConsistency() {
        ConsistencyMetrics metrics = gatherConsistencyMetrics();
        
        if (metrics.getInconsistencyRate() > 0.01) { // 1% 초과
            alertOperations(
                "High inconsistency rate detected",
                metrics.toString()
            );
        }
        
        // 메트릭 기록
        recordMetrics(metrics);
    }
}
```

이러한 정합성 보장 메커니즘을 통해:
1. 리밸런싱 중 데이터 일관성 유지
2. 장애 상황에서의 복구 가능
3. 버전 충돌 해결
4. 지속적인 모니터링과 알림

을 구현할 수 있습니다.

이와 함께, 정합성 문제가 발생했을 때의 비즈니스 영향을 최소화하기 위한 보상 트랜잭션도 구현하는 것이 중요합니다.

면접관: 실제 운영 환경에서 이러한 샤딩 전략을 적용할 때 주의해야 할 점은 무엇인가요?

## 4. 운영 관점에서의 주의사항과 모범 사례

### 4.1 모니터링 시스템
```java
@Service
public class ShardingMonitorService {
    
    private final MeterRegistry meterRegistry;
    
    // 1. 성능 모니터링
    public void monitorShardPerformance() {
        shardDataSources.forEach((shardId, dataSource) -> {
            // 쿼리 지연시간 측정
            Timer.builder("shard.query.latency")
                .tag("shard", String.valueOf(shardId))
                .register(meterRegistry)
                .record(() -> measureQueryLatency(dataSource));
                
            // 샤드별 부하 측정
            Gauge.builder("shard.load", dataSource,
                this::measureShardLoad)
                .tag("shard", String.valueOf(shardId))
                .register(meterRegistry);
        });
    }
    
    // 2. 샤드 균형 모니터링
    @Scheduled(fixedRate = 300000) // 5분마다
    public void monitorShardBalance() {
        Map<Integer, Long> shardSizes = getShardSizes();
        double averageSize = calculateAverageSize(shardSizes);
        
        shardSizes.forEach((shardId, size) -> {
            double deviation = calculateDeviation(size, averageSize);
            if (deviation > 0.2) { // 20% 이상 차이
                alertOperations("Shard imbalance detected",
                    Map.of("shardId", shardId,
                          "deviation", deviation));
            }
        });
    }
}
```

### 4.2 점진적 마이그레이션
```java
@Service
public class GradualMigrationService {
    
    // 1. 점진적 데이터 이전
    public void migrateDataGradually(int sourceShardId, int targetShardId) {
        int batchSize = 1000;
        String cursor = "0";
        
        do {
            // 배치 단위로 데이터 이전
            ScanResult<String> scanResult = scanKeys(sourceShardId, cursor, batchSize);
            cursor = scanResult.getCursor();
            
            List<String> keys = scanResult.getResult();
            migrateKeys(keys, sourceShardId, targetShardId);
            
            // 진행상황 기록
            updateMigrationProgress(sourceShardId, targetShardId, cursor);
            
            // 부하 조절을 위한 일시 정지
            Thread.sleep(100);
            
        } while (!cursor.equals("0"));
    }
    
    // 2. 무중단 마이그레이션
    private void migrateKeys(List<String> keys, int sourceShardId, int targetShardId) {
        for (String key : keys) {
            // 이중 쓰기 시작
            enableDualWrite(key);
            
            // 데이터 복사
            copyData(key, sourceShardId, targetShardId);
            
            // 검증
            verifyDataConsistency(key, sourceShardId, targetShardId);
            
            // 트래픽 전환
            switchTraffic(key, targetShardId);
            
            // 이전 데이터 정리
            cleanupSourceData(key, sourceShardId);
        }
    }
}
```

### 4.3 장애 복구 전략
```java
@Service
public class ShardFailoverService {
    
    // 1. 샤드 장애 감지
    @Scheduled(fixedRate = 10000) // 10초마다
    public void detectShardFailures() {
        shardDataSources.forEach((shardId, dataSource) -> {
            if (!isShardHealthy(dataSource)) {
                initiateFailover(shardId);
            }
        });
    }
    
    // 2. 자동 페일오버
    private void initiateFailover(int failedShardId) {
        try {
            // 읽기 트래픽을 복제본으로 전환
            switchReadTrafficToReplica(failedShardId);
            
            // 새로운 마스터 승격
            promoteNewMaster(failedShardId);
            
            // 라우팅 테이블 업데이트
            updateRoutingTable(failedShardId);
            
        } catch (Exception e) {
            // 수동 개입 필요
            alertOperations("Failover failed", 
                Map.of("shardId", failedShardId));
        }
    }
}
```

### 4.4 운영 가이드라인
```java
public class OperationalGuidelines {
    
    /*
    1. 용량 계획
       - 각 샤드는 전체 데이터의 120% 용량 확보
       - 성장률을 고려한 6개월 선제적 용량 계획
    
    2. 백업 전략
       - 샤드별 일간 전체 백업
       - 시간별 증분 백업
       - 정기적인 복구 테스트
    
    3. 모니터링 임계값
       - CPU 사용률 70% 이상 시 경고
       - 디스크 사용률 80% 이상 시 경고
       - 복제 지연 10초 이상 시 경고
    
    4. 유지보수 절차
       - 샤드 추가는 항상 2의 제곱수로 진행
       - 메이저 변경은 트래픽 적은 새벽 시간대 진행
       - 변경 전후 성능 지표 비교 필수
    */
}
```

### 4.5 성능 최적화 가이드
```java
@Configuration
public class ShardingOptimizationConfig {
    
    @Bean
    public HikariDataSource shardDataSource() {
        HikariConfig config = new HikariConfig();
        
        // 1. 커넥션 풀 최적화
        config.setMaximumPoolSize(50);
        config.setMinimumIdle(20);
        config.setIdleTimeout(300000);
        
        // 2. 쿼리 타임아웃 설정
        config.setConnectionTimeout(3000);
        config.setValidationTimeout(2000);
        
        // 3. 캐시 사용 최적화
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        
        return new HikariDataSource(config);
    }
}
```

이러한 운영 관점의 고려사항들을 통해:
1. 안정적인 서비스 운영
2. 장애 상황에 대한 신속한 대응
3. 성능 최적화
4. 확장성 있는 아키텍처 유지

를 달성할 수 있습니다.

면접관: 샤딩의 단점이나 고려해야 할 트레이드오프는 무엇인가요?

## 5. 샤딩의 트레이드오프와 대응 전략

### 5.1 복잡성 증가
```java
@Service
public class ShardingComplexityHandler {
    
    // 1. 쿼리 복잡성 관리
    public class QueryRouter {
        public <T> List<T> executeQuery(ShardedQuery<T> query) {
            if (query.requiresCrossShardExecution()) {
                // 복잡한 크로스 샤드 쿼리 처리
                return executeAcrossShards(query);
            } else {
                // 단일 샤드 쿼리 처리
                int shardId = shardingStrategy.determineShardId(query.getShardKey());
                return executeSingleShard(shardId, query);
            }
        }
    }
    
    // 2. 트랜잭션 복잡성 관리
    public class TransactionManager {
        public void executeTransaction(List<ShardedOperation> operations) {
            if (operations.stream().map(ShardedOperation::getShardId).distinct().count() > 1) {
                // 분산 트랜잭션 필요
                distributedTxManager.execute(operations);
            } else {
                // 단일 샤드 트랜잭션
                singleShardTxManager.execute(operations.get(0));
            }
        }
    }
}
```

### 5.2 성능 트레이드오프
```java
@Service
public class PerformanceTradeoffManager {
    
    // 1. Join 연산 처리
    public class JoinStrategy {
        public <T> List<T> executeJoin(JoinQuery query) {
            if (canExecuteLocalJoin(query)) {
                // 동일 샤드 내 조인
                return executeLocalJoin(query);
            } else {
                // 브로드캐스트 조인 또는 분산 조인
                return executeBroadcastJoin(query);
            }
        }
    }
    
    // 2. 집계 쿼리 최적화
    public class AggregationStrategy {
        public AggregationResult executeAggregation(AggregationQuery query) {
            if (query.canParallelize()) {
                // 병렬 집계 실행
                return executeParallelAggregation(query);
            } else {
                // 순차적 집계
                return executeSequentialAggregation(query);
            }
        }
    }
}
```

### 5.3 운영 복잡성 증가
```java
@Service
public class OperationalComplexityHandler {
    
    // 1. 백업 복잡성 관리
    public class BackupManager {
        @Scheduled(cron = "0 0 1 * * *") // 매일 새벽 1시
        public void performBackup() {
            // 전체 샤드 동시 백업
            List<CompletableFuture<Void>> backupTasks = shards.stream()
                .map(this::backupShard)
                .collect(Collectors.toList());
                
            // 정합성 보장을 위한 백업 시점 동기화
            CompletableFuture.allOf(backupTasks.toArray(new CompletableFuture[0]))
                .thenRun(this::validateBackupConsistency);
        }
    }
    
    // 2. 스키마 변경 관리
    public class SchemaManager {
        public void applySchemaChange(SchemaChange change) {
            // 점진적 스키마 변경
            for (int shardId : shards) {
                try {
                    applyChangeToShard(shardId, change);
                    validateSchemaChange(shardId);
                } catch (Exception e) {
                    rollbackSchemaChange(shardId);
                    throw new SchemaChangeException("Schema change failed", e);
                }
            }
        }
    }
}
```

### 5.4 비용 증가
```java
@Service
public class CostOptimizationManager {
    
    // 1. 리소스 최적화
    public class ResourceOptimizer {
        @Scheduled(fixedRate = 300000) // 5분마다
        public void optimizeResources() {
            // 샤드별 리소스 사용량 모니터링
            Map<Integer, ResourceMetrics> metrics = collectResourceMetrics();
            
            // 자동 스케일링 결정
            metrics.forEach((shardId, metric) -> {
                if (metric.requiresScaling()) {
                    adjustResources(shardId, metric);
                }
            });
        }
    }
    
    // 2. 비용 모니터링
    public class CostMonitor {
        public void trackShardingCosts() {
            // 샤드별 운영 비용 계산
            Map<Integer, CostMetrics> costMetrics = calculateShardCosts();
            
            // 비용 최적화 추천
            List<CostOptimizationSuggestion> suggestions = 
                analyzeCostOptimizations(costMetrics);
                
            if (!suggestions.isEmpty()) {
                alertOperations("Cost optimization suggestions available",
                    suggestions);
            }
        }
    }
}
```

### 5.5 유연성 감소 대응
```java
@Service
public class FlexibilityManager {
    
    // 1. 샤딩 키 변경 지원
    public class ShardingKeyMigrator {
        public void migrateShardingKey(
                String oldKey, 
                String newKey, 
                MigrationStrategy strategy) {
                
            // 이중 쓰기 기간 시작
            enableDualWritePhase(oldKey, newKey);
            
            // 점진적 데이터 마이그레이션
            migrateHistoricalData(oldKey, newKey, strategy);
            
            // 새로운 키로 전환
            switchToNewKey(oldKey, newKey);
        }
    }
    
    // 2. 샤드 재구성 유연성
    public class ShardReorganizer {
        public void reorganizeShards(ReorganizationPlan plan) {
            // 임시 버퍼 샤드 생성
            createBufferShards(plan);
            
            // 점진적 데이터 재분배
            redistributeData(plan);
            
            // 라우팅 업데이트
            updateRoutingRules(plan);
        }
    }
}
```

이러한 트레이드오프들을 인식하고 관리하는 것이 중요하며, 각 상황에 맞는 최적의 전략을 선택해야 합니다. 특히:

1. 초기 설계 시 충분한 확장성 고려
2. 운영 복잡성을 줄이기 위한 자동화 도구 개발
3. 비용 대비 효율성 모니터링
4. 유연한 아키텍처 유지를 위한 지속적인 개선

이 필요합니다.