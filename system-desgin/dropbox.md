# 파일 공유 시스템 설계 (Dropbox 유형)

면접관: "대용량 파일 공유 및 동기화 시스템을 설계해주세요. 파일 업로드/다운로드, 동기화, 공유 기능이 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 예상 사용자 수와 평균 저장 용량은 어느 정도인가요?
2. 파일 크기 제한과 지원해야 할 파일 유형은 어떻게 되나요?
3. 실시간 동기화가 필요한가요?
4. 버전 관리 기능이 필요한가요?

면접관:
1. DAU 100만, 사용자당 평균 10GB 저장공간
2. 단일 파일 최대 5GB, 모든 파일 유형 지원
3. 실시간 동기화 필요, 여러 기기 간 동기화 지원
4. 파일 버전 관리 필요 (최근 30일)

## 1. 파일 처리 시스템

```java
@Service
public class FileProcessingService {
    
    private final StorageService storageService;
    private final ChunkService chunkService;
    private final MetadataService metadataService;

    // 1. 파일 청크 업로드
    public UploadResult uploadFile(MultipartFile file, String userId) {
        // 파일 메타데이터 생성
        FileMetadata metadata = createFileMetadata(file, userId);
        
        // 파일을 청크로 분할
        List<FileChunk> chunks = chunkService.splitFile(file);
        
        // 병렬 업로드
        CompletableFuture<ChunkUploadResult>[] uploads = chunks.stream()
            .map(chunk -> CompletableFuture.supplyAsync(() ->
                uploadChunk(chunk, metadata)))
            .toArray(CompletableFuture[]::new);
            
        // 모든 청크 업로드 완료 대기
        CompletableFuture.allOf(uploads).join();
        
        // 메타데이터 저장
        metadataService.saveMetadata(metadata);
        
        return new UploadResult(metadata.getFileId());
    }

    // 2. 파일 동기화 처리
    private class FileSync {
        private final BlockingQueue<FileSyncEvent> syncQueue;
        private final Map<String, FileState> fileStates;

        public void handleFileChange(FileSyncEvent event) {
            switch (event.getType()) {
                case MODIFIED:
                    syncModifiedFile(event);
                    break;
                case DELETED:
                    propagateDelete(event);
                    break;
                case RENAMED:
                    handleRename(event);
                    break;
            }
        }

        private void syncModifiedFile(FileSyncEvent event) {
            // 변경된 블록만 식별
            List<FileBlock> changedBlocks = 
                identifyChangedBlocks(event.getFileId());
            
            // 변경된 블록만 동기화
            changedBlocks.forEach(block -> 
                syncBlock(block, event.getDevices()));
        }
    }
}
```

## 2. 파일 버전 관리 및 충돌 해결

```java
@Service
@Slf4j
public class VersionControlService {
    
    private final VersionRepository versionRepository;
    private final ConflictResolver conflictResolver;
    private final NotificationService notificationService;

    // 1. 버전 관리
    public Version createVersion(String fileId, FileMetadata metadata) {
        Version version = Version.builder()
            .fileId(fileId)
            .versionNumber(getNextVersionNumber(fileId))
            .timestamp(Instant.now())
            .metadata(metadata)
            .chunks(getFileChunks(fileId))
            .hash(calculateFileHash(fileId))
            .build();
            
        versionRepository.save(version);
        
        // 오래된 버전 정리
        cleanupOldVersions(fileId);
        
        return version;
    }

    // 2. 충돌 감지 및 해결
    public void handleConflict(String fileId, List<FileChange> changes) {
        // 충돌 감지
        if (detectConflict(changes)) {
            ConflictResolution resolution;
            
            if (canAutoResolve(changes)) {
                // 자동 충돌 해결
                resolution = conflictResolver.autoResolve(changes);
            } else {
                // 사용자에게 충돌 알림
                notificationService.notifyConflict(fileId, changes);
                resolution = conflictResolver.createConflictCopy(changes);
            }
            
            applyResolution(resolution);
        }
    }

    private boolean detectConflict(List<FileChange> changes) {
        Map<String, Long> lastModified = new HashMap<>();
        
        for (FileChange change : changes) {
            Long previousModification = 
                lastModified.put(change.getPath(), change.getTimestamp());
                
            if (previousModification != null && 
                !change.getBaseVersion().equals(getLatestVersion(change.getFileId()))) {
                return true;
            }
        }
        return false;
    }
}

@Service
public class ConflictResolver {
    
    // 3. 충돌 해결 전략
    public ConflictResolution autoResolve(List<FileChange> changes) {
        // 타임스탬프 기반 해결
        if (canResolveByTimestamp(changes)) {
            return resolveByTimestamp(changes);
        }
        
        // 컨텐츠 기반 해결
        if (canResolveByContent(changes)) {
            return resolveByContent(changes);
        }
        
        // 병합 가능한 경우
        if (canMergeChanges(changes)) {
            return mergeChanges(changes);
        }
        
        // 해결 불가능한 경우 충돌 복사본 생성
        return createConflictCopy(changes);
    }

    // 4. 버전 관리 정책
    @Scheduled(fixedRate = 86400000) // 매일 실행
    public void applyRetentionPolicy() {
        // 30일 이상 된 버전 찾기
        List<Version> oldVersions = versionRepository
            .findVersionsOlderThan(Duration.ofDays(30));
            
        for (Version version : oldVersions) {
            // 중요 버전은 유지
            if (!version.isSignificant()) {
                // 버전 아카이빙 또는 삭제
                archiveVersion(version);
            }
        }
    }

    // 5. 차이점 감지
    public class DiffCalculator {
        public FileDiff calculateDiff(String fileId, 
                                    String oldVersion, 
                                    String newVersion) {
            byte[] oldContent = getVersionContent(fileId, oldVersion);
            byte[] newContent = getVersionContent(fileId, newVersion);
            
            // 바이트 레벨 비교
            List<DiffBlock> diffBlocks = new ArrayList<>();
            int blockSize = 4096; // 4KB 블록
            
            for (int i = 0; i < oldContent.length; i += blockSize) {
                byte[] oldBlock = Arrays.copyOfRange(
                    oldContent, i, Math.min(i + blockSize, oldContent.length));
                byte[] newBlock = Arrays.copyOfRange(
                    newContent, i, Math.min(i + blockSize, newContent.length));
                
                if (!Arrays.equals(oldBlock, newBlock)) {
                    diffBlocks.add(new DiffBlock(i, newBlock));
                }
            }
            
            return new FileDiff(fileId, oldVersion, newVersion, diffBlocks);
        }
    }
}

// 6. 병합 전략
@Service
public class MergeStrategy {
    
    public MergeResult mergeChanges(FileChange change1, FileChange change2) {
        // 공통 조상 버전 찾기
        Version commonAncestor = findCommonAncestor(
            change1.getBaseVersion(), 
            change2.getBaseVersion()
        );
        
        // 변경 사항 분석
        List<Change> changes1 = analyzeChanges(commonAncestor, change1);
        List<Change> changes2 = analyzeChanges(commonAncestor, change2);
        
        // 충돌 영역 식별
        List<Conflict> conflicts = findConflicts(changes1, changes2);
        
        if (conflicts.isEmpty()) {
            // 충돌 없는 경우 자동 병합
            return autoMerge(changes1, changes2);
        } else {
            // 충돌 있는 경우 사용자 개입 필요
            return createConflictMarkers(conflicts);
        }
    }
    
    private MergeResult autoMerge(List<Change> changes1, 
                                 List<Change> changes2) {
        // 변경 사항 정렬 (위치 기반)
        TreeMap<Integer, Change> mergedChanges = new TreeMap<>();
        
        // 양쪽 변경 사항 통합
        changes1.forEach(change -> 
            mergedChanges.put(change.getPosition(), change));
        changes2.forEach(change -> 
            mergedChanges.put(change.getPosition(), change));
            
        // 최종 버전 생성
        return applyChanges(mergedChanges.values());
    }
}
```

이러한 버전 관리 및 충돌 해결 시스템을 통해:

1. 효율적인 버전 이력 관리
    - 증분 버전 관리로 저장 공간 최적화
    - 중요 버전 보존
    - 자동 정리 정책

2. 지능적인 충돌 해결
    - 자동 충돌 감지
    - 다양한 해결 전략
    - 사용자 개입 최소화

3. 효율적인 동기화
    - 변경된 부분만 동기화
    - 병렬 처리로 성능 최적화
    - 네트워크 사용 최소화

를 구현할 수 있습니다.

면접관: 대규모 사용자의 실시간 동기화 처리는 어떻게 구현하시겠습니까?

## 3. 실시간 동기화 시스템

```java
@Service
public class RealTimeSyncService {
    
    private final WebSocketHandler webSocketHandler;
    private final SyncQueueManager syncQueueManager;
    private final DeviceRegistry deviceRegistry;

    // 1. WebSocket 연결 관리
    @Component
    public class SyncWebSocketHandler extends TextWebSocketHandler {
        
        private final ConcurrentMap<String, WebSocketSession> deviceSessions = 
            new ConcurrentHashMap<>();
        
        @Override
        public void afterConnectionEstablished(WebSocketSession session) {
            String deviceId = extractDeviceId(session);
            deviceSessions.put(deviceId, session);
            deviceRegistry.registerDevice(deviceId);
            
            // 초기 동기화 시작
            initiateInitialSync(deviceId);
        }
        
        private void initiateInitialSync(String deviceId) {
            // 마지막 동기화 시점 이후의 변경사항 조회
            List<Change> pendingChanges = 
                getPendingChanges(deviceId);
            
            // 변경사항 전송
            syncQueueManager.queueChanges(deviceId, pendingChanges);
        }
    }

    // 2. 변경사항 전파
    public class ChangePropagate {
        
        private final LoadBalancer loadBalancer;
        
        public void propagateChange(FileChange change) {
            // 영향받는 디바이스 조회
            Set<String> affectedDevices = 
                deviceRegistry.getAffectedDevices(change.getFileId());
                
            // 변경사항 분배
            int batchSize = 1000;
            Lists.partition(new ArrayList<>(affectedDevices), batchSize)
                .forEach(deviceBatch -> 
                    CompletableFuture.runAsync(() -> 
                        sendChangesToDevices(deviceBatch, change)));
        }
        
        private void sendChangesToDevices(List<String> devices, 
                                        FileChange change) {
            devices.forEach(deviceId -> {
                try {
                    WebSocketSession session = 
                        webSocketHandler.getSession(deviceId);
                        
                    if (session != null && session.isOpen()) {
                        session.sendMessage(new TextMessage(
                            serializeChange(change)));
                    } else {
                        // 오프라인 디바이스는 큐에 저장
                        syncQueueManager.queueChange(deviceId, change);
                    }
                } catch (Exception e) {
                    log.error("Failed to send change to device: " + 
                        deviceId, e);
                }
            });
        }
    }

    // 3. 동기화 큐 관리
    @Service
    public class SyncQueueManager {
        
        private final Map<String, PriorityQueue<Change>> deviceQueues = 
            new ConcurrentHashMap<>();
            
        public void queueChange(String deviceId, Change change) {
            deviceQueues.computeIfAbsent(deviceId, 
                k -> new PriorityQueue<>(changeComparator))
                .offer(change);
                
            // 큐 크기 제한 확인
            enforceQueueSizeLimit(deviceId);
        }
        
        @Scheduled(fixedRate = 60000) // 1분마다
        public void processQueues() {
            deviceQueues.forEach((deviceId, queue) -> {
                if (deviceRegistry.isDeviceOnline(deviceId)) {
                    drainQueue(deviceId, queue);
                }
            });
        }
        
        private void drainQueue(String deviceId, 
                              Queue<Change> queue) {
            List<Change> batch = new ArrayList<>();
            Change change;
            
            // 최대 100개씩 배치 처리
            while ((change = queue.poll()) != null && 
                   batch.size() < 100) {
                batch.add(change);
            }
            
            if (!batch.isEmpty()) {
                sendChangeBatch(deviceId, batch);
            }
        }
    }

    // 4. 네트워크 최적화
    @Component
    public class NetworkOptimizer {
        
        public byte[] optimizeChangePayload(Change change) {
            // 델타 압축 적용
            byte[] deltaCompressed = 
                deltaCompression.compress(change.getContent());
            
            // 일반 압축 적용
            return generalCompression.compress(deltaCompressed);
        }
        
        public void adjustSyncFrequency(String deviceId) {
            DeviceMetrics metrics = 
                deviceRegistry.getDeviceMetrics(deviceId);
                
            // 네트워크 상태에 따라 동기화 주기 조정
            if (metrics.getNetworkQuality() < THRESHOLD) {
                increaseSyncInterval(deviceId);
            } else {
                decreaseSyncInterval(deviceId);
            }
        }
    }
}
```

이러한 실시간 동기화 시스템을 통해:

1. 효율적인 연결 관리
    - WebSocket 기반 실시간 통신
    - 연결 상태 추적
    - 자동 재연결 처리

2. 확장성 있는 변경사항 전파
    - 배치 처리로 성능 최적화
    - 부하 분산
    - 실패 처리 및 재시도

3. 신뢰성 있는 동기화 큐
    - 오프라인 디바이스 지원
    - 우선순위 기반 처리
    - 큐 크기 관리

4. 네트워크 효율성
    - 델타 압축
    - 배치 처리
    - 적응형 동기화 주기

를 구현할 수 있습니다.

이를 통해 대규모 사용자의 실시간 동기화를 효율적으로 처리할 수 있습니다.