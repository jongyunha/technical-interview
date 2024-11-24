# 동영상 스트리밍 플랫폼 설계 (Netflix/YouTube 유형)

면접관: "대규모 동영상 스트리밍 플랫폼을 설계해주세요. 동영상 업로드, 트랜스코딩, 스트리밍 기능이 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 예상 사용자 수와 동시 시청자 수는 어느 정도인가요?
2. 평균 동영상 크기와 길이는 어느 정도인가요?
3. 지원해야 하는 화질과 디바이스 종류는 어떻게 되나요?
4. 실시간 스트리밍도 지원해야 하나요?

면접관:
1. DAU 100만, 최대 동시 시청자 50만 명
2. 평균 10분 길이, 500MB 크기
3. 240p부터 4K까지, 모바일/태블릿/웹/스마트TV 지원
4. 실시간 스트리밍은 현재 단계에서는 불필요

## 1. 동영상 업로드 및 저장

```java
@Service
public class VideoUploadService {
    
    private final S3Client s3Client;
    private final CloudFrontClient cloudFrontClient;
    private final TranscodingService transcodingService;
    
    // 1. 청크 기반 업로드
    public UploadResponse handleChunkUpload(MultipartFile chunk, 
                                          String uploadId, 
                                          int partNumber) {
        try {
            // 청크 임시 저장
            String chunkKey = String.format("temp/%s/part-%d", uploadId, partNumber);
            s3Client.putObject(PutObjectRequest.builder()
                .bucket(UPLOAD_BUCKET)
                .key(chunkKey)
                .build(), 
                RequestBody.fromInputStream(chunk.getInputStream(), 
                                         chunk.getSize()));
            
            // 업로드 진행상황 업데이트
            updateUploadProgress(uploadId, partNumber);
            
            return new UploadResponse(uploadId, partNumber, "SUCCESS");
        } catch (Exception e) {
            log.error("Upload failed for chunk: " + partNumber, e);
            return new UploadResponse(uploadId, partNumber, "FAILED");
        }
    }
    
    // 2. 청크 병합 및 트랜스코딩 시작
    public void completeUpload(String uploadId) {
        // 청크 병합
        List<CompletedPart> completedParts = getCompletedParts(uploadId);
        String finalKey = "raw/" + generateVideoId() + ".mp4";
        
        s3Client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
            .bucket(UPLOAD_BUCKET)
            .key(finalKey)
            .uploadId(uploadId)
            .multipartUpload(CompletedMultipartUpload.builder()
                .parts(completedParts)
                .build())
            .build());
            
        // 트랜스코딩 작업 시작
        transcodingService.startTranscoding(finalKey);
    }
}
```

## 2. 트랜스코딩 서비스

```java
@Service
public class TranscodingService {
    
    private final MediaConvertClient mediaConvert;
    private final JobRepository jobRepository;
    private final NotificationService notificationService;
    
    // 1. 트랜스코딩 작업 생성
    public void startTranscoding(String videoKey) {
        // 품질별 출력 설정
        List<OutputGroup> outputs = Arrays.asList(
            createOutput("240p", 426, 240),
            createOutput("480p", 854, 480),
            createOutput("720p", 1280, 720),
            createOutput("1080p", 1920, 1080),
            createOutput("4K", 3840, 2160)
        );
        
        // 트랜스코딩 작업 시작
        Job job = Job.builder()
            .input(Input.builder()
                .fileInput("s3://" + UPLOAD_BUCKET + "/" + videoKey)
                .build())
            .outputGroups(outputs)
            .build();
            
        StartJobResponse response = mediaConvert.startJob(job);
        
        // 작업 상태 추적
        jobRepository.save(new TranscodingJob(
            response.jobId(),
            videoKey,
            JobStatus.PROCESSING
        ));
    }
    
    // 2. 트랜스코딩 완료 처리
    @EventListener
    public void handleTranscodingComplete(TranscodingCompleteEvent event) {
        // CDN 캐시 무효화
        invalidateCDNCache(event.getVideoId());
        
        // 메타데이터 업데이트
        updateVideoMetadata(event.getVideoId(), event.getOutputFiles());
        
        // 알림 전송
        notificationService.notifyTranscodingComplete(event.getVideoId());
    }
}
```

## 3. 스트리밍 서비스

```java
@Service
public class StreamingService {
    
    private final CDNService cdnService;
    private final VideoMetadataService metadataService;
    private final BandwidthMonitor bandwidthMonitor;

    // 1. 적응형 스트리밍 매니페스트 생성
    public String generateManifest(String videoId, String deviceType) {
        VideoMetadata metadata = metadataService.getMetadata(videoId);
        
        // HLS 매니페스트 생성
        M3U8Manifest manifest = M3U8Manifest.builder()
            .version(3)
            .targetDuration(10)
            .streams(generateStreamInfo(metadata, deviceType))
            .build();
            
        return manifest.toString();
    }

    // 2. 스트림 품질 선택 로직
    private List<StreamInfo> generateStreamInfo(VideoMetadata metadata, 
                                              String deviceType) {
        List<StreamInfo> streams = new ArrayList<>();
        
        // 디바이스별 최적 품질 설정
        int maxQuality = determineMaxQuality(deviceType);
        
        metadata.getAvailableQualities()
            .stream()
            .filter(quality -> quality.getHeight() <= maxQuality)
            .forEach(quality -> {
                streams.add(StreamInfo.builder()
                    .bandwidth(quality.getBitrate())
                    .resolution(quality.getWidth(), quality.getHeight())
                    .codecs("avc1.64001f,mp4a.40.2")
                    .url(generateStreamUrl(metadata.getId(), quality))
                    .build());
            });
            
        return streams;
    }

    // 3. CDN 스트리밍 URL 생성
    private String generateStreamUrl(String videoId, Quality quality) {
        String baseUrl = cdnService.getBaseUrl();
        String token = generateSecureToken(videoId, quality);
        
        return String.format("%s/videos/%s/%s/index.m3u8?token=%s",
            baseUrl, videoId, quality.getName(), token);
    }
}

@Service
public class AdaptiveBitrateService {
    
    // 4. 클라이언트 대역폭 모니터링
    public void trackClientBandwidth(String sessionId, 
                                   BandwidthSample sample) {
        bandwidthMonitor.recordSample(sessionId, sample);
        
        // 품질 전환 필요성 체크
        if (shouldSwitchQuality(sessionId)) {
            Quality newQuality = determineOptimalQuality(
                sessionId,
                bandwidthMonitor.getAverageBandwidth(sessionId)
            );
            
            notifyQualityChange(sessionId, newQuality);
        }
    }

    // 5. 버퍼링 방지를 위한 선제적 품질 조정
    @Scheduled(fixedRate = 1000)
    public void monitorBuffering() {
        activeSessions.forEach((sessionId, session) -> {
            BufferingMetrics metrics = session.getBufferingMetrics();
            
            if (metrics.getBufferHealth() < BUFFER_THRESHOLD) {
                // 품질 다운그레이드
                downgradeQuality(sessionId);
            }
        });
    }
}

@Service
public class VideoDeliveryService {
    
    // 6. 지역 기반 CDN 라우팅
    public String getOptimalCDNEndpoint(String clientIp) {
        GeoLocation location = geoService.getLocation(clientIp);
        List<CDNEndpoint> endpoints = cdnService.getAvailableEndpoints();
        
        return endpoints.stream()
            .min(Comparator.comparingDouble(endpoint -> 
                calculateLatency(location, endpoint.getLocation())))
            .map(CDNEndpoint::getUrl)
            .orElse(defaultEndpoint);
    }

    // 7. 동시 시청자 관리
    public void manageViewerLoad(String videoId) {
        int currentViewers = getActiveViewers(videoId);
        
        if (currentViewers > VIEWER_THRESHOLD) {
            // 추가 CDN 용량 확보
            cdnService.scaleUpCapacity(videoId);
            
            // 부하 분산
            redistributeViewers(videoId);
        }
    }
}
```

## 4. 캐싱 및 성능 최적화

```java
@Configuration
public class CachingConfig {
    
    // 1. 다층 캐싱 전략
    @Bean
    public CacheManager videoCacheManager() {
        return new LayeredCacheManager(
            new EdgeCache(1000),      // Edge 캐시
            new RegionalCache(),      // 지역 캐시
            new OriginCache()         // 원본 캐시
        );
    }

    // 2. 인기 컨텐츠 프리로딩
    @Scheduled(fixedRate = 3600000)  // 1시간마다
    public void preloadPopularContent() {
        List<String> popularVideos = 
            analyticsService.getTopVideos(100);
            
        popularVideos.forEach(videoId -> {
            List<Quality> qualities = 
                metadataService.getAvailableQualities(videoId);
                
            // 주요 품질의 시작 세그먼트를 프리로드
            qualities.forEach(quality -> 
                cacheManager.preload(videoId, quality));
        });
    }
}

@Service
public class PerformanceOptimizer {
    
    // 3. 성능 모니터링 및 최적화
    @Scheduled(fixedRate = 5000)
    public void optimizePerformance() {
        // CDN 성능 모니터링
        Map<String, PerformanceMetrics> cdnMetrics = 
            monitorCDNPerformance();
            
        // 문제 있는 엣지 노드 식별
        List<String> problematicNodes = 
            identifyProblematicNodes(cdnMetrics);
            
        // 트래픽 재라우팅
        problematicNodes.forEach(node -> 
            rerouteTraffic(node, findHealthyNode()));
            
        // 성능 지표 로깅
        logPerformanceMetrics(cdnMetrics);
    }
}
```

이러한 설계를 통해:

1. 효율적인 동영상 업로드 및 저장
2. 다양한 화질의 트랜스코딩 지원
3. 적응형 스트리밍으로 최적의 시청 경험
4. 효율적인 CDN 활용
5. 성능 최적화 및 모니터링

을 구현할 수 있습니다.

면접관: 대규모 트래픽 상황에서 비용 최적화는 어떻게 하시겠습니까?

## 5. 비용 최적화 전략

```java
@Service
public class CostOptimizationService {

    // 1. 스토리지 계층화
    private class StorageTierManager {
        private final S3Client s3Client;
        
        public void optimizeStorageTiers() {
            // 접근 패턴 분석
            Map<String, AccessPattern> accessPatterns = 
                analyzeVideoAccessPatterns();
                
            accessPatterns.forEach((videoId, pattern) -> {
                if (pattern.getLastAccessedDays() > 30) {
                    // 낮은 빈도 접근 영상 -> Glacier로 이동
                    moveToGlacier(videoId);
                } else if (pattern.getLastAccessedDays() > 7) {
                    // 중간 빈도 접근 영상 -> S3 IA로 이동
                    moveToInfrequentAccess(videoId);
                }
            });
        }
        
        private void moveToGlacier(String videoId) {
            // 메타데이터 유지, 실제 컨텐츠만 이동
            s3Client.copyObject(CopyObjectRequest.builder()
                .sourceBucket(CONTENT_BUCKET)
                .sourceKey(videoId)
                .destinationBucket(GLACIER_BUCKET)
                .storageClass(StorageClass.GLACIER)
                .build());
        }
    }

    // 2. CDN 비용 최적화
    public class CDNCostOptimizer {
        
        @Scheduled(cron = "0 0 * * * *") // 매시간
        public void optimizeCDNUsage() {
            // 지역별 트래픽 분석
            Map<Region, TrafficStats> trafficStats = 
                analyzeRegionalTraffic();
                
            trafficStats.forEach((region, stats) -> {
                if (stats.getCost() > stats.getRevenue()) {
                    // 비용이 수익을 초과하는 지역의 캐시 정책 조정
                    adjustCachingPolicy(region, CachePolicy.AGGRESSIVE);
                    
                    // 저품질 스트림으로 기본 설정 변경
                    adjustDefaultQuality(region, Quality.MEDIUM);
                }
            });
        }
        
        private void adjustCachingPolicy(Region region, CachePolicy policy) {
            CloudFrontDistribution distribution = 
                getDistributionForRegion(region);
                
            // TTL 및 캐시 동작 조정
            distribution.updateCacheBehavior(behavior -> 
                behavior.withTTL(policy.getTtl())
                       .withCompression(true)
                       .withQueryStringForwarding(false));
        }
    }

    // 3. 트랜스코딩 최적화
    public class TranscodingOptimizer {
        
        public void optimizeTranscodingSettings(String videoId, 
                                              VideoMetadata metadata) {
            // 컨텐츠 타입별 최적 설정
            if (metadata.getType() == ContentType.ANIMATION) {
                // 애니메이션용 최적화 설정
                return TranscodingSettings.builder()
                    .keyframeInterval(120)
                    .bitrateLadder(BitrateOptimizer.forAnimation())
                    .build();
            } else if (metadata.getType() == ContentType.LECTURE) {
                // 강의용 최적화 설정
                return TranscodingSettings.builder()
                    .keyframeInterval(240)
                    .bitrateLadder(BitrateOptimizer.forLecture())
                    .build();
            }
        }
        
        // 병렬 트랜스코딩 최적화
        public void scheduleTranscoding(List<String> videoIds) {
            // 우선순위 기반 스케줄링
            PriorityQueue<TranscodingJob> queue = new PriorityQueue<>(
                Comparator.comparingInt(this::calculatePriority));
                
            queue.addAll(createJobs(videoIds));
            
            // 배치 처리로 비용 최적화
            while (!queue.isEmpty()) {
                List<TranscodingJob> batch = 
                    getBatchWithinCostLimit(queue);
                executeTranscodingBatch(batch);
            }
        }
    }

    // 4. 비용 모니터링 및 알림
    @Service
    public class CostMonitoringService {
        
        private final AlertService alertService;
        
        @Scheduled(cron = "0 0 * * * *")
        public void monitorCosts() {
            CostBreakdown costs = calculateCurrentCosts();
            
            // 예산 초과 확인
            if (costs.isOverBudget()) {
                // 자동 비용 절감 조치
                applyCostReductionMeasures(costs);
                
                // 관리자 알림
                alertService.sendAlert(AlertLevel.HIGH,
                    "Budget exceeded: " + costs.getDetails());
            }
        }
        
        private void applyCostReductionMeasures(CostBreakdown costs) {
            if (costs.getCdnCosts() > THRESHOLD) {
                // CDN 비용 절감
                reduceCDNCosts();
            }
            
            if (costs.getStorageCosts() > THRESHOLD) {
                // 스토리지 비용 절감
                reduceStorageCosts();
            }
            
            if (costs.getTranscodingCosts() > THRESHOLD) {
                // 트랜스코딩 비용 절감
                reduceTranscodingCosts();
            }
        }
    }
}
```

이러한 비용 최적화 전략을 통해:

1. 스토리지 비용 최적화
    - 접근 빈도에 따른 스토리지 계층화
    - 오래된 컨텐츠의 자동 아카이빙
    - 중복 제거 및 압축

2. CDN 비용 최적화
    - 지역별 트래픽 분석 및 최적화
    - 캐싱 정책 최적화
    - 품질 설정 자동 조정

3. 트랜스코딩 비용 최적화
    - 컨텐츠 타입별 최적 설정
    - 배치 처리를 통한 비용 절감
    - 우선순위 기반 스케줄링

4. 실시간 비용 모니터링
    - 비용 초과 시 자동 알림
    - 자동화된 비용 절감 조치
    - 상세한 비용 분석 및 리포팅

이를 통해 서비스 품질을 유지하면서도 효율적인 비용 관리가 가능합니다.
