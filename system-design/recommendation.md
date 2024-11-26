# 실시간 추천 시스템 설계

면접관: "대규모 사용자를 위한 실시간 추천 시스템을 설계해주세요. 사용자 행동에 기반한 개인화된 추천과 실시간 업데이트가 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 일일 활성 사용자 수와 추천 요청량은 어느 정도인가요?
2. 추천 결과의 실시간성이 얼마나 중요한가요?
3. 개인화 수준과 컨텍스트 고려 범위는 어떻게 되나요?
4. 콜드 스타트 문제 해결이 필요한가요?

면접관:
1. DAU 1000만, 초당 10,000개 추천 요청
2. 사용자 행동 반영은 5분 이내
3. 사용자 히스토리, 현재 컨텍스트, 인구통계학적 데이터 모두 고려
4. 새로운 사용자/아이템에 대한 콜드 스타트 해결 필요

## 1. 실시간 추천 엔진

```java
@Service
public class RealtimeRecommendationEngine {

    // 1. 실시간 특징 처리
    @Service
    public class FeatureProcessor {
        private final KafkaTemplate<String, UserEvent> kafkaTemplate;
        private final RedisTemplate<String, FeatureVector> redisTemplate;

        public void processUserEvent(UserEvent event) {
            // 실시간 특징 추출
            FeatureVector features = extractFeatures(event);
            
            // 특징 벡터 캐싱
            String featureKey = "features:" + event.getUserId();
            redisTemplate.opsForValue().set(
                featureKey, 
                features,
                Duration.ofHours(24)
            );

            // 특징 업데이트 이벤트 발행
            kafkaTemplate.send(
                "feature-updates", 
                event.getUserId(), 
                event
            );
        }

        private FeatureVector extractFeatures(UserEvent event) {
            return FeatureVector.builder()
                .userFeatures(extractUserFeatures(event))
                .contextFeatures(extractContextFeatures(event))
                .temporalFeatures(extractTemporalFeatures(event))
                .build();
        }
    }

    // 2. 실시간 모델 서빙
    @Service
    public class ModelServingService {
        private final LoadingCache<String, RecommendationModel> modelCache;
        private final FeatureService featureService;

        public List<Recommendation> getRecommendations(
            RecommendationRequest request) {
            
            // 사용자 특징 로드
            FeatureVector userFeatures = 
                featureService.getUserFeatures(request.getUserId());
            
            // 컨텍스트 특징 결합
            FeatureVector combinedFeatures = 
                combineFeatures(userFeatures, request.getContext());
            
            // 모델 추론
            RecommendationModel model = modelCache.get("latest");
            List<ScoredItem> scoredItems = 
                model.predict(combinedFeatures);
            
            // 후처리 및 필터링
            return postProcessResults(scoredItems, request);
        }

        private List<Recommendation> postProcessResults(
            List<ScoredItem> items, 
            RecommendationRequest request) {
            
            return items.stream()
                .filter(item -> passesBusinessRules(item, request))
                .filter(item -> !isRecentlyRecommended(
                    item, request.getUserId()))
                .sorted(Comparator.comparing(ScoredItem::getScore)
                    .reversed())
                .limit(request.getLimit())
                .map(this::toRecommendation)
                .collect(Collectors.toList());
        }
    }
}
```

## 2. 개인화 및 컨텍스트 처리 시스템

```java
@Service
public class PersonalizationService {

    // 1. 사용자 프로필 관리
    @Service
    public class UserProfileManager {
        private final RedisTemplate<String, UserProfile> redisTemplate;
        private final CassandraTemplate cassandraTemplate;

        @Data
        public class UserProfile {
            private String userId;
            private Map<String, Double> interests;    // 관심사-점수 매핑
            private Map<String, Integer> preferences; // 선호도 데이터
            private List<String> recentInteractions;  // 최근 상호작용
            private Map<String, Double> categoryAffinity; // 카테고리 친밀도
        }

        public UserProfile getOrCreateProfile(String userId) {
            String cacheKey = "profile:" + userId;
            
            // 캐시 확인
            UserProfile profile = redisTemplate.opsForValue()
                .get(cacheKey);
                
            if (profile == null) {
                // DB에서 로드
                profile = loadFromDB(userId);
                // 캐시에 저장
                redisTemplate.opsForValue().set(
                    cacheKey, 
                    profile,
                    Duration.ofHours(24)
                );
            }
            
            return profile;
        }

        @Async
        public void updateProfile(String userId, UserEvent event) {
            UserProfile profile = getOrCreateProfile(userId);
            
            // 프로필 업데이트
            switch (event.getType()) {
                case CLICK:
                    updateClickBehavior(profile, event);
                    break;
                case PURCHASE:
                    updatePurchaseBehavior(profile, event);
                    break;
                case RATING:
                    updateRatingBehavior(profile, event);
                    break;
            }
            
            // 변경사항 저장
            saveProfile(profile);
        }
    }

    // 2. 컨텍스트 처리
    @Service
    public class ContextProcessor {
        private final LocationService locationService;
        private final TimeBasedAnalyzer timeAnalyzer;

        public ContextVector processContext(RequestContext context) {
            return ContextVector.builder()
                .timeFeatures(extractTimeFeatures(context))
                .locationFeatures(extractLocationFeatures(context))
                .deviceFeatures(extractDeviceFeatures(context))
                .sessionFeatures(extractSessionFeatures(context))
                .build();
        }

        private Map<String, Double> extractTimeFeatures(
            RequestContext context) {
            
            LocalDateTime time = context.getTimestamp();
            return Map.of(
                "hour_of_day", normalizeHour(time.getHour()),
                "day_of_week", normalizeDayOfWeek(time.getDayOfWeek()),
                "is_weekend", isWeekend(time) ? 1.0 : 0.0,
                "is_holiday", timeAnalyzer.isHoliday(time) ? 1.0 : 0.0
            );
        }

        private Map<String, Double> extractLocationFeatures(
            RequestContext context) {
            
            Location location = context.getLocation();
            return Map.of(
                "latitude", location.getLatitude(),
                "longitude", location.getLongitude(),
                "location_type", 
                    locationService.getLocationType(location),
                "local_time_offset", 
                    calculateLocalTimeOffset(location)
            );
        }
    }

    // 3. 실시간 세션 분석
    @Service
    public class SessionAnalyzer {
        private final SessionCache sessionCache;
        private final BehaviorAnalyzer behaviorAnalyzer;

        public SessionInsights analyzeCurrentSession(
            String userId, 
            String sessionId) {
            
            // 현재 세션 데이터 로드
            Session session = sessionCache.getSession(sessionId);
            
            // 세션 행동 분석
            return SessionInsights.builder()
                .intentScore(
                    calculateIntentScore(session))
                .interestLevel(
                    calculateInterestLevel(session))
                .browsingPattern(
                    analyzeBrowsingPattern(session))
                .searchContext(
                    analyzeSearchContext(session))
                .build();
        }

        private double calculateIntentScore(Session session) {
            return behaviorAnalyzer.analyzeIntent(
                session.getEvents(),
                session.getDuration(),
                session.getDepth()
            );
        }

        private BrowsingPattern analyzeBrowsingPattern(
            Session session) {
            
            return BrowsingPattern.builder()
                .pageViews(session.getPageViews())
                .categoryTransitions(
                    analyzeCategoryTransitions(session))
                .dwellTimes(
                    analyzeDwellTimes(session))
                .searchRefinements(
                    analyzeSearchBehavior(session))
                .build();
        }
    }

    // 4. 개인화 규칙 엔진
    @Service
    public class PersonalizationRuleEngine {
        private final RuleRepository ruleRepository;
        private final ScoreAggregator scoreAggregator;

        public List<RecommendationRule> getApplicableRules(
            UserProfile profile, 
            ContextVector context) {
            
            return ruleRepository.findAll().stream()
                .filter(rule -> rule.matches(profile, context))
                .sorted(Comparator.comparing(RecommendationRule::getPriority)
                    .reversed())
                .collect(Collectors.toList());
        }

        public double applyRules(
            ScoredItem item, 
            List<RecommendationRule> rules) {
            
            double baseScore = item.getScore();
            
            // 규칙 기반 점수 조정
            for (RecommendationRule rule : rules) {
                baseScore = rule.adjustScore(
                    baseScore, 
                    item, 
                    context
                );
            }
            
            return baseScore;
        }
    }
}
```

이러한 개인화 및 컨텍스트 처리 시스템을 통해:

1. 사용자 프로필 관리
    - 실시간 프로필 업데이트
    - 다차원 선호도 추적
    - 효율적인 캐싱

2. 컨텍스트 처리
    - 시간적 컨텍스트
    - 위치 기반 컨텍스트
    - 디바이스 컨텍스트

3. 세션 분석
    - 실시간 행동 분석
    - 의도 파악
    - 패턴 인식

4. 규칙 기반 개인화
    - 동적 규칙 적용
    - 점수 조정
    - 우선순위 관리

을 구현할 수 있습니다.

면접관: 추천의 정확도와 다양성(Diversity) 사이의 균형은 어떻게 맞추시겠습니까?

## 3. 추천 다양성 관리 시스템

```java
@Service
public class RecommendationDiversityManager {

    // 1. 다양성 메트릭 계산
    @Service
    public class DiversityMetricsCalculator {
        private final CategoryService categoryService;
        private final SimilarityCalculator similarityCalculator;

        public DiversityMetrics calculateDiversity(List<Recommendation> recommendations) {
            return DiversityMetrics.builder()
                .categoryDiversity(calculateCategoryDiversity(recommendations))
                .contentDiversity(calculateContentDiversity(recommendations))
                .temporalDiversity(calculateTemporalDiversity(recommendations))
                .attributeDiversity(calculateAttributeDiversity(recommendations))
                .build();
        }

        private double calculateCategoryDiversity(List<Recommendation> items) {
            // 카테고리 분포 계산
            Map<String, Integer> categoryCount = items.stream()
                .map(item -> categoryService.getCategory(item.getItemId()))
                .collect(Collectors.groupingBy(
                    category -> category,
                    Collectors.summingInt(category -> 1)
                ));

            // 엔트로피 기반 다양성 점수 계산
            return calculateEntropy(categoryCount, items.size());
        }

        private double calculateContentDiversity(List<Recommendation> items) {
            // 아이템 간 평균 유사도 계산
            double totalSimilarity = 0;
            int comparisons = 0;

            for (int i = 0; i < items.size(); i++) {
                for (int j = i + 1; j < items.size(); j++) {
                    totalSimilarity += similarityCalculator.calculate(
                        items.get(i), items.get(j));
                    comparisons++;
                }
            }

            return 1.0 - (totalSimilarity / comparisons); // 다양성 점수
        }
    }

    // 2. 다양성 최적화
    @Service
    public class DiversityOptimizer {
        private final DiversityMetricsCalculator metricsCalculator;
        
        public List<Recommendation> optimizeDiversity(
            List<Recommendation> candidates, 
            DiversityConfig config) {

            List<Recommendation> selected = new ArrayList<>();
            Set<String> selectedCategories = new HashSet<>();

            // MMR (Maximal Marginal Relevance) 알고리즘 적용
            while (selected.size() < config.getTargetSize()) {
                Recommendation nextItem = findNextDiverseItem(
                    candidates, selected, selectedCategories, config);
                
                if (nextItem == null) break;
                
                selected.add(nextItem);
                selectedCategories.add(
                    categoryService.getCategory(nextItem.getItemId()));
            }

            return selected;
        }

        private Recommendation findNextDiverseItem(
            List<Recommendation> candidates,
            List<Recommendation> selected,
            Set<String> selectedCategories,
            DiversityConfig config) {

            return candidates.stream()
                .filter(item -> !selected.contains(item))
                .max(Comparator.comparingDouble(item -> 
                    calculateDiversityScore(
                        item, selected, selectedCategories, config)))
                .orElse(null);
        }

        private double calculateDiversityScore(
            Recommendation item,
            List<Recommendation> selected,
            Set<String> selectedCategories,
            DiversityConfig config) {

            double relevanceScore = item.getScore();
            double diversityScore = calculateDiversityContribution(
                item, selected, selectedCategories);

            return config.getLambda() * relevanceScore + 
                   (1 - config.getLambda()) * diversityScore;
        }
    }

    // 3. 적응형 다양성 조정
    @Service
    public class AdaptiveDiversityAdjuster {
        private final UserFeedbackAnalyzer feedbackAnalyzer;
        private final MetricsRegistry metricsRegistry;

        public DiversityConfig adaptDiversitySettings(
            String userId, 
            UserProfile profile) {

            // 사용자 피드백 분석
            UserFeedbackStats feedback = 
                feedbackAnalyzer.analyzeFeedback(userId);

            // 참여도 메트릭 계산
            EngagementMetrics engagement = 
                calculateEngagementMetrics(userId);

            return DiversityConfig.builder()
                .lambda(calculateOptimalLambda(feedback, engagement))
                .categoryWeight(calculateCategoryWeight(profile))
                .noveltyWeight(calculateNoveltyWeight(profile))
                .temporalWeight(calculateTemporalWeight(profile))
                .build();
        }

        private double calculateOptimalLambda(
            UserFeedbackStats feedback, 
            EngagementMetrics engagement) {

            // 기본 람다 값 설정
            double baseLambda = 0.7;  // relevance와 diversity의 기본 비율

            // 피드백 기반 조정
            if (feedback.getDiversityPreference() > 0.8) {
                baseLambda -= 0.2;  // diversity 선호도가 높은 경우
            }

            // 참여도 기반 조정
            if (engagement.getClickThroughRate() < 0.1) {
                baseLambda -= 0.1;  // 참여도가 낮은 경우 다양성 증가
            }

            return Math.max(0.3, Math.min(0.9, baseLambda));
        }
    }

    // 4. A/B 테스팅 및 모니터링
    @Service
    public class DiversityExperimentManager {
        private final ExperimentService experimentService;
        private final MetricsCollector metricsCollector;

        public void runDiversityExperiment(String experimentId) {
            // 실험 그룹 설정
            ExperimentConfig config = ExperimentConfig.builder()
                .controlGroup(DiversityConfig.defaultConfig())
                .treatmentGroup(DiversityConfig.highDiversityConfig())
                .metrics(Arrays.asList(
                    "click_through_rate",
                    "session_length",
                    "user_satisfaction",
                    "recommendation_diversity"
                ))
                .duration(Duration.ofDays(7))
                .build();

            experimentService.startExperiment(experimentId, config);

            // 메트릭 모니터링
            monitorExperimentMetrics(experimentId);
        }

        @Scheduled(fixedRate = 3600000) // 매시간 실행
        public void monitorExperimentMetrics(String experimentId) {
            ExperimentMetrics metrics = 
                metricsCollector.collectMetrics(experimentId);

            // 통계적 유의성 검증
            if (metrics.hasSignificantDifference()) {
                notifyStakeholders(metrics);
            }

            // 실험 결과 로깅
            logExperimentResults(experimentId, metrics);
        }
    }
}
```

이러한 다양성 관리 시스템을 통해:

1. 다양성 측정
   - 카테고리 다양성
   - 콘텐츠 다양성
   - 시간적 다양성
   - 속성 다양성

2. 다양성 최적화
   - MMR 알고리즘
   - 밸런싱 전략
   - 카테고리 분산

3. 적응형 조정
   - 사용자 피드백 반영
   - 참여도 기반 조정
   - 동적 파라미터 조정

4. 실험 및 모니터링
   - A/B 테스팅
   - 메트릭 추적
   - 결과 분석

을 구현하여 추천의 정확도와 다양성 사이의 최적의 균형을 찾을 수 있습니다.

특히 중요한 점은:
- 사용자별 맞춤 다양성
- 피드백 기반 조정
- 지속적인 실험
- 메트릭 기반 의사결정

입니다.