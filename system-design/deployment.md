# 코드 배포 시스템 설계 (Spinnaker 유형)

면접관: "대규모 마이크로서비스 환경에서 안전하고 효율적인 코드 배포 시스템을 설계해주세요. 카나리 배포, 블루/그린 배포, 자동 롤백 기능이 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 일일 배포 횟수와 동시 배포 수는 어느 정도인가요?
2. 다중 클라우드/리전 지원이 필요한가요?
3. 자동 롤백의 기준은 어떻게 되나요?
4. 배포 승인 프로세스가 필요한가요?

면접관:
1. 일일 100회 배포, 최대 10개 동시 배포
2. AWS, GCP 멀티 클라우드 지원 필요
3. Error rate 2% 초과 또는 P99 레이턴시 1초 초과 시 롤백
4. Production 환경은 수동 승인 필요

## 1. 배포 파이프라인 시스템

```java
@Service
public class DeploymentPipelineService {

    // 1. 배포 파이프라인 정의
    public class Pipeline {
        private final String pipelineId;
        private final ApplicationConfig application;
        private final DeploymentStrategy strategy;
        private final List<Stage> stages;
        private final Map<String, String> parameters;

        @Data
        public class Stage {
            private final String name;
            private final StageType type;
            private final Map<String, Object> config;
            private final List<String> dependencies;
            private final ApprovalConfig approval;
        }
    }

    // 2. 배포 전략 구현
    @Service
    public class DeploymentStrategyExecutor {
        private final CloudProviderClient cloudClient;
        private final MetricsService metricsService;

        public DeploymentResult executeCanaryDeployment(
            DeploymentConfig config) {
            
            // 카나리 인스턴스 배포
            String canaryId = deployCanaryInstance(config);
            
            // 트래픽 점진적 증가
            for (int percentage : config.getTrafficSteps()) {
                routeTraffic(canaryId, percentage);
                
                // 메트릭 모니터링
                if (!validateCanaryMetrics(canaryId)) {
                    return rollbackCanary(canaryId);
                }
                
                // 안정화 대기
                waitForStabilization(Duration.ofMinutes(5));
            }
            
            // 완전 전환
            return promoteCanary(canaryId);
        }

        public DeploymentResult executeBlueGreenDeployment(
            DeploymentConfig config) {
            
            // 그린 환경 배포
            String greenId = deployGreenEnvironment(config);
            
            // 헬스체크
            if (!validateEnvironment(greenId)) {
                return rollbackDeployment(greenId);
            }
            
            // 트래픽 전환
            switchTraffic(config.getBlueId(), greenId);
            
            // 블루 환경 정리
            if (config.isCleanupEnabled()) {
                scheduleCleanup(config.getBlueId());
            }
            
            return DeploymentResult.success(greenId);
        }
    }

    // 3. 멀티 클라우드 배포
    @Service
    public class MultiCloudDeploymentService {
        private final Map<CloudProvider, CloudDeploymentHandler> handlers;
        
        public void executeMultiCloudDeployment(
            DeploymentConfig config) {
            
            // 클라우드별 배포 설정
            Map<CloudProvider, DeploymentConfig> cloudConfigs = 
                prepareCloudConfigs(config);
            
            // 병렬 배포 실행
            CompletableFuture<?>[] deployments = cloudConfigs.entrySet()
                .stream()
                .map(entry -> CompletableFuture.runAsync(() ->
                    handlers.get(entry.getKey())
                           .deploy(entry.getValue())))
                .toArray(CompletableFuture[]::new);
                
            try {
                CompletableFuture.allOf(deployments).join();
            } catch (Exception e) {
                // 실패 시 모든 클라우드에서 롤백
                rollbackAllClouds(cloudConfigs.keySet());
                throw e;
            }
        }
    }
}
```

## 2. 배포 모니터링과 롤백 시스템

```java
@Service
public class DeploymentMonitoringService {

    // 1. 메트릭 모니터링
    public class MetricMonitor {
        private final PrometheusClient prometheusClient;
        private final DatadogClient datadogClient;
        private final AlertManager alertManager;

        public boolean validateDeployment(String deploymentId) {
            DeploymentMetrics metrics = collectMetrics(deploymentId);
            
            // 주요 메트릭 검증
            boolean isHealthy = Stream.of(
                validateErrorRate(metrics),
                validateLatency(metrics),
                validateResourceUsage(metrics),
                validateCustomMetrics(metrics)
            ).allMatch(result -> result);

            if (!isHealthy) {
                alertManager.sendAlert(AlertLevel.HIGH,
                    String.format("Deployment %s failed metrics validation", 
                        deploymentId));
            }

            return isHealthy;
        }

        private boolean validateErrorRate(DeploymentMetrics metrics) {
            double errorRate = metrics.getErrorRate();
            if (errorRate > 0.02) { // 2% threshold
                logViolation("Error rate too high: " + errorRate);
                return false;
            }
            return true;
        }

        private boolean validateLatency(DeploymentMetrics metrics) {
            Duration p99Latency = metrics.getP99Latency();
            if (p99Latency.toMillis() > 1000) { // 1s threshold
                logViolation("P99 latency too high: " + p99Latency);
                return false;
            }
            return true;
        }
    }

    // 2. 자동 롤백 시스템
    @Service
    public class AutoRollbackService {
        private final DeploymentManager deploymentManager;
        private final MetricMonitor metricMonitor;
        private final NotificationService notificationService;

        public void monitorAndRollback(String deploymentId) {
            try {
                // 배포 모니터링 시작
                DeploymentWatch watch = new DeploymentWatch(deploymentId);
                watch.startMonitoring(Duration.ofMinutes(30)); // 30분 모니터링

                while (watch.isActive()) {
                    if (!metricMonitor.validateDeployment(deploymentId)) {
                        initiateRollback(deploymentId, 
                            RollbackReason.METRIC_VIOLATION);
                        break;
                    }
                    Thread.sleep(10000); // 10초마다 체크
                }
            } catch (Exception e) {
                handleMonitoringFailure(deploymentId, e);
            }
        }

        private void initiateRollback(String deploymentId, 
                                    RollbackReason reason) {
            try {
                // 롤백 전 스냅샷 생성
                DeploymentSnapshot snapshot = 
                    deploymentManager.createSnapshot(deploymentId);
                
                // 롤백 실행
                RollbackResult result = 
                    deploymentManager.rollback(deploymentId, snapshot);
                
                // 결과 통보
                notifyRollback(deploymentId, reason, result);
                
            } catch (Exception e) {
                handleRollbackFailure(deploymentId, e);
            }
        }
    }

    // 3. 배포 상태 추적
    @Service
    public class DeploymentStateTracker {
        private final RedisTemplate<String, DeploymentState> redisTemplate;
        
        public void trackDeploymentProgress(String deploymentId) {
            DeploymentState state = new DeploymentState();
            
            // 단계별 진행상황 추적
            state.addStage("PREPARATION", StageStatus.COMPLETED);
            state.addStage("VALIDATION", StageStatus.IN_PROGRESS);
            
            // Redis에 상태 저장
            redisTemplate.opsForValue().set(
                "deployment:" + deploymentId, 
                state,
                Duration.ofHours(24)
            );
        }

        // 실시간 상태 조회
        public DeploymentState getDeploymentState(String deploymentId) {
            return redisTemplate.opsForValue()
                .get("deployment:" + deploymentId);
        }

        // 배포 이력 관리
        @Scheduled(fixedRate = 86400000) // 매일 실행
        public void archiveDeploymentHistory() {
            List<DeploymentState> completedDeployments = 
                findCompletedDeployments();
            
            for (DeploymentState deployment : completedDeployments) {
                // S3에 이력 저장
                archiveDeployment(deployment);
                // Redis에서 제거
                redisTemplate.delete("deployment:" + 
                    deployment.getDeploymentId());
            }
        }
    }

    // 4. 알림 및 리포팅
    @Service
    public class DeploymentNotificationService {
        private final SlackClient slackClient;
        private final EmailService emailService;
        private final MetricsRegistry metricsRegistry;

        public void sendDeploymentNotification(
            DeploymentEvent event) {
            
            // Slack 알림
            if (event.isHighPriority()) {
                slackClient.sendMessage(
                    "#deployments-critical",
                    createSlackMessage(event)
                );
            }

            // 이메일 알림
            if (event.requiresApproval()) {
                emailService.sendApprovalRequest(
                    event.getApprovers(),
                    createApprovalEmail(event)
                );
            }

            // 메트릭 기록
            metricsRegistry.counter(
                "deployments.total",
                "status", event.getStatus().toString()
            ).increment();
        }

        public DeploymentReport generateDeploymentReport(
            String deploymentId) {
            
            DeploymentState state = stateTracker
                .getDeploymentState(deploymentId);
            
            return DeploymentReport.builder()
                .deploymentId(deploymentId)
                .duration(state.getDuration())
                .stages(state.getStages())
                .metrics(metricMonitor.getMetrics(deploymentId))
                .incidents(getIncidents(deploymentId))
                .build();
        }
    }
}
```

이러한 모니터링과 롤백 시스템을 통해:

1. 실시간 메트릭 모니터링
    - 에러율 모니터링
    - 레이턴시 모니터링
    - 리소스 사용량 모니터링
    - 커스텀 메트릭 지원

2. 자동 롤백
    - 메트릭 기반 롤백
    - 스냅샷 기반 복구
    - 롤백 실패 처리

3. 상태 추적
    - 실시간 진행상황 추적
    - 이력 관리
    - 아카이빙

4. 알림 및 보고
    - 다중 채널 알림
    - 승인 요청 관리
    - 상세 리포트 생성

을 구현할 수 있습니다.

면접관: 대규모 배포 시 발생할 수 있는 리스크를 어떻게 관리하시겠습니까?

## 3. 배포 리스크 관리 시스템

```java
@Service
public class DeploymentRiskManagementService {

    // 1. 리스크 평가 시스템
    @Service
    public class RiskAssessor {
        private final DeploymentHistory deploymentHistory;
        private final ServiceDependencyGraph dependencyGraph;

        public RiskAssessment assessDeploymentRisk(DeploymentRequest request) {
            RiskScore score = RiskScore.builder()
                .changeScope(evaluateChangeScope(request))
                .timeRisk(evaluateTimeRisk(request))
                .dependencyRisk(evaluateDependencyRisk(request))
                .historicalRisk(evaluateHistoricalRisk(request))
                .build();

            // 리스크 레벨 결정
            RiskLevel riskLevel = determineRiskLevel(score);
            
            return RiskAssessment.builder()
                .level(riskLevel)
                .score(score)
                .mitigationStrategies(suggestMitigationStrategies(riskLevel))
                .build();
        }

        private double evaluateDependencyRisk(DeploymentRequest request) {
            Set<String> impactedServices = 
                dependencyGraph.getImpactedServices(request.getServiceId());
            
            return impactedServices.stream()
                .mapToDouble(serviceId -> 
                    calculateServiceCriticality(serviceId))
                .sum();
        }

        private double evaluateHistoricalRisk(DeploymentRequest request) {
            // 과거 배포 성공률 분석
            DeploymentStats stats = deploymentHistory
                .getStats(request.getServiceId());
            
            return calculateRiskScore(
                stats.getFailureRate(),
                stats.getAverageRecoveryTime(),
                stats.getRollbackFrequency()
            );
        }
    }

    // 2. 배포 제약 조건 관리
    @Service
    public class DeploymentConstraintManager {
        private final BusinessCalendarService calendarService;
        private final TrafficAnalyzer trafficAnalyzer;

        public boolean validateDeploymentConstraints(
            DeploymentRequest request) {
            
            return Stream.of(
                validateTimeWindow(request),
                validateConcurrentDeployments(),
                validateTrafficConditions(),
                validateDependencyFreeze()
            ).allMatch(result -> result);
        }

        private boolean validateTimeWindow(DeploymentRequest request) {
            // 배포 금지 시간대 체크
            if (isInBlackoutPeriod()) {
                throw new DeploymentConstraintException(
                    "Deployment not allowed during blackout period");
            }

            // 비즈니스 크리티컬 시간대 체크
            if (isBusinessCriticalPeriod() && 
                !request.hasEmergencyOverride()) {
                throw new DeploymentConstraintException(
                    "Deployment not allowed during business critical hours");
            }

            return true;
        }

        private boolean validateTrafficConditions() {
            TrafficStats currentTraffic = 
                trafficAnalyzer.getCurrentTraffic();
            
            return currentTraffic.getLoad() < 
                   ThresholdConfig.MAX_DEPLOYMENT_TRAFFIC_LOAD;
        }
    }

    // 3. 점진적 롤아웃 제어
    @Service
    public class GradualRolloutController {
        private final MetricAnalyzer metricAnalyzer;
        private final AlertManager alertManager;

        public void controlRollout(DeploymentConfig config) {
            List<RolloutStage> stages = planRolloutStages(config);
            
            for (RolloutStage stage : stages) {
                try {
                    executeRolloutStage(stage);
                    
                    // 안정화 기간 동안 모니터링
                    if (!monitorStageHealth(stage)) {
                        initiateStageRollback(stage);
                        throw new RolloutException(
                            "Stage health check failed");
                    }
                    
                } catch (Exception e) {
                    handleRolloutFailure(stage, e);
                    break;
                }
            }
        }

        private boolean monitorStageHealth(RolloutStage stage) {
            Duration monitoringWindow = stage.getMonitoringWindow();
            Instant deadline = Instant.now().plus(monitoringWindow);
            
            while (Instant.now().isBefore(deadline)) {
                StageMetrics metrics = 
                    metricAnalyzer.analyzeStage(stage);
                
                if (metrics.hasAnomalies()) {
                    alertManager.sendAlert(
                        createAnomalyAlert(stage, metrics));
                    return false;
                }
                
                Thread.sleep(stage.getCheckInterval().toMillis());
            }
            
            return true;
        }
    }

    // 4. 긴급 대응 시스템
    @Service
    public class EmergencyResponseSystem {
        private final IncidentManager incidentManager;
        private final ServiceRegistry serviceRegistry;

        public void handleEmergencyDeployment(
            EmergencyDeploymentRequest request) {
            
            // 인시던트 생성
            Incident incident = incidentManager
                .createIncident(request.getReason());
            
            try {
                // 긴급 배포 실행
                executeEmergencyDeployment(request);
                
                // 영향 분석
                analyzeImpact(request.getServiceId());
                
            } catch (Exception e) {
                // 자동 롤백 및 알림
                handleEmergencyFailure(incident, e);
            }
        }

        public void initiateEmergencyRollback(String deploymentId) {
            // 롤백 전 상태 스냅샷
            DeploymentSnapshot snapshot = 
                createPreRollbackSnapshot(deploymentId);
            
            // 우선순위 격상
            prioritizeRollback(deploymentId);
            
            // 긴급 롤백 실행
            executeEmergencyRollback(deploymentId, snapshot);
        }
    }
}
```

이러한 리스크 관리 시스템을 통해:

1. 사전 리스크 평가
    - 변경 범위 평가
    - 시간대 리스크
    - 의존성 리스크
    - 히스토리 기반 리스크

2. 배포 제약 관리
    - 시간 윈도우 제한
    - 동시 배포 제한
    - 트래픽 기반 제한
    - 의존성 프리즈

3. 점진적 롤아웃
    - 단계별 배포
    - 실시간 모니터링
    - 자동 롤백
    - 안정화 기간 관리

4. 긴급 상황 대응
    - 긴급 배포 프로세스
    - 신속한 롤백
    - 영향 분석
    - 인시던트 관리

을 구현하여 대규모 배포의 리스크를 최소화할 수 있습니다.

특히 중요한 점은:
- 데이터 기반 의사결정
- 자동화된 안전장치
- 점진적 접근
- 긴급 상황 대비

입니다.