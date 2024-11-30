면접관: "LoL 전적 검색 시스템을 설계해야 합니다. Riot API를 사용해서 데이터를 수집하고 사용자들에게 서비스해야 하는데요. 어떻게 접근하시겠습니까?"

저: "네, 먼저 시스템의 주요 요구사항과 제약사항을 정리하고 싶습니다. 가장 중요한 것은 Riot API의 rate limit 관리일 것 같은데요. 이 부분에 대해 자세히 알 수 있을까요?"

면접관: "네, Riot API는 Development API key의 경우 초당 20 요청, Production API key의 경우 10초당 500 요청으로 제한됩니다. 또한 각 엔드포인트별로도 별도의 rate limit이 있습니다."

저: "이해했습니다. 그렇다면 먼저 Rate Limit 관리를 위한 코드를 작성해보겠습니다:"



```java
@Service
public class RateLimitManager {
    private final Map<String, RateLimiter> rateLimiters;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    public RateLimitManager(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.objectMapper = new ObjectMapper();
        
        // 각 엔드포인트별 Rate Limiter 초기화
        this.rateLimiters = new ConcurrentHashMap<>();
        rateLimiters.put("application", RateLimiter.create(50.0)); // 초당 50 요청
        rateLimiters.put("summonerByName", RateLimiter.create(2000.0/120.0)); // 2000 requests/2min
        rateLimiters.put("matchHistory", RateLimiter.create(1000.0/60.0)); // 1000 requests/1min
    }

    public <T> CompletableFuture<T> executeWithRateLimit(
            String endpoint, 
            Supplier<T> apiCall) {
        
        if (!rateLimiters.get("application").tryAcquire() || 
            !rateLimiters.get(endpoint).tryAcquire()) {
            
            // Rate limit 초과시 큐에 작업 추가
            return queueRequest(endpoint, apiCall);
        }

        return CompletableFuture.supplyAsync(apiCall);
    }

    private <T> CompletableFuture<T> queueRequest(
            String endpoint, 
            Supplier<T> apiCall) {
        
        ApiRequest<T> request = new ApiRequest<>(endpoint, apiCall);
        String queueKey = "riot-api-queue:" + endpoint;
        
        try {
            String serializedRequest = objectMapper.writeValueAsString(request);
            redisTemplate.opsForList().rightPush(queueKey, serializedRequest);
            
            // 비동기 결과를 위한 CompletableFuture 생성
            CompletableFuture<T> future = new CompletableFuture<>();
            
            // 요청 처리 상태를 추적하기 위한 키 저장
            String requestId = UUID.randomUUID().toString();
            redisTemplate.opsForValue().set(
                "request-status:" + requestId, 
                "QUEUED", 
                Duration.ofHours(1)
            );
            
            return future;
            
        } catch (JsonProcessingException e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}

```

저: "이렇게 Rate Limit 관리를 구현했습니다. Token Bucket 알고리즘을 사용하고, 초과된 요청은 Redis에 큐잉하는 방식입니다. 추가로 큐잉된 요청을 처리하는 워커도 필요한데, 이를 구현해볼까요?"

면접관: "네, 좋습니다. 큐잉된 요청은 어떤 순서로 처리할 계획인가요?"

저: "우선순위에 따라 처리하려고 합니다. 예를 들어 최근에 게임을 한 유저의 요청이나, 실시간으로 조회중인 유저의 요청을 우선 처리하는 방식으로요. 다음과 같이 구현하겠습니다:"

```java
@Component
public class QueuedRequestProcessor {
    private final RateLimitManager rateLimitManager;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    @Scheduled(fixedRate = 100) // 100ms 마다 실행
    public void processQueue() {
        for (String endpoint : rateLimitManager.getEndpoints()) {
            String queueKey = "riot-api-queue:" + endpoint;
            
            // 우선순위가 높은 요청부터 처리
            String request = redisTemplate.opsForList().leftPop(queueKey);
            if (request != null) {
                try {
                    ApiRequest<?> apiRequest = objectMapper.readValue(
                        request, 
                        ApiRequest.class
                    );
                    
                    if (shouldProcess(apiRequest)) {
                        processRequest(apiRequest);
                    } else {
                        // 나중에 처리하도록 다시 큐에 넣음
                        redisTemplate.opsForList().rightPush(queueKey, request);
                    }
                    
                } catch (JsonProcessingException e) {
                    log.error("Failed to process request", e);
                }
            }
        }
    }

    private boolean shouldProcess(ApiRequest<?> request) {
        // 우선순위 결정 로직
        String summonerId = request.getSummonerId();
        
        // 1. 실시간 조회 중인 유저 체크
        Boolean isActiveUser = redisTemplate.hasKey(
            "active-user:" + summonerId
        );
        
        if (Boolean.TRUE.equals(isActiveUser)) {
            return true;
        }
        
        // 2. 최근 게임 시간 체크
        String lastGameTime = redisTemplate.opsForValue().get(
            "last-game:" + summonerId
        );
        
        if (lastGameTime != null) {
            long timeSinceLastGame = System.currentTimeMillis() - 
                Long.parseLong(lastGameTime);
            return timeSinceLastGame < TimeUnit.HOURS.toMillis(1);
        }
        
        return false;
    }

    private <T> void processRequest(ApiRequest<T> request) {
        if (rateLimitManager.canProcess(request.getEndpoint())) {
            try {
                T result = request.getApiCall().get();
                redisTemplate.opsForValue().set(
                    "request-result:" + request.getId(), 
                    objectMapper.writeValueAsString(result)
                );
                redisTemplate.opsForValue().set(
                    "request-status:" + request.getId(), 
                    "COMPLETED"
                );
            } catch (Exception e) {
                redisTemplate.opsForValue().set(
                    "request-status:" + request.getId(), 
                    "FAILED"
                );
            }
        }
    }
}

```

저: "이렇게 구현하면 Rate Limit을 준수하면서도 우선순위에 따라 효율적으로 요청을 처리할 수 있습니다. 실시간 조회 중인 유저나 최근에 게임을 한 유저의 요청을 우선적으로 처리하고, 나머지는 여유 있을 때 처리하는 방식입니다."

면접관: "실제로 클라이언트에서 이 시스템을 어떻게 사용하게 되나요? 특히 큐잉된 요청의 결과를 어떻게 전달할 계획인가요?"

저: "클라이언트와의 연동을 위한 API와 WebSocket 구현을 보여드리겠습니다..."

저: "클라이언트와의 연동을 위해 다음과 같이 구현하겠습니다:"

```java
@RestController
@RequestMapping("/api/summoner")
public class SummonerController {
    private final RateLimitManager rateLimitManager;
    private final SimpMessagingTemplate websocket;
    private final RedisTemplate<String, String> redisTemplate;

    @GetMapping("/search/{summonerName}")
    public ResponseEntity<?> searchSummoner(@PathVariable String summonerName) {
        // 1. 캐시 확인
        String cacheKey = "summoner:" + summonerName;
        String cachedData = redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedData != null) {
            SummonerDTO summoner = objectMapper.readValue(cachedData, SummonerDTO.class);
            return ResponseEntity.ok()
                .header("X-Data-Source", "cache")
                .body(summoner);
        }

        // 2. API 호출 시도
        CompletableFuture<SummonerDTO> future = rateLimitManager.executeWithRateLimit(
            "summonerByName",
            () -> riotClient.getSummonerByName(summonerName)
        );

        try {
            SummonerDTO summoner = future.getNow(null);
            if (summoner != null) {
                // 즉시 응답 가능한 경우
                return ResponseEntity.ok(summoner);
            } else {
                // 큐잉된 경우
                String requestId = future.handle((result, ex) -> {
                    if (result != null) {
                        // WebSocket으로 결과 전송
                        websocket.convertAndSend(
                            "/topic/summoner/" + summonerName, 
                            result
                        );
                    }
                    return ex == null ? "SUCCESS" : "FAILED";
                }).toString();

                return ResponseEntity.accepted()
                    .header("X-Request-ID", requestId)
                    .build();
            }
        } catch (Exception e) {
            return ResponseEntity.status(503)
                .body(new ErrorResponse("Service temporarily unavailable"));
        }
    }
}

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOrigins("*")
            .withSockJS();
    }
}

```

저: "이런 식으로 구현하면 클라이언트는 다음과 같은 흐름으로 데이터를 받을 수 있습니다:

1. REST API로 소환사 검색 요청
2. 캐시된 데이터가 있으면 즉시 반환
3. Rate Limit 여유가 있으면 실시간 데이터 반환
4. Rate Limit 초과시:
    - 202 Accepted 응답과 함께 요청 ID 반환
    - 클라이언트는 WebSocket으로 업데이트 구독
    - 데이터 준비되면 WebSocket으로 전송

이렇게 하면 사용자 경험을 해치지 않으면서도 시스템을 안정적으로 운영할 수 있습니다."

면접관: "좋은 접근입니다. 그런데 만약 특정 소환사의 데이터 요청이 갑자기 많이 들어오면 어떻게 처리하실 건가요? 예를 들어 유명 스트리머가 방송을 시작했다거나 하는 경우요."

저: "아, 네. 그런 경우를 대비한 추가적인 최적화가 필요할 것 같습니다. 다음과 같이 구현하면 어떨까요?"

```java
@Service
public class HotDataManager {
    private final RedisTemplate<String, String> redisTemplate;
    private final RateLimitManager rateLimitManager;
    private final LoadingCache<String, Integer> requestCounter;

    public HotDataManager() {
        this.requestCounter = Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofMinutes(5))
            .build(key -> 0);
    }

    @Async
    public void handleRequest(String summonerName) {
        // 요청 빈도 카운팅
        int count = requestCounter.get(summonerName);
        requestCounter.put(summonerName, count + 1);

        // 핫 데이터 감지 (5분간 100회 이상 요청)
        if (count > 100) {
            promoteToHotData(summonerName);
        }
    }

    private void promoteToHotData(String summonerName) {
        String hotKey = "hot-summoner:" + summonerName;
        
        // 이미 핫 데이터로 등록되어 있는지 확인
        if (Boolean.FALSE.equals(redisTemplate.hasKey(hotKey))) {
            // 1. 데이터 미리 수집
            CompletableFuture.runAsync(() -> {
                // 기본 정보 수집
                SummonerDTO summoner = rateLimitManager
                    .executeWithRateLimit("summonerByName",
                        () -> riotClient.getSummonerByName(summonerName))
                    .join();

                // 최근 매치 정보 수집
                List<String> matchIds = rateLimitManager
                    .executeWithRateLimit("matchHistory",
                        () -> riotClient.getMatchList(summoner.getPuuid()))
                    .join();

                // 데이터를 Redis에 캐싱 (30분간 유지)
                redisTemplate.opsForValue().set(
                    "summoner:" + summonerName,
                    objectMapper.writeValueAsString(summoner),
                    Duration.ofMinutes(30)
                );

                redisTemplate.opsForValue().set(
                    "matches:" + summonerName,
                    objectMapper.writeValueAsString(matchIds),
                    Duration.ofMinutes(30)
                );

                // 핫 데이터 표시 (30분간 유지)
                redisTemplate.opsForValue().set(
                    hotKey,
                    "true",
                    Duration.ofMinutes(30)
                );
            });

            // 2. 실시간 업데이트 워커 시작
            startRealtimeUpdateWorker(summonerName);
        }
    }

    @Scheduled(fixedRate = 60000) // 1분마다 실행
    private void startRealtimeUpdateWorker(String summonerName) {
        String hotKey = "hot-summoner:" + summonerName;
        
        // 아직 핫 데이터인 경우에만 업데이트
        if (Boolean.TRUE.equals(redisTemplate.hasKey(hotKey))) {
            // 새로운 매치 확인 및 업데이트
            updateLatestMatches(summonerName);
        }
    }
}

```

저: "이렇게 구현하면 다음과 같은 이점이 있습니다:

1. 인기 소환사 자동 감지
    - 5분간 100회 이상 요청되면 핫 데이터로 등록
    - 데이터를 선제적으로 수집하고 캐싱

2. 실시간 업데이트
    - 핫 데이터로 등록된 소환사는 1분마다 자동 업데이트
    - 새로운 매치 정보 즉시 반영

3. 자동 최적화
    - 트래픽이 줄어들면 자동으로 일반 모드로 전환
    - 리소스 효율적 사용

이렇게 하면 갑작스러운 트래픽 증가에도 안정적으로 서비스할 수 있습니다."

면접관: "실제 서비스 운영 관점에서 모니터링이나 장애 대응은 어떻게 하실 건가요?"

저: "서비스 운영 관점에서 모니터링과 장애 대응을 위한 시스템을 구현해보겠습니다:"

```java
@Configuration
public class MonitoringConfig {
    
    @Service
    @Slf4j
    public class ServiceMonitor {
        private final MeterRegistry registry;
        private final AlertService alertService;
        private final CircuitBreaker riotApiCircuitBreaker;

        public ServiceMonitor(MeterRegistry registry, AlertService alertService) {
            this.registry = registry;
            this.alertService = alertService;
            
            // Circuit Breaker 설정
            this.riotApiCircuitBreaker = CircuitBreaker.builder("riotApi")
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .slidingWindowSize(100)
                .build();
        }

        @Around("@annotation(RiotApiCall)")
        public Object monitorRiotApi(ProceedingJoinPoint joinPoint) throws Throwable {
            Timer.Sample sample = Timer.start(registry);
            String endpoint = joinPoint.getSignature().getName();
            
            try {
                // Circuit Breaker로 API 호출 래핑
                return riotApiCircuitBreaker.executeSupplier(() -> {
                    try {
                        return joinPoint.proceed();
                    } catch (Throwable t) {
                        throw new RuntimeException(t);
                    }
                });
                
            } catch (Exception e) {
                // 에러 메트릭 기록
                registry.counter("riot.api.error", 
                    "endpoint", endpoint,
                    "error", e.getClass().getSimpleName())
                    .increment();
                
                // 심각한 에러 알림
                if (isCriticalError(e)) {
                    alertService.sendAlert(new Alert(
                        AlertLevel.HIGH,
                        "Riot API Critical Error",
                        String.format("Endpoint: %s, Error: %s", endpoint, e.getMessage())
                    ));
                }
                
                throw e;
                
            } finally {
                // 응답 시간 기록
                sample.stop(registry.timer("riot.api.latency", 
                    "endpoint", endpoint));
            }
        }

        @Scheduled(fixedRate = 60000)
        public void checkSystemHealth() {
            // 1. Rate Limit 사용량 체크
            checkRateLimitUsage();
            
            // 2. Redis 연결 상태 체크
            checkRedisHealth();
            
            // 3. 데이터 처리 지연 체크
            checkProcessingDelay();
        }

        private void checkRateLimitUsage() {
            Map<String, Double> usageRates = rateLimitManager.getCurrentUsage();
            
            for (Map.Entry<String, Double> entry : usageRates.entrySet()) {
                // 사용량 메트릭 기록
                registry.gauge("riot.api.rate.limit", 
                    Tags.of("endpoint", entry.getKey()), 
                    entry.getValue());
                
                // 임계치(80%) 초과시 알림
                if (entry.getValue() > 0.8) {
                    alertService.sendAlert(new Alert(
                        AlertLevel.MEDIUM,
                        "High Rate Limit Usage",
                        String.format("Endpoint %s usage: %.2f%%", 
                            entry.getKey(), 
                            entry.getValue() * 100)
                    ));
                }
            }
        }

        private void checkProcessingDelay() {
            // 큐잉된 요청 처리 지연 체크
            for (String endpoint : rateLimitManager.getEndpoints()) {
                String queueKey = "riot-api-queue:" + endpoint;
                Long queueSize = redisTemplate.opsForList().size(queueKey);
                
                if (queueSize > 1000) {  // 큐 크기가 1000 초과시
                    alertService.sendAlert(new Alert(
                        AlertLevel.HIGH,
                        "Large Request Queue",
                        String.format("Endpoint %s queue size: %d", 
                            endpoint, 
                            queueSize)
                    ));
                }
            }
        }
    }
}

```

저: "이렇게 모니터링 시스템을 구현하면 다음과 같은 이점이 있습니다:

1. 실시간 메트릭 수집
    - API 응답 시간
    - 에러율
    - Rate Limit 사용량
    - 큐 크기

2. 자동 장애 대응
    - Circuit Breaker로 API 장애 시 자동 차단
    - Rate Limit 초과 예방을 위한 조기 경보
    - 큐 처리 지연 모니터링

3. 알림 시스템
    - 심각도별 알림 레벨 구분
    - 즉각적인 운영팀 알림
    - 장애 상황별 대응 가이드

추가로 로깅 시스템도 구현하면 좋을 것 같은데요. 특히 장애 추적을 위한 로깅이 중요할 것 같습니다."

면접관: "네, 로깅 시스템도 중요하죠. 특히 분산 환경에서는 로그 추적이 더 까다로울 텐데, 어떻게 구현하실 건가요?"

저: "분산 환경에서의 로그 추적을 위해 다음과 같이 구현하겠습니다:"

```java
@Configuration
@EnableAsync
public class LoggingConfig {
    
    @Bean
    public Tracer tracer() {
        return Tracing.newBuilder()
            .localServiceName("lol-history-service")
            .spanReporter(spans -> {
                // ELK 스택으로 전송
                spans.forEach(span -> {
                    logSpanToElastic(span);
                });
            })
            .build()
            .tracer();
    }
    
    @Aspect
    @Component
    public class RequestTraceAspect {
        private final Tracer tracer;
        
        @Around("@annotation(Traced)")
        public Object traceMethod(ProceedingJoinPoint joinPoint) throws Throwable {
            String operationName = joinPoint.getSignature().getName();
            Span span = tracer.newTrace().name(operationName).start();
            
            try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
                // 메소드 파라미터 로깅
                Arrays.stream(joinPoint.getArgs())
                    .forEach(arg -> span.tag("param", arg.toString()));
                
                Object result = joinPoint.proceed();
                
                // 성공 결과 로깅
                span.tag("status", "success");
                return result;
                
            } catch (Exception e) {
                // 실패 정보 로깅
                span.tag("status", "error");
                span.tag("error.message", e.getMessage());
                span.tag("error.type", e.getClass().getName());
                
                throw e;
                
            } finally {
                span.finish();
            }
        }
    }

    private void logSpanToElastic(Span span) {
        Map<String, Object> logData = new HashMap<>();
        logData.put("traceId", span.context().traceId());
        logData.put("spanId", span.context().spanId());
        logData.put("operation", span.name());
        logData.put("timestamp", System.currentTimeMillis());
        logData.put("tags", span.tags());
        
        // Elasticsearch에 로그 저장
        elasticsearchClient.index(i -> i
            .index("service-logs-" + 
                LocalDate.now().format(DateTimeFormatter.ISO_DATE))
            .document(logData)
        );
    }
}

```

저: "이렇게 구현하면 분산 환경에서도 요청 추적이 가능합니다:

1. 분산 트레이싱
    - 요청별 고유 TraceID 발급
    - 서비스간 요청 추적
    - 전체 처리 흐름 시각화

2. 구조화된 로깅
    - JSON 형태로 모든 로그 저장
    - 메서드 파라미터와 결과 추적
    - 에러 상황 상세 기록

3. 로그 분석 용이성
    - Elasticsearch를 통한 빠른 검색
    - Kibana 대시보드로 시각화
    - 실시간 로그 모니터링

이를 통해 장애 발생시 빠른 원인 파악과 해결이 가능합니다."

면접관: "지금까지 설계한 시스템의 확장성은 어떻게 보장할 수 있을까요? 특히 사용자가 급증하는 경우에 대해서요."

저: "시스템 확장성을 위해 다음과 같이 설계하고 구현하겠습니다:"

```java
@Configuration
public class ScalabilityConfig {
    
    @Service
    public class LoadBalancedRiotClient {
        private final List<RiotApiClient> apiClients;
        private final LoadBalancer loadBalancer;
        
        public LoadBalancedRiotClient(
            @Value("${riot.api.keys}") List<String> apiKeys) {
            
            // 여러 API 키로 클라이언트 풀 구성
            this.apiClients = apiKeys.stream()
                .map(key -> new RiotApiClient(key))
                .collect(Collectors.toList());
                
            // 라운드 로빈 로드밸런서 구성
            this.loadBalancer = LoadBalancer.roundRobin(apiClients);
        }

        public CompletableFuture<SummonerDTO> getSummoner(String name) {
            return loadBalancer.execute(client -> 
                client.getSummonerByName(name));
        }
    }

    @Configuration
    public class ShardingConfig {
        @Bean
        public ShardingDataSource shardingDataSource() {
            // 매치 데이터 샤딩 규칙
            TableRuleConfiguration matchRule = new TableRuleConfiguration("matches");
            matchRule.setTableShardingStrategy(
                new StandardShardingStrategy(
                    "match_id",
                    new MatchIdShardingAlgorithm()
                )
            );

            // 통계 데이터 샤딩 규칙
            TableRuleConfiguration statsRule = new TableRuleConfiguration("summoner_stats");
            statsRule.setTableShardingStrategy(
                new StandardShardingStrategy(
                    "summoner_id",
                    new SummonerIdShardingAlgorithm()
                )
            );

            return ShardingDataSourceFactory.createDataSource(
                createDataSourceMap(),
                Arrays.asList(matchRule, statsRule)
            );
        }
    }

    @Service
    public class AutoScalingManager {
        private final KubernetesClient kubernetesClient;
        private final MetricsService metricsService;

        @Scheduled(fixedRate = 60000) // 1분마다 체크
        public void checkScaling() {
            // CPU 사용량 체크
            double cpuUsage = metricsService.getAverageCpuUsage();
            if (cpuUsage > 70) {  // 70% 이상시 스케일 아웃
                scaleOut();
            } else if (cpuUsage < 30) {  // 30% 이하시 스케일 인
                scaleIn();
            }

            // 메모리 사용량 체크
            double memoryUsage = metricsService.getAverageMemoryUsage();
            if (memoryUsage > 80) {  // 80% 이상시 스케일 아웃
                scaleOut();
            }

            // Request Queue 크기 체크
            long queueSize = metricsService.getTotalQueueSize();
            if (queueSize > 10000) {  // 큐 크기가 10000 이상시 스케일 아웃
                scaleOut();
            }
        }

        private void scaleOut() {
            // 현재 레플리카 수 확인
            int currentReplicas = getCurrentReplicas();
            int newReplicas = Math.min(currentReplicas * 2, MAX_REPLICAS);

            // 스케일 아웃
            kubernetesClient.apps()
                .deployments()
                .inNamespace("lol-history")
                .withName("api-service")
                .scale(newReplicas);

            // 캐시 노드 확장
            scaleCache(newReplicas);
        }

        private void scaleCache(int replicas) {
            // Redis 클러스터 노드 추가
            RedisClusterCommand command = new RedisClusterCommand(
                redisConfig.getClusterNodes());
            command.addShards(calculateRequiredShards(replicas));
        }
    }
}

```

저: "확장성을 위해 다음과 같은 전략을 구현했습니다:

1. API 키 관리와 로드 밸런싱
    - 여러 API 키를 활용한 처리량 증가
    - 라운드 로빈 방식의 로드 밸런싱
    - API 키별 독립적인 Rate Limit 관리

2. 데이터 샤딩
    - 매치 데이터: 매치 ID 기반 샤딩
    - 통계 데이터: 소환사 ID 기반 샤딩
    - 지역별 데이터 분리

3. 자동 스케일링
    - CPU/메모리 사용량 모니터링
    - 요청 큐 크기 기반 스케일링
    - 캐시 노드 자동 확장

또한 성능 최적화를 위해 다음과 같은 추가 구현도 필요합니다:"

```java
@Service
public class PerformanceOptimizer {
    private final CacheManager cacheManager;
    private final LoadingCache<String, CompletableFuture<MatchData>> matchDataLoader;

    public PerformanceOptimizer() {
        this.matchDataLoader = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .build(key -> CompletableFuture.supplyAsync(() ->
                loadMatchData(key)));
    }

    @Cacheable(
        value = "summoner-stats",
        key = "#summonerId",
        condition = "#summonerId != null",
        unless = "#result == null"
    )
    public SummonerStats getStats(String summonerId) {
        return statsRepository.findById(summonerId)
            .orElseGet(() -> calculateStats(summonerId));
    }

    private MatchData loadMatchData(String matchId) {
        // 캐시 계층 구조 구현
        // L1: Local Cache (Caffeine)
        // L2: Redis
        // L3: Database
        
        return cacheManager.getOrCompute(
            matchId,
            () -> matchRepository.findById(matchId),
            CacheLevel.THREE_LEVEL
        );
    }

    // 배치 처리로 통계 미리 계산
    @Scheduled(cron = "0 0 * * * *") // 매시간 실행
    public void preCalculateStats() {
        List<String> activeSummoners = getActiveSummoners();
        
        CompletableFuture.allOf(
            activeSummoners.stream()
                .map(this::calculateStatsAsync)
                .toArray(CompletableFuture[]::new)
        ).join();
    }
}

```

저: "이런 최적화를 통해:

1. 다중 캐시 계층
    - Local Cache (In-Memory)
    - Redis Cache (분산 캐시)
    - Database (영구 저장소)

2. 배치 처리
    - 통계 사전 계산
    - 인기 소환사 데이터 미리 로드
    - 피크 타임 대비

3. 비동기 처리
    - 요청 병렬 처리
    - Non-blocking I/O
    - 리소스 효율적 사용

이렇게 구현하면 급격한 사용자 증가에도 안정적인 서비스가 가능합니다."

면접관: "좋은 설계네요. 마지막으로 이 시스템의 배포 전략과 무중단 배포는 어떻게 구현하실 건가요?"

저: "무중단 배포를 위한 전략을 다음과 같이 구현하겠습니다:"

```java
@Configuration
public class DeploymentConfig {
    
    @Service
    public class DeploymentManager {
        private final KubernetesClient kubernetesClient;
        private final HealthCheckService healthCheckService;
        private final MetricsService metricsService;

        public void performBlueGreenDeployment(String newVersion) {
            try {
                // 1. 새로운 버전(Green) 배포
                deployNewVersion(newVersion);
                
                // 2. 헬스체크 수행
                if (isNewVersionHealthy(newVersion)) {
                    // 3. 트래픽 전환
                    switchTraffic(newVersion);
                    
                    // 4. 이전 버전(Blue) 제거
                    removeOldVersion();
                } else {
                    // 배포 실패시 롤백
                    rollback();
                }
            } catch (Exception e) {
                performRollback(e);
            }
        }

        private void deployNewVersion(String version) {
            // 새로운 버전 배포 설정
            Deployment greenDeployment = new DeploymentBuilder()
                .withNewMetadata()
                    .withName("lol-history-" + version)
                    .addToLabels("version", version)
                    .addToLabels("environment", "green")
                .endMetadata()
                .withNewSpec()
                    .withReplicas(0) // 초기에는 0개 레플리카로 시작
                    .withNewTemplate()
                        .withNewSpec()
                            .addNewContainer()
                                .withName("lol-history")
                                .withImage("lol-history:" + version)
                                .withReadinessProbe(createReadinessProbe())
                                .withLivenessProbe(createLivenessProbe())
                            .endContainer()
                        .endSpec()
                    .endTemplate()
                .endSpec()
                .build();

            // 점진적으로 레플리카 수 증가
            kubernetesClient.apps()
                .deployments()
                .inNamespace("lol-history")
                .create(greenDeployment);

            // 단계적으로 레플리카 증가
            scaleDeployment(version, 1);  // 25%
            waitForStability();
            scaleDeployment(version, 2);  // 50%
            waitForStability();
            scaleDeployment(version, 4);  // 100%
        }

        private Probe createReadinessProbe() {
            return new ProbeBuilder()
                .withHttpGet(new HTTPGetActionBuilder()
                    .withPath("/actuator/health/readiness")
                    .withPort(new IntOrString(8080))
                    .build())
                .withInitialDelaySeconds(10)
                .withPeriodSeconds(5)
                .withFailureThreshold(3)
                .build();
        }

        private boolean isNewVersionHealthy(String version) {
            // 1. 기본 헬스체크
            boolean basicHealth = healthCheckService
                .checkHealth("lol-history-" + version);

            if (!basicHealth) return false;

            // 2. 메트릭 검증
            MetricsValidation metricsValidation = metricsService
                .validateDeployment(version);

            return metricsValidation.isSuccessful() &&
                   metricsValidation.getErrorRate() < 0.1 &&  // 에러율 10% 미만
                   metricsValidation.getResponseTime() < 500;  // 응답시간 500ms 미만
        }

        private void switchTraffic(String version) {
            // 서비스 설정 업데이트
            Service service = kubernetesClient.services()
                .inNamespace("lol-history")
                .withName("lol-history-service")
                .edit(s -> new ServiceBuilder(s)
                    .editSpec()
                        .withSelector(Map.of(
                            "app", "lol-history",
                            "version", version
                        ))
                    .endSpec()
                    .build()
                );

            // 트래픽 전환 모니터링
            monitorTrafficSwitch(version);
        }

        private void monitorTrafficSwitch(String version) {
            ScheduledExecutorService executor = 
                Executors.newSingleThreadScheduledExecutor();

            executor.scheduleAtFixedRate(() -> {
                MetricsValidation validation = 
                    metricsService.validateDeployment(version);

                if (!validation.isSuccessful()) {
                    rollback();
                }
            }, 0, 1, TimeUnit.MINUTES);
        }
    }

    @Service
    public class SessionDrainManager {
        private final RedisTemplate<String, String> redisTemplate;

        public void drainSessions(String oldVersion) {
            // 1. 새로운 세션 생성 중단
            markForDraining(oldVersion);

            // 2. 기존 세션 만료 대기
            waitForSessionDrain(oldVersion);
        }

        private void markForDraining(String version) {
            String drainKey = "drain:version:" + version;
            redisTemplate.opsForValue().set(drainKey, "true");
        }

        private void waitForSessionDrain(String version) {
            while (hasActiveSessions(version)) {
                Thread.sleep(1000);
            }
        }
    }
}

```

저: "무중단 배포를 위해 Blue-Green 배포 전략을 구현했습니다. 주요 특징은:

1. 점진적 배포
    - 새 버전을 단계적으로 확장
    - 각 단계마다 안정성 검증
    - 문제 발생 시 즉시 롤백

2. 철저한 헬스체크
    - 기본적인 서비스 상태 확인
    - 메트릭 기반 검증 (에러율, 응답시간)
    - 지속적인 모니터링

3. 세션 관리
    - 기존 사용자 세션 유지
    - 점진적인 세션 드레이닝
    - 무중단 전환

추가로 다음과 같은 배포 관련 구현도 필요합니다:"

```yaml
# GitLab CI/CD Pipeline
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_REGISTRY: "registry.example.com"
  APP_NAME: "lol-history"

test:
  stage: test
  script:
    - ./gradlew test
    - ./gradlew integrationTest
    - sonar-scanner
  artifacts:
    reports:
      junit: build/test-results/test/*.xml
      coverage: build/reports/jacoco/test/jacocoTestReport.xml

build:
  stage: build
  script:
    - ./gradlew build
    - docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${CI_COMMIT_SHA} .
    - docker push ${DOCKER_REGISTRY}/${APP_NAME}:${CI_COMMIT_SHA}
  only:
    - main
    - /^release-.*$/

deploy_staging:
  stage: deploy
  script:
    - kubectl config use-context staging
    - helm upgrade --install ${APP_NAME} ./helm \
        --set image.tag=${CI_COMMIT_SHA} \
        --set environment=staging
  environment:
    name: staging
  only:
    - main

deploy_production:
  stage: deploy
  script:
    - kubectl config use-context production
    - |
      if [ "$(kubectl get deploy -l app=${APP_NAME} --no-headers | wc -l)" -eq 0 ]; then
        # 최초 배포
        helm install ${APP_NAME} ./helm \
          --set image.tag=${CI_COMMIT_SHA} \
          --set environment=production
      else
        # Blue-Green 배포
        ./scripts/blue-green-deploy.sh \
          --app ${APP_NAME} \
          --version ${CI_COMMIT_SHA}
      fi
  environment:
    name: production
  when: manual
  only:
    - /^release-.*$/

```

저: "이렇게 구현하면:

1. 자동화된 CI/CD 파이프라인
    - 자동 테스트 실행
    - 코드 품질 검사
    - 자동 빌드 및 배포

2. 환경별 배포 전략
    - 스테이징 환경: 자동 배포
    - 프로덕션 환경: 수동 승인 후 배포
    - Blue-Green 배포 자동화

3. 안전한 배포 프로세스
    - 테스트 커버리지 검증
    - 단계별 검증
    - 롤백 자동화

이를 통해 안정적이고 신뢰성 있는 배포가 가능합니다."

면접관: "전체적으로 잘 설계되었네요. 실제 운영 중에 발생할 수 있는 장애 상황에 대한 시나리오와 대응 계획도 있으신가요?"

저: "네, 주요 장애 시나리오와 대응 계획을 설명드리겠습니다:"

```java
@Configuration
public class IncidentResponseConfig {
    
    @Service
    public class IncidentManager {
        private final AlertService alertService;
        private final MetricsService metricsService;
        private final CircuitBreakerRegistry circuitBreakerRegistry;
        
        // 장애 상황별 대응 전략
        private final Map<IncidentType, IncidentStrategy> strategies = Map.of(
            IncidentType.RIOT_API_ERROR, new RiotApiErrorStrategy(),
            IncidentType.HIGH_LATENCY, new HighLatencyStrategy(),
            IncidentType.DATA_INCONSISTENCY, new DataInconsistencyStrategy(),
            IncidentType.MEMORY_LEAK, new MemoryLeakStrategy(),
            IncidentType.DATABASE_CONNECTION, new DatabaseConnectionStrategy()
        );

        @Scheduled(fixedRate = 30000) // 30초마다 체크
        public void monitorSystem() {
            // 시스템 상태 체크
            SystemHealth health = metricsService.checkSystemHealth();
            
            if (!health.isHealthy()) {
                handleIncident(health.getIncidentType(), health.getDetails());
            }
        }

        public void handleIncident(IncidentType type, Map<String, Object> details) {
            // 1. 장애 로깅
            logIncident(type, details);
            
            // 2. 알림 발송
            alertService.sendIncidentAlert(type, details);
            
            // 3. 대응 전략 실행
            IncidentStrategy strategy = strategies.get(type);
            strategy.handle(details);
        }
    }

    // Riot API 장애 대응 전략
    @Component
    public class RiotApiErrorStrategy implements IncidentStrategy {
        private final CircuitBreaker circuitBreaker;
        private final CacheManager cacheManager;

        @Override
        public void handle(Map<String, Object> details) {
            // 1. Circuit Breaker 동작
            circuitBreaker.transitionToOpenState();
            
            // 2. 캐시 TTL 연장
            cacheManager.extendExpiration("summoner-data", Duration.ofHours(2));
            
            // 3. 백업 데이터 활성화
            activateBackupData();
            
            // 4. 사용자에게 알림
            notifyUsers("일시적인 데이터 지연이 발생했습니다.");
        }
    }

    // 높은 지연시간 대응 전략
    @Component
    public class HighLatencyStrategy implements IncidentStrategy {
        private final LoadBalancer loadBalancer;
        private final CacheManager cacheManager;

        @Override
        public void handle(Map<String, Object> details) {
            // 1. 캐시 계층 최적화
            optimizeCacheLayers();
            
            // 2. 불필요한 요청 필터링
            enableRequestFiltering();
            
            // 3. 자동 스케일링 트리거
            triggerAutoScaling();
        }

        private void optimizeCacheLayers() {
            // 캐시 히트율 개선
            cacheManager.adjustCacheSize("hot-data", calculateOptimalSize());
            cacheManager.preloadFrequentlyAccessedData();
        }
    }

    // 데이터 불일치 대응 전략
    @Component
    public class DataInconsistencyStrategy implements IncidentStrategy {
        private final DataConsistencyChecker consistencyChecker;
        private final DataRepairService repairService;

        @Override
        public void handle(Map<String, Object> details) {
            // 1. 불일치 데이터 식별
            List<InconsistentData> inconsistencies = 
                consistencyChecker.findInconsistencies();
            
            // 2. 자동 복구 시도
            for (InconsistentData data : inconsistencies) {
                if (data.isAutoRepairable()) {
                    repairService.repair(data);
                } else {
                    // 수동 개입 필요 알림
                    alertService.sendManualInterventionRequired(data);
                }
            }
        }
    }

    // 메모리 누수 대응 전략
    @Component
    public class MemoryLeakStrategy implements IncidentStrategy {
        private final MemoryAnalyzer memoryAnalyzer;
        private final ResourceManager resourceManager;

        @Override
        public void handle(Map<String, Object> details) {
            // 1. 메모리 덤프 생성
            HeapDump heapDump = memoryAnalyzer.createHeapDump();
            
            // 2. 누수 지점 분석
            List<LeakPoint> leakPoints = memoryAnalyzer.analyzeLeaks(heapDump);
            
            // 3. 자동 복구 시도
            for (LeakPoint leak : leakPoints) {
                if (leak.isRecoverable()) {
                    resourceManager.cleanupResources(leak);
                }
            }
            
            // 4. 필요시 서비스 재시작
            if (isRestartRequired()) {
                performRollingRestart();
            }
        }
    }
}

```

저: "주요 장애 시나리오별 대응 계획을 다음과 같이 준비했습니다:

1. Riot API 장애 상황
    - Circuit Breaker 패턴으로 장애 전파 방지
    - 캐시 TTL 연장으로 서비스 지속
    - 백업 데이터 활용
    - 사용자 커뮤니케이션

2. 높은 지연시간 발생
    - 캐시 계층 최적화
    - 불필요한 요청 필터링
    - 자동 스케일링 트리거
    - 로드밸런싱 조정

3. 데이터 불일치 발생
    - 자동 데이터 정합성 체크
    - 자동 복구 프로세스
    - 수동 개입 필요시 알림
    - 복구 이력 관리

4. 메모리 누수 발생
    - 자동 메모리 덤프 및 분석
    - 리소스 자동 정리
    - 필요시 롤링 재시작
    - 근본 원인 분석

추가로 장애 대응을 위한 모니터링 대시보드도 구현했습니다:"

```tsx
import React, { useState, useEffect } from 'react';
import { LineChart, XAxis, YAxis, Tooltip, Line, CartesianGrid } from 'recharts';
import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert';

const MonitoringDashboard = () => {
  const [metrics, setMetrics] = useState({
    apiLatency: [],
    errorRates: [],
    memoryUsage: [],
    activeRequests: []
  });
  
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    // 메트릭 데이터 주기적 업데이트
    const interval = setInterval(() => {
      fetchMetrics();
    }, 5000);

    return () => clearInterval(interval);
  }, []);

  return (
    <div className="p-4">
      {/* 알림 섹션 */}
      <div className="mb-6">
        {alerts.map((alert, index) => (
          <Alert key={index} variant={alert.severity}>
            <AlertTitle>{alert.title}</AlertTitle>
            <AlertDescription>{alert.description}</AlertDescription>
          </Alert>
        ))}
      </div>

      {/* 메트릭 차트 */}
      <div className="grid grid-cols-2 gap-4">
        {/* API 지연시간 차트 */}
        <div className="p-4 bg-white rounded-lg shadow">
          <h3 className="text-lg font-semibold mb-4">API Latency</h3>
          <LineChart width={500} height={300} data={metrics.apiLatency}>
            <XAxis dataKey="timestamp" />
            <YAxis />
            <Tooltip />
            <CartesianGrid stroke="#eee" />
            <Line type="monotone" dataKey="value" stroke="#8884d8" />
          </LineChart>
        </div>

        {/* 에러율 차트 */}
        <div className="p-4 bg-white rounded-lg shadow">
          <h3 className="text-lg font-semibold mb-4">Error Rates</h3>
          <LineChart width={500} height={300} data={metrics.errorRates}>
            <XAxis dataKey="timestamp" />
            <YAxis />
            <Tooltip />
            <CartesianGrid stroke="#eee" />
            <Line type="monotone" dataKey="value" stroke="#82ca9d" />
          </LineChart>
        </div>

        {/* 메모리 사용량 차트 */}
        <div className="p-4 bg-white rounded-lg shadow">
          <h3 className="text-lg font-semibold mb-4">Memory Usage</h3>
          <LineChart width={500} height={300} data={metrics.memoryUsage}>
            <XAxis dataKey="timestamp" />
            <YAxis />
            <Tooltip />
            <CartesianGrid stroke="#eee" />
            <Line type="monotone" dataKey="value" stroke="#ffc658" />
          </LineChart>
        </div>

        {/* 활성 요청 차트 */}
        <div className="p-4 bg-white rounded-lg shadow">
          <h3 className="text-lg font-semibold mb-4">Active Requests</h3>
          <LineChart width={500} height={300} data={metrics.activeRequests}>
            <XAxis dataKey="timestamp" />
            <YAxis />
            <Tooltip />
            <CartesianGrid stroke="#eee" />
            <Line type="monotone" dataKey="value" stroke="#ff7300" />
          </LineChart>
        </div>
      </div>
    </div>
  );
};

export default MonitoringDashboard;

```

저: "이런 방식으로 장애 상황을 실시간으로 모니터링하고 자동으로 대응할 수 있습니다. 특히 중요한 것은 장애 발생 시 빠른 감지와 자동화된 초기 대응, 그리고 명확한 에스컬레이션 프로세스입니다."

면접관: "좋은 설계네요. 실제 구현하신다면 어떤 부분에서 가장 어려움이 있을 것 같으신가요? 또 어떻게 해결하실 건가요?"

저: "실제 구현 시 가장 큰 도전 과제는 크게 세 가지라고 생각합니다:

1. Rate Limit 관리와 데이터 일관성
    - Riot API의 Rate Limit을 지키면서도 실시간성을 보장해야 함
    - 여러 서버에서 동시에 API를 호출할 때 Rate Limit을 정확히 관리해야 함
    - 분산 환경에서 데이터 정합성 유지가 필요

이를 위한 구체적인 해결 방안을 보여드리겠습니다:"

```java
@Service
@Slf4j
public class DistributedRateLimitManager {
    private final RedisTemplate<String, String> redisTemplate;
    private final RedissonClient redissonClient;
    
    @Value("${riot.api.key}")
    private String apiKey;

    public <T> T executeWithDistributedRateLimit(
            String method, 
            Supplier<T> apiCall) throws RateLimitException {
            
        String rateLimitKey = "ratelimit:" + apiKey + ":" + method;
        RLock lock = redissonClient.getLock("ratelock:" + method);
        
        try {
            // 분산 락 획득 (최대 1초 대기)
            if (!lock.tryLock(1, TimeUnit.SECONDS)) {
                throw new RateLimitException("Failed to acquire rate limit lock");
            }
            
            // 현재 요청 수 확인
            String currentCount = redisTemplate.opsForValue().get(rateLimitKey);
            int count = currentCount != null ? Integer.parseInt(currentCount) : 0;
            
            // Rate Limit 설정값 (메소드별)
            RateLimitConfig config = getRateLimitConfig(method);
            
            if (count >= config.getMaxRequests()) {
                // 큐잉 시스템으로 전환
                return queueRequest(method, apiCall);
            }
            
            // 요청 카운트 증가 (TTL 설정)
            redisTemplate.opsForValue().increment(rateLimitKey);
            redisTemplate.expire(rateLimitKey, 
                config.getTimeWindow(), 
                TimeUnit.SECONDS);
            
            // API 호출 실행
            return apiCall.get();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RateLimitException("Rate limit check interrupted");
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

    private <T> T queueRequest(String method, Supplier<T> apiCall) {
        // 우선순위 큐에 요청 추가
        String queueKey = "request-queue:" + method;
        QueuedRequest<T> request = new QueuedRequest<>(
            apiCall,
            calculatePriority(method)
        );
        
        redisTemplate.opsForZSet().add(
            queueKey,
            objectMapper.writeValueAsString(request),
            request.getPriority()
        );
        
        // 비동기 처리 결과 반환
        return handleQueuedRequest(request);
    }

    private double calculatePriority(String method) {
        // 우선순위 계산 로직
        // 1. 실시간 사용자 요청 (높은 우선순위)
        // 2. 최근 게임 데이터 수집
        // 3. 통계 데이터 업데이트 (낮은 우선순위)
        switch (method) {
            case "summonerByName":
                return 100.0;
            case "matchHistory":
                return 50.0;
            default:
                return 10.0;
        }
    }

    @Scheduled(fixedRate = 100) // 100ms마다 실행
    public void processQueue() {
        for (String method : getRateLimitMethods()) {
            String queueKey = "request-queue:" + method;
            
            // 우선순위가 가장 높은 요청 처리
            Set<String> requests = redisTemplate.opsForZSet()
                .rangeByScore(queueKey, 
                    Double.NEGATIVE_INFINITY, 
                    Double.POSITIVE_INFINITY, 
                    0, 1);
                    
            if (!requests.isEmpty()) {
                QueuedRequest<?> request = objectMapper
                    .readValue(requests.iterator().next(), 
                        QueuedRequest.class);
                        
                try {
                    executeWithDistributedRateLimit(
                        method, 
                        request.getApiCall()
                    );
                    
                    // 성공한 요청 제거
                    redisTemplate.opsForZSet()
                        .remove(queueKey, 
                            request);
                            
                } catch (RateLimitException e) {
                    // 다음 주기에 재시도
                    break;
                }
            }
        }
    }
}

```

저: "두 번째 큰 도전 과제는:

2. 캐시 일관성과 성능 최적화
    - 여러 서버의 캐시 동기화
    - 캐시 갱신 시점 결정
    - 메모리 사용량 관리

이를 위한 해결 방안입니다:"

```java
@Service
public class DistributedCacheManager {
    private final RedisTemplate<String, String> redisTemplate;
    private final LoadingCache<String, CacheEntry> localCache;
    private final RedisPubSubListener cacheInvalidationListener;

    public DistributedCacheManager() {
        this.localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats()
            .build(key -> loadFromRedis(key));
            
        // 캐시 무효화 리스너 설정
        this.cacheInvalidationListener = new RedisPubSubListener() {
            @Override
            public void onMessage(String channel, String message) {
                if ("cache-invalidation".equals(channel)) {
                    InvalidationEvent event = objectMapper
                        .readValue(message, InvalidationEvent.class);
                    localCache.invalidate(event.getKey());
                }
            }
        };
    }

    public <T> T getWithCache(
            String key, 
            Supplier<T> dataLoader, 
            CachePolicy policy) {
            
        CacheEntry entry = localCache.get(key);
        
        if (entry != null && !isStale(entry, policy)) {
            return entry.getValue();
        }
        
        // 분산 락으로 동시 로드 방지
        RLock lock = redissonClient.getLock("cache-lock:" + key);
        
        try {
            if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
                // 데이터 로드 및 캐시 업데이트
                T value = dataLoader.get();
                updateCache(key, value, policy);
                return value;
            } else {
                // 락 획득 실패시 기존 데이터 반환
                return entry != null ? entry.getValue() : null;
            }
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

    private void updateCache(
            String key, 
            Object value, 
            CachePolicy policy) {
            
        // 로컬 캐시 업데이트
        localCache.put(key, new CacheEntry(value, System.currentTimeMillis()));
        
        // Redis 캐시 업데이트
        redisTemplate.opsForValue().set(
            "cache:" + key,
            objectMapper.writeValueAsString(value),
            policy.getTtl()
        );
        
        // 다른 서버에 캐시 업데이트 알림
        publishCacheInvalidation(key);
    }

    public void monitorCacheHealth() {
        // 캐시 상태 모니터링
        CacheStats stats = localCache.stats();
        
        // 히트율이 낮으면 캐시 워밍
        if (stats.hitRate() < 0.5) {
            warmupCache();
        }
        
        // 메모리 사용량 체크
        if (stats.evictionCount() > 1000) {
            // 캐시 크기 조정 필요
            adjustCacheSize();
        }
    }

    private void warmupCache() {
        // 자주 접근되는 데이터 미리 로드
        List<String> hotKeys = getHotKeys();
        for (String key : hotKeys) {
            localCache.get(key);
        }
    }

    private void adjustCacheSize() {
        // 메모리 사용량에 따라 캐시 크기 조정
        Runtime runtime = Runtime.getRuntime();
        long maxMemory = runtime.maxMemory();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        
        double memoryUsageRatio = (double) usedMemory / maxMemory;
        
        if (memoryUsageRatio > 0.8) {
            // 캐시 크기 축소
            reduceCacheSize();
        }
    }
}

```

저: "마지막 도전 과제는:

3. 대규모 트래픽 처리와 확장성
    - 갑작스러운 트래픽 증가 대응
    - 데이터 샤딩과 파티셔닝
    - 서비스 안정성 유지

이런 기술적 도전들을 해결하면서 가장 중요한 것은 점진적인 개선과 모니터링입니다. 각 해결책을 단계적으로 적용하고, 효과를 측정하면서 지속적으로 개선해 나가야 합니다.

특히 초기에는:
1. 기본적인 Rate Limit 관리와 캐싱
2. 단순한 형태의 분산 처리
3. 핵심 메트릭 모니터링

이렇게 시작하고, 서비스 규모가 커짐에 따라 점진적으로:
1. 더 정교한 분산 Rate Limit 관리
2. 다층 캐시 시스템
3. 상세한 모니터링과 자동화된 대응

이런 식으로 발전시켜 나가는 것이 현실적인 접근 방법이라고 생각합니다."
