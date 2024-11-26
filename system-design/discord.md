# Discord 채팅 검색 시스템 설계

면접관: "Discord와 같은 실시간 채팅 시스템에서 대량의 메시지 중에서 특정 단어나 문구를 검색하여 해당 채팅 스레드를 찾는 시스템을 설계해주세요."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 하루 평균 메시지 양과 검색 쿼리 수는 얼마나 되나요?
2. 검색 결과의 실시간성이 얼마나 중요한가요?
3. 검색 범위(기간, 채널 등)에 대한 제한이 있나요?
4. 검색 결과의 정확도와 속도 중 무엇이 더 중요한가요?

면접관:
1. 일일 1억 개의 새로운 메시지, 초당 1000개의 검색 쿼리
2. 1분 이내의 지연 시간 허용
3. 최대 1년치 메시지 검색 지원, 채널/서버별 검색 필요
4. 정확도와 속도 모두 중요, 검색 결과는 1초 이내 반환 필요

## 1. 검색 인덱싱 시스템

```java
@Service
public class MessageIndexingService {
    
    private final ElasticsearchClient esClient;
    private final KafkaTemplate<String, Message> kafkaTemplate;
    
    // 1. 실시간 메시지 인덱싱
    @KafkaListener(topics = "chat-messages")
    public void indexMessage(Message message) {
        // 메시지 전처리
        IndexableMessage indexableMsg = preprocessMessage(message);
        
        // 비동기 인덱싱
        CompletableFuture.runAsync(() -> {
            IndexRequest<IndexableMessage> request = IndexRequest.of(r -> r
                .index("messages")
                .id(message.getId())
                .document(indexableMsg)
                .routing(message.getServerId()) // 서버별 샤딩
                .pipeline("message-enrichment")); // 인덱싱 파이프라인
                
            try {
                esClient.index(request);
            } catch (Exception e) {
                // 실패한 인덱싱은 재시도 큐로
                kafkaTemplate.send("failed-indexing", message);
            }
        });
    }
    
    // 2. 인덱스 최적화
    @Data
    @Document(indexName = "messages")
    public class IndexableMessage {
        @Id
        private String id;
        
        @Field(type = FieldType.Text, analyzer = "custom_analyzer")
        private String content;
        
        @Field(type = FieldType.Keyword)
        private String serverId;
        
        @Field(type = FieldType.Keyword)
        private String channelId;
        
        @Field(type = FieldType.Date)
        private Instant timestamp;
        
        @Field(type = FieldType.Nested)
        private List<Attachment> attachments;
    }
    
    // 3. 커스텀 분석기 설정
    @Bean
    public Analysis createCustomAnalyzer() {
        return Analysis.of(a -> a
            .analyzer("custom_analyzer", 
                CustomAnalyzer.of(ca -> ca
                    .tokenizer("standard")
                    .filter("lowercase", 
                            "stop",
                            "snowball",
                            "edge_ngram"))));
    }
}
```

## 2. 검색 쿼리 처리 시스템

```java
@Service
public class SearchService {

    private final ElasticsearchClient esClient;
    private final SearchCache searchCache;
    private final SearchOptimizer searchOptimizer;

    // 1. 검색 쿼리 처리
    public SearchResult search(SearchRequest request) {
        // 캐시 확인
        String cacheKey = generateCacheKey(request);
        SearchResult cachedResult = searchCache.get(cacheKey);
        if (cachedResult != null) {
            return cachedResult;
        }

        // 검색 쿼리 구성
        Query query = buildSearchQuery(request);
        
        // 검색 실행
        SearchResponse<IndexableMessage> response = esClient.search(s -> s
            .index("messages")
            .routing(request.getServerId())  // 서버별 샤딩 활용
            .query(query)
            .highlight(h -> h
                .fields("content", f -> f
                    .preTags("<mark>")
                    .postTags("</mark>")
                    .numberOfFragments(3)))
            .sort(so -> so
                .field(f -> f.field("timestamp").order(SortOrder.Desc)))
            .from(request.getOffset())
            .size(request.getLimit())
            , IndexableMessage.class
        );

        // 결과 변환 및 캐시 저장
        SearchResult result = convertToSearchResult(response);
        searchCache.put(cacheKey, result);
        return result;
    }

    // 2. 고급 검색 쿼리 구성
    private Query buildSearchQuery(SearchRequest request) {
        BoolQuery.Builder queryBuilder = new BoolQuery.Builder();

        // 키워드 검색
        queryBuilder.must(m -> m
            .matchPhrasePrefix(mp -> mp
                .field("content")
                .query(request.getKeyword())
                .maxExpansions(50)));

        // 필터 적용
        if (request.getChannelId() != null) {
            queryBuilder.filter(f -> f
                .term(t -> t
                    .field("channelId")
                    .value(request.getChannelId())));
        }

        // 시간 범위 필터
        if (request.getTimeRange() != null) {
            queryBuilder.filter(f -> f
                .range(r -> r
                    .field("timestamp")
                    .gte(JsonData.of(request.getTimeRange().getStart()))
                    .lte(JsonData.of(request.getTimeRange().getEnd()))));
        }

        return queryBuilder.build()._toQuery();
    }

    // 3. 검색 최적화
    @Component
    public class SearchOptimizer {
        
        // 검색어 자동 완성
        public List<String> getSuggestions(String prefix) {
            return esClient.search(s -> s
                .index("message_suggestions")
                .suggest(sug -> sug
                    .suggestor("content_suggest", cs -> cs
                        .prefix(prefix)
                        .completion(c -> c
                            .field("suggest")
                            .size(5)
                            .skipDuplicates(true)))), 
                IndexableMessage.class)
                .suggest()
                .get("content_suggest")
                .stream()
                .map(suggestion -> suggestion.text())
                .collect(Collectors.toList());
        }

        // 검색 결과 스코어링
        private double calculateRelevanceScore(SearchHit<IndexableMessage> hit) {
            double score = hit.score();
            
            // 시간 가중치
            long ageInHours = Duration.between(
                hit.source().getTimestamp(), 
                Instant.now()).toHours();
            double timeBoost = 1.0 / Math.log(ageInHours + 2);
            
            // 채널 활성도 가중치
            double channelActivityBoost = 
                getChannelActivityScore(hit.source().getChannelId());
            
            return score * timeBoost * channelActivityBoost;
        }
    }

    // 4. 검색 결과 캐싱
    @Component
    public class SearchCache {
        private final LoadingCache<String, SearchResult> cache;

        public SearchCache() {
            this.cache = Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(5))
                .build(this::loadSearchResult);
        }

        private SearchResult loadSearchResult(String cacheKey) {
            SearchRequest request = deserializeRequest(cacheKey);
            return performSearch(request);
        }

        // 캐시 무효화 (새 메시지 도착 시)
        public void invalidateRelatedCache(Message message) {
            // 관련된 검색 결과 캐시 무효화
            Set<String> relatedCacheKeys = findRelatedCacheKeys(message);
            relatedCacheKeys.forEach(cache::invalidate);
        }
    }

    // 5. 분산 검색 처리
    @Component
    public class DistributedSearchHandler {
        
        public SearchResult executeDistributedSearch(SearchRequest request) {
            // 서버/채널별 샤드 결정
            List<SearchShard> targetShards = determineTargetShards(request);
            
            // 병렬 검색 실행
            List<CompletableFuture<PartialSearchResult>> futures = 
                targetShards.stream()
                    .map(shard -> CompletableFuture.supplyAsync(() ->
                        executeSearchOnShard(request, shard)))
                    .collect(Collectors.toList());

            // 결과 병합
            return CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0]))
                .thenApply(v -> mergeResults(futures))
                .join();
        }

        private SearchResult mergeResults(
            List<CompletableFuture<PartialSearchResult>> futures) {
            
            return futures.stream()
                .map(CompletableFuture::join)
                .reduce(new SearchResult(), this::mergePartialResults);
        }
    }
}
```

이러한 검색 시스템을 통해:

1. 효율적인 인덱싱
    - 실시간 메시지 인덱싱
    - 커스텀 분석기로 검색 품질 향상
    - 샤딩을 통한 성능 최적화

2. 고성능 검색
    - 캐싱을 통한 응답 시간 개선
    - 분산 검색으로 대규모 데이터 처리
    - 결과 스코어링으로 정확도 향상

3. 사용자 경험 최적화
    - 검색어 자동 완성
    - 하이라이팅
    - 관련성 기반 정렬

4. 확장성
    - 샤딩을 통한 수평적 확장
    - 캐시를 통한 부하 분산
    - 비동기 처리로 성능 향상

을 구현할 수 있습니다.

면접관: 대규모 히스토리 검색 시 발생할 수 있는 성능 이슈는 어떻게 해결하시겠습니까?


## 3. 대규모 히스토리 검색 성능 최적화 전략

```java
@Service
public class HistoricalSearchOptimizationService {

    // 1. 시간 기반 인덱스 샤딩
    @Component
    public class TimeBasedIndexManager {
        private static final String INDEX_PATTERN = "messages-%s";
        
        public List<String> determineTargetIndices(TimeRange searchRange) {
            List<String> targetIndices = new ArrayList<>();
            
            // 월별 인덱스 구성
            YearMonth current = YearMonth.from(searchRange.getStart());
            YearMonth end = YearMonth.from(searchRange.getEnd());
            
            while (!current.isAfter(end)) {
                targetIndices.add(String.format(
                    INDEX_PATTERN, 
                    current.format(DateTimeFormatter.ofPattern("yyyy-MM"))
                ));
                current = current.plusMonths(1);
            }
            
            return targetIndices;
        }

        // 인덱스 수명 주기 관리
        @Scheduled(cron = "0 0 0 * * *")
        public void manageIndices() {
            // 오래된 인덱스 압축
            compressOldIndices();
            
            // 아카이브 정책 적용
            applyArchivePolicy();
        }
    }

    // 2. 검색 결과 페이징 최적화
    @Component
    public class SearchPaginationOptimizer {
        private final RedisTemplate<String, String> redisTemplate;
        
        public SearchResult getNextPage(String searchId, int page) {
            // 검색 컨텍스트 복원
            SearchContext context = getSearchContext(searchId);
            
            // search_after 사용하여 페이징
            SearchResponse<IndexableMessage> response = esClient.search(s -> s
                .index(context.getTargetIndices())
                .query(context.getQuery())
                .searchAfter(context.getLastSort())
                .sort(so -> so
                    .field(f -> f.field("timestamp").order(SortOrder.Desc))
                    .field(f -> f.field("_id").order(SortOrder.Desc)))
                .size(context.getPageSize())
            );
            
            // 다음 페이지 컨텍스트 저장
            updateSearchContext(searchId, response);
            
            return convertToSearchResult(response);
        }

        // 검색 컨텍스트 관리
        private class SearchContext {
            private final List<String> targetIndices;
            private final Query query;
            private final List<Sort> sorts;
            private List<Object> lastSort;
            private final int pageSize;
            
            // TTL 설정
            @PostConstruct
            public void setTTL() {
                redisTemplate.expire(
                    "search:context:" + searchId, 
                    Duration.ofHours(1)
                );
            }
        }
    }

    // 3. 결과 캐싱 계층화
    @Component
    public class LayeredSearchCache {
        private final LoadingCache<String, SearchResult> localCache;
        private final RedisTemplate<String, SearchResult> redisCache;
        
        public SearchResult getCachedResult(SearchRequest request) {
            String cacheKey = generateCacheKey(request);
            
            // L1 캐시 (로컬 메모리)
            SearchResult localResult = localCache.getIfPresent(cacheKey);
            if (localResult != null) {
                return localResult;
            }
            
            // L2 캐시 (Redis)
            SearchResult redisResult = redisCache.opsForValue()
                .get("search:" + cacheKey);
            if (redisResult != null) {
                localCache.put(cacheKey, redisResult);
                return redisResult;
            }
            
            // 캐시 미스
            return null;
        }
    }

    // 4. 쿼리 최적화
    @Component
    public class QueryOptimizer {
        
        public Query optimizeQuery(Query originalQuery) {
            BoolQuery.Builder optimizedQuery = new BoolQuery.Builder();
            
            // 필드 가중치 적용
            optimizedQuery.should(s -> s
                .matchPhrase(mp -> mp
                    .field("content^2")  // 컨텐츠 필드 가중치
                    .query(originalQuery)));
                    
            // 필터 최적화
            optimizedQuery.filter(f -> f
                .bool(b -> b
                    .minimumShouldMatch(1)
                    .cache(true)));  // 필터 캐싱
                    
            return optimizedQuery.build()._toQuery();
        }
        
        // 쿼리 실행 계획 분석
        public QueryPlan analyzeQuery(Query query) {
            return esClient.indices()
                .validateQuery(v -> v
                    .query(query)
                    .explain(true))
                .explain();
        }
    }

    // 5. 비동기 검색 처리
    @Service
    public class AsyncSearchService {
        
        public CompletableFuture<SearchResult> asyncSearch(
            SearchRequest request) {
            
            // 검색 작업 생성
            String searchTaskId = UUID.randomUUID().toString();
            
            // 비동기 검색 실행
            return CompletableFuture.supplyAsync(() -> {
                try {
                    return executeSearch(request);
                } catch (Exception e) {
                    // 실패 처리
                    handleSearchFailure(searchTaskId, e);
                    throw e;
                }
            }).whenComplete((result, error) -> {
                if (error == null) {
                    // 성공 시 결과 캐싱
                    cacheSearchResult(searchTaskId, result);
                }
            });
        }
        
        // 검색 진행 상황 조회
        public SearchProgress getSearchProgress(String searchTaskId) {
            return new SearchProgress(
                searchTaskId,
                getCompletedShards(),
                getTotalShards(),
                getEstimatedTimeRemaining()
            );
        }
    }
}
```

이러한 최적화 전략을 통해:

1. 시간 기반 인덱스 분할
    - 월별/연도별 인덱스 생성
    - 오래된 데이터 압축/아카이빙
    - 검색 범위 최적화

2. 효율적인 페이징
    - search_after 사용
    - 검색 컨텍스트 관리
    - 상태 유지 페이징

3. 다층 캐싱
    - 로컬 캐시
    - 분산 캐시
    - 결과 재사용

4. 쿼리 최적화
    - 필드 가중치
    - 필터 캐싱
    - 실행 계획 분석

5. 비동기 처리
    - 백그라운드 검색
    - 진행 상황 모니터링
    - 부분 결과 반환

을 구현하여 대규모 히스토리 검색의 성능을 개선할 수 있습니다.