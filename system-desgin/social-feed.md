# 소셜 미디어 피드 시스템 설계

면접관: "Twitter나 Instagram과 같은 소셜 미디어 피드 시스템을 설계해주세요. 게시물 작성, 팔로우 관계, 피드 생성 기능이 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 예상 사용자 수와 일일 게시물 수는 어느 정도인가요?
2. 피드에 표시될 게시물의 시간 범위는 어떻게 되나요?
3. 게시물 종류(텍스트, 이미지 등)는 어떻게 되나요?
4. 실시간성이 얼마나 중요한가요?

면접관:
1. DAU 1000만, 일일 게시물 500만개
2. 최근 1주일 게시물 우선, 정렬은 시간순
3. 텍스트, 이미지, 짧은 동영상 지원
4. 1분 이내 업데이트 필요

## 1. 피드 시스템 설계

```java
@Service
public class FeedService {
    
    private final PostRepository postRepository;
    private final UserGraphService userGraphService;
    private final RedisCacheManager cacheManager;
    private final KafkaTemplate<String, FeedEvent> kafkaTemplate;

    // 1. 피드 생성 (Pull 모델)
    public List<Post> generateUserFeed(String userId, int page, int size) {
        // 캐시 확인
        String cacheKey = "feed:" + userId + ":" + page;
        List<Post> cachedFeed = cacheManager.get(cacheKey);
        
        if (cachedFeed != null) {
            return cachedFeed;
        }

        // 팔로우하는 사용자 목록 조회
        Set<String> following = userGraphService.getFollowing(userId);
        
        // 최근 게시물 조회 (Redis Sorted Set 사용)
        List<Post> feed = postRepository.findLatestPosts(following, page, size);
        
        // 캐시 저장 (5분 TTL)
        cacheManager.set(cacheKey, feed, Duration.ofMinutes(5));
        
        return feed;
    }

    // 2. 게시물 생성과 팬아웃 (Push 모델)
    @Transactional
    public void createPost(Post post) {
        // 게시물 저장
        postRepository.save(post);
        
        // 피드 이벤트 발행
        FeedEvent event = FeedEvent.builder()
            .type(FeedEventType.NEW_POST)
            .postId(post.getId())
            .userId(post.getUserId())
            .timestamp(Instant.now())
            .build();
            
        kafkaTemplate.send("feed-events", event);
    }
}
```

## 2. 게시물 처리와 피드 전파

```java
@Service
@Slf4j
public class PostProcessingService {
    
    private final UserGraphService userGraphService;
    private final NotificationService notificationService;
    private final FeedCacheManager feedCacheManager;

    // 1. 팬아웃 처리 (Fan-out on write)
    @KafkaListener(topics = "feed-events")
    public void handleFeedEvent(FeedEvent event) {
        if (event.getType() == FeedEventType.NEW_POST) {
            // 영향력 있는 사용자(많은 팔로워) 확인
            if (isInfluencer(event.getUserId())) {
                // 영향력 있는 사용자는 Pull 모델 사용
                handleInfluencerPost(event);
            } else {
                // 일반 사용자는 Push 모델 사용
                fanoutToFollowers(event);
            }
        }
    }

    private void fanoutToFollowers(FeedEvent event) {
        // 팔로워 목록 조회 (페이지네이션 적용)
        Pageable pageable = PageRequest.of(0, 1000);
        Page<String> followers;
        
        do {
            followers = userGraphService.getFollowers(
                event.getUserId(), pageable);
            
            // 배치로 피드 업데이트
            List<CompletableFuture<Void>> futures = 
                followers.getContent().stream()
                    .map(followerId -> CompletableFuture.runAsync(() ->
                        updateUserFeed(followerId, event.getPostId())))
                    .collect(Collectors.toList());
                    
            // 배치 처리 완료 대기
            CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0]))
                .join();
                
            pageable = pageable.next();
        } while (followers.hasNext());
    }

    // 2. 피드 캐시 관리
    private void updateUserFeed(String userId, String postId) {
        String feedKey = "feed:" + userId;
        try {
            feedCacheManager.zAdd(feedKey, 
                System.currentTimeMillis(), postId);
                
            // 피드 크기 제한 (최근 1000개 유지)
            feedCacheManager.zRemRangeByRank(
                feedKey, 0, -1001);
                
        } catch (Exception e) {
            log.error("Failed to update feed cache for user: " + 
                userId, e);
            // 실패한 업데이트는 재처리 큐에 추가
            retryQueue.add(new FeedUpdate(userId, postId));
        }
    }

    // 3. 실시간 업데이트 처리
    @Service
    public class RealTimeUpdateService {
        private final SseEmitters emitters = 
            new ConcurrentHashMap<String, SseEmitter>();
            
        public SseEmitter subscribe(String userId) {
            SseEmitter emitter = new SseEmitter(
                Duration.ofMinutes(10).toMillis());
                
            emitters.put(userId, emitter);
            
            emitter.onCompletion(() -> 
                emitters.remove(userId));
            emitter.onTimeout(() -> 
                emitters.remove(userId));
                
            return emitter;
        }
        
        public void sendUpdate(String userId, Post post) {
            SseEmitter emitter = emitters.get(userId);
            if (emitter != null) {
                try {
                    emitter.send(SseEmitter.event()
                        .name("post")
                        .data(post));
                } catch (IOException e) {
                    emitters.remove(userId);
                }
            }
        }
    }
}

@Service
public class FeedRankingService {
    
    // 4. 피드 랭킹과 개인화
    public List<Post> rankFeed(String userId, List<Post> posts) {
        // 사용자 관심사 로드
        UserPreferences prefs = 
            userPreferenceService.getPreferences(userId);
            
        // 게시물 점수 계산
        return posts.stream()
            .map(post -> {
                double score = calculatePostScore(post, prefs);
                return new ScoredPost(post, score);
            })
            .sorted(Comparator.comparingDouble(
                ScoredPost::getScore).reversed())
            .map(ScoredPost::getPost)
            .collect(Collectors.toList());
    }
    
    private double calculatePostScore(Post post, 
                                    UserPreferences prefs) {
        return PostScorer.builder()
            .recency(getRecencyScore(post.getCreatedAt()))
            .engagement(getEngagementScore(post))
            .relevance(getRelevanceScore(post, prefs))
            .build()
            .calculateFinalScore();
    }
    
    // 5. 성능 최적화
    @Cacheable(value = "postScores", 
               key = "#post.id + ':' + #prefs.userId")
    public double getRelevanceScore(Post post, 
                                   UserPreferences prefs) {
        // 텍스트 유사도 계산
        double textSimilarity = 
            textAnalyzer.calculateSimilarity(
                post.getContent(), 
                prefs.getInterests());
                
        // 상호작용 기반 유사도
        double interactionSimilarity = 
            interactionAnalyzer.calculateSimilarity(
                post.getUserId(), 
                prefs.getInteractionHistory());
                
        return 0.7 * textSimilarity + 0.3 * interactionSimilarity;
    }
}
```

이러한 설계를 통해:

1. 하이브리드 팬아웃 전략
    - 일반 사용자: Push 모델
    - 영향력 있는 사용자: Pull 모델
    - 최적의 성능과 리소스 사용

2. 효율적인 캐시 관리
    - 피드 캐싱
    - 점수 캐싱
    - TTL 기반 만료

3. 실시간 업데이트
    - SSE를 통한 실시간 전송
    - 연결 관리
    - 실패 처리

4. 개인화된 피드 랭킹
    - 다중 요소 점수 계산
    - 사용자 선호도 반영
    - 실시간 업데이트

를 구현할 수 있습니다.

면접관: 대규모 사용자의 피드 업데이트 시 발생할 수 있는 성능 이슈는 어떻게 해결하시겠습니까?

## 3. 성능 최적화 전략

```java
@Service
public class FeedOptimizationService {

    // 1. 피드 캐싱 계층화
    @Component
    public class MultilevelCache {
        private final LoadingCache<String, List<Post>> localCache;
        private final RedisTemplate<String, List<Post>> redisCache;
        
        public MultilevelCache() {
            this.localCache = Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(5))
                .build(key -> loadFromRedis(key));
        }
        
        public List<Post> getFeed(String userId) {
            try {
                // L1 캐시 (로컬 메모리)
                return localCache.get(userId);
            } catch (Exception e) {
                // L2 캐시 (Redis)
                String redisKey = "feed:" + userId;
                List<Post> feed = redisCache.opsForValue().get(redisKey);
                
                if (feed == null) {
                    // DB에서 로드
                    feed = loadFromDatabase(userId);
                    redisCache.opsForValue().set(redisKey, feed, 
                        Duration.ofMinutes(30));
                }
                return feed;
            }
        }
    }

    // 2. 배치 처리 최적화
    @Service
    public class BatchProcessor {
        private final BlockingQueue<FeedUpdate> updateQueue = 
            new LinkedBlockingQueue<>(10_000);
            
        @Scheduled(fixedRate = 100) // 100ms마다 실행
        public void processBatch() {
            List<FeedUpdate> batch = new ArrayList<>();
            updateQueue.drainTo(batch, 1000); // 최대 1000개씩 처리
            
            if (!batch.isEmpty()) {
                // 배치 단위로 업데이트 그룹화
                Map<String, List<String>> userUpdates = batch.stream()
                    .collect(Collectors.groupingBy(
                        FeedUpdate::getUserId,
                        Collectors.mapping(
                            FeedUpdate::getPostId, 
                            Collectors.toList())));
                            
                // 병렬 처리
                userUpdates.entrySet().parallelStream()
                    .forEach(entry -> 
                        updateUserFeedBatch(
                            entry.getKey(), 
                            entry.getValue()));
            }
        }
    }

    // 3. 메모리 사용 최적화
    @Component
    public class MemoryOptimizer {
        
        public List<Post> getFeedOptimized(String userId) {
            // 필요한 필드만 조회
            return postRepository.findFeedProjection(userId)
                .stream()
                .map(this::convertToLightweightPost)
                .collect(Collectors.toList());
        }
        
        // 경량화된 게시물 객체
        @Value
        public class LightweightPost {
            String id;
            String content;
            long timestamp;
            // 필수 필드만 포함
        }
    }

    // 4. 데이터베이스 최적화
    @Repository
    public class OptimizedPostRepository {
        
        @QueryHint(name = HINT_FETCH_SIZE, value = "1000")
        @Query(value = """
            SELECT p.id, p.content, p.timestamp
            FROM posts p
            WHERE p.user_id IN (
                SELECT following_id 
                FROM user_follows 
                WHERE follower_id = :userId
            )
            AND p.timestamp > :cutoffTime
            ORDER BY p.timestamp DESC
            """)
        List<PostProjection> findFeedProjection(
            @Param("userId") String userId,
            @Param("cutoffTime") long cutoffTime);
            
        // 인덱스 최적화
        @Query(value = """
            CREATE INDEX idx_posts_user_timestamp 
            ON posts(user_id, timestamp DESC)
            """)
        void createOptimizedIndex();
    }

    // 5. 부하 분산
    @Configuration
    public class LoadBalancingConfig {
        
        @Bean
        public ShardingStrategy shardingStrategy() {
            return new ConsistentHashingStrategy(
                shardCount = 128,
                replicationFactor = 3
            );
        }
        
        // 사용자 기반 샤딩
        public String determinePostShard(String userId) {
            int shardId = shardingStrategy.getShardId(userId);
            return "shard_" + shardId;
        }
    }

    // 6. 성능 모니터링
    @Component
    @Slf4j
    public class PerformanceMonitor {
        
        private final MeterRegistry registry;
        
        public void recordFeedGeneration(String userId, long duration) {
            registry.timer("feed.generation.time")
                .record(duration, TimeUnit.MILLISECONDS);
                
            if (duration > 1000) { // 1초 초과
                log.warn("Slow feed generation for user: {}", userId);
                analyzeSlowFeed(userId);
            }
        }
        
        @Scheduled(fixedRate = 60000) // 1분마다
        public void checkPerformanceMetrics() {
            // 주요 성능 지표 확인
            double p99Latency = registry.timer("feed.generation.time")
                .percentile(0.99);
                
            if (p99Latency > 2000) { // 2초 초과
                triggerAutomaticOptimization();
            }
        }
    }
}
```

이러한 성능 최적화 전략을 통해:

1. 캐시 최적화
    - 다층 캐싱으로 응답 시간 단축
    - 메모리 사용 효율화
    - 캐시 갱신 최적화

2. 배치 처리
    - 업데이트 그룹화
    - 병렬 처리
    - 큐 기반 처리

3. 메모리 사용 최적화
    - 필요한 데이터만 로드
    - 경량화된 객체 사용
    - 메모리 압력 감소

4. 데이터베이스 최적화
    - 인덱스 최적화
    - 쿼리 최적화
    - 커넥션 풀 관리

5. 부하 분산
    - 샤딩 전략
    - 일관된 해싱
    - 데이터 복제

6. 모니터링 및 자동 최적화
    - 실시간 성능 모니터링
    - 자동화된 최적화
    - 알림 및 경고

이를 통해 대규모 사용자의 피드 업데이트를 효율적으로 처리할 수 있습니다.