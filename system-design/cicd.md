# CI/CD 파이프라인 시스템 설계

면접관: "대규모 조직에서 사용할 수 있는 CI/CD 파이프라인 시스템을 설계해주세요. 다중 레포지토리 지원, 병렬 빌드, 테스트 자동화, 배포 전략이 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 일일 빌드 수와 동시 빌드 수는 어느 정도인가요?
2. 빌드/테스트 환경의 격리 수준은 어떻게 되나요?
3. 아티팩트 보관 기간과 크기 제한이 있나요?
4. 빌드 실패 시 알림이나 롤백 전략이 필요한가요?

면접관:
1. 일일 1000회 빌드, 최대 100개 동시 빌드
2. 완전한 컨테이너 격리 필요, 민감한 환경 변수 관리 포함
3. 아티팩트는 90일 보관, 저장소당 최대 500GB
4. Slack/이메일 알림, 자동 롤백 기능 필요

## 1. 파이프라인 코어 시스템

```java
@Service
public class PipelineService {
    
    private final JobExecutor jobExecutor;
    private final ArtifactStorage artifactStorage;
    private final NotificationService notificationService;

    // 1. 파이프라인 정의
    public class Pipeline {
        private final String pipelineId;
        private final List<Stage> stages;
        private final Map<String, String> environment;
        
        @Data
        public class Stage {
            private final String stageName;
            private final List<Job> jobs;
            private final StageType type;  // BUILD, TEST, DEPLOY
            private final RetryPolicy retryPolicy;
            private final List<String> dependencies;
        }

        public PipelineExecution execute() {
            // DAG 기반 스테이지 실행
            return stages.stream()
                .filter(stage -> canExecuteStage(stage))
                .map(stage -> executeStage(stage))
                .collect(Collectors.toList());
        }
    }

    // 2. 작업 실행 엔진
    @Service
    public class JobExecutor {
        private final KubernetesClient kubernetesClient;
        private final SecretManager secretManager;
        
        public JobResult executeJob(Job job) {
            // 작업 환경 준비
            Pod pod = preparePodSpec(job);
            
            // 환경 변수 및 시크릿 주입
            injectSecrets(pod, job.getSecrets());
            
            // 작업 실행
            try {
                kubernetesClient.pods().create(pod);
                return watchJobCompletion(pod.getMetadata().getName());
            } catch (Exception e) {
                handleJobFailure(job, e);
                return JobResult.failure(e);
            }
        }

        private Pod preparePodSpec(Job job) {
            return new PodBuilder()
                .withNewMetadata()
                    .withName(job.getId())
                    .withLabels(job.getLabels())
                .endMetadata()
                .withNewSpec()
                    .withContainers(createJobContainer(job))
                    .withRestartPolicy("Never")
                    .withServiceAccount("ci-runner")
                .endSpec()
                .build();
        }
    }

    // 3. 아티팩트 관리
    @Service
    public class ArtifactManager {
        private final S3Client s3Client;
        private final ArtifactMetadataRepository metadataRepo;
        
        public void storeArtifact(String pipelineId, 
                                 MultipartFile artifact) {
            String artifactPath = generateArtifactPath(pipelineId);
            
            // S3에 아티팩트 저장
            s3Client.putObject(PutObjectRequest.builder()
                .bucket(ARTIFACT_BUCKET)
                .key(artifactPath)
                .build(), 
                RequestBody.fromInputStream(
                    artifact.getInputStream(), 
                    artifact.getSize()
                ));
                
            // 메타데이터 저장
            ArtifactMetadata metadata = ArtifactMetadata.builder()
                .pipelineId(pipelineId)
                .path(artifactPath)
                .size(artifact.getSize())
                .createdAt(Instant.now())
                .build();
                
            metadataRepo.save(metadata);
        }
        
        // 아티팩트 보관 정책 적용
        @Scheduled(cron = "0 0 0 * * *")  // 매일 자정
        public void applyRetentionPolicy() {
            LocalDate cutoffDate = 
                LocalDate.now().minusDays(90);
                
            List<ArtifactMetadata> expiredArtifacts = 
                metadataRepo.findByCreatedAtBefore(cutoffDate);
                
            expiredArtifacts.forEach(artifact -> {
                s3Client.deleteObject(DeleteObjectRequest.builder()
                    .bucket(ARTIFACT_BUCKET)
                    .key(artifact.getPath())
                    .build());
                    
                metadataRepo.delete(artifact);
            });
        }
    }
}
```

## 2. 파이프라인 실행과 모니터링 시스템

```java
@Service
public class PipelineExecutionService {

    // 1. 병렬 실행 관리
    public class ParallelExecutionManager {
        private final ExecutorService executorService;
        private final SemaphoreManager semaphoreManager;

        public ParallelExecutionManager() {
            this.executorService = Executors.newFixedThreadPool(
                Runtime.getRuntime().availableProcessors() * 2
            );
            this.semaphoreManager = new SemaphoreManager(100); // 최대 100개 동시 실행
        }

        public CompletableFuture<StageResult> executeStageInParallel(Stage stage) {
            return CompletableFuture.supplyAsync(() -> {
                String resourceType = stage.getResourceRequirements().getType();
                try {
                    semaphoreManager.acquire(resourceType);
                    return executeStage(stage);
                } finally {
                    semaphoreManager.release(resourceType);
                }
            }, executorService);
        }
    }

    // 2. 실시간 모니터링
    @Service
    public class PipelineMonitor {
        private final MetricsRegistry metricsRegistry;
        private final AlertService alertService;

        public void monitorPipeline(String pipelineId) {
            Pipeline pipeline = getPipeline(pipelineId);
            
            // 빌드 매트릭스 수집
            metricsRegistry.gauge(
                "pipeline.duration",
                pipeline.getDuration().toSeconds()
            );
            
            metricsRegistry.counter(
                "pipeline.stages.total",
                pipeline.getStages().size()
            );
            
            // 성능 모니터링
            if (pipeline.getDuration().toMinutes() > 30) {
                alertService.sendAlert(
                    AlertLevel.WARNING,
                    "Pipeline duration exceeds 30 minutes: " + pipelineId
                );
            }

            // 리소스 사용량 모니터링
            monitorResourceUsage(pipeline);
        }

        private void monitorResourceUsage(Pipeline pipeline) {
            ResourceUsage usage = pipeline.getResourceUsage();
            
            if (usage.getCpuUsage() > 80) {
                alertService.sendAlert(
                    AlertLevel.CRITICAL,
                    "High CPU usage in pipeline: " + pipeline.getId()
                );
            }
        }
    }

    // 3. 로그 집계 시스템
    @Service
    public class LogAggregator {
        private final ElasticsearchClient esClient;
        private final KafkaTemplate<String, LogEvent> kafkaTemplate;

        @KafkaListener(topics = "pipeline-logs")
        public void handleLogs(LogEvent logEvent) {
            // 로그 인덱싱
            IndexRequest<LogEvent> request = IndexRequest.of(r -> r
                .index("pipeline-logs-" + 
                    LocalDate.now().format(DateTimeFormatter.ISO_DATE))
                .document(logEvent)
            );

            esClient.index(request);

            // 실시간 로그 스트리밍
            if (logEvent.getLevel().equals(LogLevel.ERROR)) {
                notifyError(logEvent);
            }
        }

        public List<LogEvent> queryLogs(String pipelineId, 
                                      TimeRange range) {
            // 로그 검색 쿼리
            SearchRequest request = SearchRequest.of(r -> r
                .index("pipeline-logs-*")
                .query(q -> q
                    .bool(b -> b
                        .must(m -> m
                            .term(t -> t
                                .field("pipelineId")
                                .value(pipelineId)))
                        .must(m -> m
                            .range(ra -> ra
                                .field("timestamp")
                                .gte(JsonData.of(range.getStart()))
                                .lte(JsonData.of(range.getEnd()))))))
            );

            return esClient.search(request, LogEvent.class)
                .hits()
                .hits()
                .stream()
                .map(hit -> hit.source())
                .collect(Collectors.toList());
        }
    }

    // 4. 실패 분석 및 자동 복구
    @Service
    public class FailureAnalyzer {
        
        public void analyzePipelineFailure(Pipeline pipeline) {
            FailureReport report = FailureReport.builder()
                .pipelineId(pipeline.getId())
                .failureStage(pipeline.getFailedStage())
                .logs(extractRelevantLogs(pipeline))
                .resourceUsage(pipeline.getResourceUsage())
                .build();

            // 실패 패턴 분석
            FailurePattern pattern = detectFailurePattern(report);

            // 자동 복구 시도
            if (pattern.isAutoRecoverable()) {
                attemptRecovery(pipeline, pattern);
            } else {
                // 수동 개입 필요
                notifyTeam(report);
            }
        }

        private void attemptRecovery(Pipeline pipeline, 
                                   FailurePattern pattern) {
            switch (pattern.getType()) {
                case RESOURCE_EXHAUSTION:
                    scaleResources(pipeline);
                    break;
                case TRANSIENT_FAILURE:
                    retryPipeline(pipeline);
                    break;
                case DEPENDENCY_FAILURE:
                    updateDependencies(pipeline);
                    break;
            }
        }
    }
}
```

이러한 모니터링 및 실행 시스템을 통해:

1. 병렬 실행 관리
    - 리소스 기반 제한
    - 세마포어를 통한 동시성 제어
    - 효율적인 스케줄링

2. 실시간 모니터링
    - 성능 메트릭 수집
    - 리소스 사용량 추적
    - 알림 시스템

3. 로그 관리
    - 실시간 로그 수집
    - 검색 가능한 로그 저장
    - 에러 감지 및 알림

4. 실패 분석
    - 패턴 기반 분석
    - 자동 복구 시도
    - 팀 알림

을 구현할 수 있습니다.

면접관: 대규모 마이크로서비스 환경에서 여러 서비스의 CI/CD를 어떻게 조정하시겠습니까?

## 3. 마이크로서비스 CI/CD 조정 시스템

```java
@Service
public class MicroserviceCICoordinator {

    // 1. 의존성 그래프 관리
    public class DependencyGraphManager {
        private final Map<String, ServiceNode> serviceGraph = new ConcurrentHashMap<>();

        @Data
        public class ServiceNode {
            private final String serviceId;
            private final Set<String> dependencies;
            private final Set<String> dependents;
            private ServiceVersion currentVersion;
            private DeploymentStatus status;
        }

        // 의존성 기반 빌드 순서 결정
        public List<String> determineBuildOrder() {
            // 위상 정렬 구현
            Set<String> visited = new HashSet<>();
            List<String> buildOrder = new ArrayList<>();
            
            serviceGraph.keySet().forEach(serviceId -> {
                if (!visited.contains(serviceId)) {
                    topologicalSort(serviceId, visited, buildOrder);
                }
            });
            
            return buildOrder;
        }
        
        // 영향도 분석
        public Set<String> analyzeImpact(String changedService) {
            Set<String> impactedServices = new HashSet<>();
            Queue<String> queue = new LinkedList<>();
            queue.add(changedService);
            
            while (!queue.isEmpty()) {
                String service = queue.poll();
                ServiceNode node = serviceGraph.get(service);
                
                node.getDependents().forEach(dependent -> {
                    if (impactedServices.add(dependent)) {
                        queue.add(dependent);
                    }
                });
            }
            
            return impactedServices;
        }
    }

    // 2. 버전 관리 및 호환성 체크
    @Service
    public class VersionCompatibilityManager {
        private final ContractTestRunner contractTestRunner;

        public boolean checkCompatibility(ServiceVersion newVersion, 
                                        Set<ServiceVersion> dependencyVersions) {
            // API 계약 테스트 실행
            ContractTestResult result = contractTestRunner
                .runTests(newVersion, dependencyVersions);

            if (!result.isCompatible()) {
                handleCompatibilityFailure(result);
                return false;
            }

            // 성능 회귀 테스트
            PerformanceTestResult perfResult = 
                runPerformanceTests(newVersion);
                
            return perfResult.meetsThreshold();
        }

        private void handleCompatibilityFailure(ContractTestResult result) {
            // 실패한 계약 테스트 분석
            List<FailedContract> failedContracts = result.getFailedContracts();
            
            // 영향받는 서비스 팀에 알림
            notifyTeams(failedContracts);
            
            // 변경 가이드 생성
            generateChangeGuide(failedContracts);
        }
    }

    // 3. 통합 테스트 오케스트레이션
    @Service
    public class IntegrationTestOrchestrator {
        private final KubernetesClient kubernetesClient;
        private final TestEnvironmentManager testEnvManager;

        public IntegrationTestResult runIntegrationTests(
            Set<MicroserviceDeployment> deployments) {
            
            // 테스트 환경 프로비저닝
            TestEnvironment env = testEnvManager
                .provisionEnvironment(deployments);

            try {
                // 서비스 배포
                deployServices(env, deployments);

                // 통합 테스트 실행
                TestSuite integrationTests = 
                    TestSuiteBuilder.forDeployments(deployments)
                        .withEnvironment(env)
                        .build();

                return testRunner.runTests(integrationTests);

            } finally {
                // 환경 정리
                testEnvManager.cleanupEnvironment(env);
            }
        }

        private void deployServices(TestEnvironment env, 
                                  Set<MicroserviceDeployment> deployments) {
            // 병렬 배포 실행
            CompletableFuture<?>[] deployFutures = deployments.stream()
                .map(deployment -> CompletableFuture.runAsync(() ->
                    deployService(env, deployment)))
                .toArray(CompletableFuture[]::new);

            CompletableFuture.allOf(deployFutures).join();
        }
    }

    // 4. 롤아웃 조정
    @Service
    public class RolloutCoordinator {
        private final DeploymentManager deploymentManager;
        private final HealthCheckService healthCheckService;

        public void coordinateRollout(
            Set<MicroserviceDeployment> deployments) {
            
            // 배포 그룹 생성
            List<DeploymentGroup> deploymentGroups = 
                createDeploymentGroups(deployments);

            for (DeploymentGroup group : deploymentGroups) {
                // 그룹별 롤아웃
                RolloutResult result = rolloutGroup(group);
                
                if (!result.isSuccessful()) {
                    // 롤백 처리
                    handleRolloutFailure(group, result);
                    break;
                }
                
                // 안정화 대기
                waitForStabilization(group);
            }
        }

        private void waitForStabilization(DeploymentGroup group) {
            // 헬스체크 및 메트릭 모니터링
            Duration stabilizationPeriod = Duration.ofMinutes(15);
            Instant deadline = Instant.now().plus(stabilizationPeriod);
            
            while (Instant.now().isBefore(deadline)) {
                if (!isGroupHealthy(group)) {
                    throw new StabilizationException("Group failed to stabilize");
                }
                Thread.sleep(Duration.ofSeconds(30).toMillis());
            }
        }

        private boolean isGroupHealthy(DeploymentGroup group) {
            return group.getDeployments().stream()
                .allMatch(deployment -> 
                    healthCheckService.isHealthy(deployment) &&
                    metricsService.isWithinThresholds(deployment));
        }
    }
}
```

이러한 마이크로서비스 CI/CD 조정 시스템을 통해:

1. 의존성 관리
    - 서비스 간 의존성 추적
    - 빌드 순서 최적화
    - 영향도 분석

2. 버전 호환성
    - API 계약 테스트
    - 성능 회귀 테스트
    - 호환성 문제 조기 발견

3. 통합 테스트
    - 자동화된 테스트 환경
    - 병렬 테스트 실행
    - 환경 관리 자동화

4. 조정된 롤아웃
    - 그룹 기반 배포
    - 헬스체크 모니터링
    - 자동 롤백

을 구현하여 대규모 마이크로서비스 환경에서의 CI/CD를 효과적으로 관리할 수 있습니다.

특히 중요한 점은:
- 서비스 간 의존성 관리
- 자동화된 호환성 검증
- 안정적인 롤아웃 전략
- 효율적인 테스트 실행

입니다.