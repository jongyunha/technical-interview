# 메시징 시스템 설계 (WhatsApp/Slack 유사 시스템)

면접관: "실시간 메시징 시스템을 설계해주세요. 다음 기능이 필요합니다:
1. 1:1 채팅
2. 그룹 채팅
3. 온라인 상태 표시
4. 메시지 저장 및 동기화
5. 푸시 알림"

지원자: 먼저 몇 가지 요구사항을 확인하고 싶습니다.

1. 예상 사용자 수와 메시지 처리량은 어느 정도인가요?
2. 메시지 저장 기간과 크기 제한은 있나요?
3. 어떤 종류의 메시지를 지원해야 하나요? (텍스트, 이미지, 파일 등)
4. 배달 확인이나 읽음 확인 기능이 필요한가요?

면접관:
1. DAU 100만 명, 초당 평균 10,000건의 메시지
2. 메시지는 영구 보관, 파일은 30일, 크기는 메시지당 최대 100KB
3. 텍스트, 이미지, 파일 모두 지원 필요
4. 메시지 배달 확인과 읽음 확인 모두 필요

지원자: 알겠습니다. 전체적인 아키텍처를 설계해보겠습니다.

## 1. 전체 시스템 아키텍처

```plaintext
[클라이언트]
    ↓
[API Gateway / Load Balancer]
    ↓
[Connection Manager (WebSocket)] -- [Presence Service]
    ↓
[Chat Service] -- [Push Notification Service]
    ↓
[Message Queue (Kafka)]
    ↓
[Message Store (Cassandra)] -- [User Store (PostgreSQL)]
    ↓
[File Store (S3/CDN)]
```

## 2. WebSocket 연결 관리
```java
@Service
public class ConnectionManager {
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
    
    @OnOpen
    public void onConnect(WebSocketSession session, @PathParam("userId") String userId) {
        sessions.put(userId, session);
        presenceService.updateStatus(userId, Status.ONLINE);
    }
    
    @OnMessage
    public void onMessage(String message, WebSocketSession session) {
        MessageDTO msg = parseMessage(message);
        chatService.processMessage(msg);
    }
    
    @OnClose
    public void onDisconnect(WebSocketSession session, @PathParam("userId") String userId) {
        sessions.remove(userId);
        presenceService.updateStatus(userId, Status.OFFLINE);
    }
}
```

## 3. 메시지 처리
```java
@Service
public class ChatService {
    private final KafkaTemplate<String, Message> kafkaTemplate;
    private final MessageRepository messageRepository;
    private final PushNotificationService pushService;
    
    public void processMessage(Message message) {
        // 1. 메시지 유효성 검증
        validateMessage(message);
        
        // 2. 메시지 저장
        messageRepository.save(message);
        
        // 3. 메시지 전달
        deliverMessage(message);
        
        // 4. 오프라인 사용자에게 푸시 알림
        if (!isUserOnline(message.getReceiverId())) {
            pushService.sendPush(message);
        }
    }
    
    private void deliverMessage(Message message) {
        // Kafka로 메시지 발행
        kafkaTemplate.send("chat-messages", message);
        
        // 온라인 사용자에게 즉시 전달
        WebSocketSession session = connectionManager.getSession(message.getReceiverId());
        if (session != null) {
            session.sendMessage(new TextMessage(message.toJson()));
        }
    }
}
```
## 4. 메시지 저장소 설계

```java
// 1. 메시지 스키마 (Cassandra)
@Table("messages")
public class Message {
    @PrimaryKey
    private MessageId id;  // 복합키: chat_id + timestamp
    
    private String senderId;
    private String chatId;      // 1:1 또는 그룹 채팅 ID
    private String content;
    private MessageType type;   // TEXT, IMAGE, FILE
    private long timestamp;
    private MessageStatus status; // SENT, DELIVERED, READ
    
    @Data
    public static class MessageId {
        private String chatId;
        private long timestamp;  // 역순 정렬을 위해 사용
    }
}

// 2. 메시지 저장소
@Repository
public class MessageRepository {
    private final CassandraTemplate cassandraTemplate;
    private final S3Client s3Client;
    
    public void saveMessage(Message message) {
        if (message.getType() != MessageType.TEXT) {
            // 미디어 파일 저장
            String fileUrl = uploadToS3(message.getContent());
            message.setContent(fileUrl);
        }
        
        cassandraTemplate.insert(message);
    }
    
    public List<Message> getMessages(String chatId, long lastMessageTimestamp, int limit) {
        return cassandraTemplate.select(Query.builder()
            .partition("chat_id = ?", chatId)
            .range("timestamp < ?", lastMessageTimestamp)
            .limit(limit)
            .build(), Message.class);
    }
}
```

## 5. 실시간 상태 관리

```java
@Service
public class PresenceService {
    private final RedisTemplate<String, UserStatus> redisTemplate;
    private final UserStatusPublisher statusPublisher;
    
    public void updateStatus(String userId, Status status) {
        // Redis에 상태 저장 (TTL 설정)
        redisTemplate.opsForValue().set(
            "presence:" + userId, 
            new UserStatus(userId, status),
            30,
            TimeUnit.MINUTES
        );
        
        // 상태 변경 이벤트 발행
        statusPublisher.publish(new StatusChangeEvent(userId, status));
    }
    
    // 사용자 상태 조회 (bulk)
    public Map<String, Status> getUserStatuses(List<String> userIds) {
        List<String> keys = userIds.stream()
            .map(id -> "presence:" + id)
            .collect(Collectors.toList());
            
        return redisTemplate.opsForValue()
            .multiGet(keys).stream()
            .collect(Collectors.toMap(
                UserStatus::getUserId,
                UserStatus::getStatus
            ));
    }
    
    // 하트비트 처리
    @Scheduled(fixedRate = 30000)  // 30초마다
    public void processHeartbeats() {
        // 만료된 상태 처리
        Set<String> expiredUsers = findExpiredUsers();
        expiredUsers.forEach(userId -> 
            updateStatus(userId, Status.OFFLINE));
    }
}
```

## 6. 그룹 채팅 관리

```java
@Service
public class GroupChatService {
    private final RedisTemplate<String, Set<String>> redisTemplate;
    private final ChatRepository chatRepository;
    
    public void createGroup(String groupId, String name, List<String> members) {
        // 1. 그룹 메타데이터 저장
        ChatGroup group = ChatGroup.builder()
            .id(groupId)
            .name(name)
            .createdAt(Instant.now())
            .build();
        chatRepository.saveGroup(group);
        
        // 2. 멤버 정보 캐싱
        redisTemplate.opsForSet()
            .add("group:" + groupId, members.toArray(new String[0]));
    }
    
    public void sendGroupMessage(Message message) {
        Set<String> members = getGroupMembers(message.getChatId());
        
        // 각 멤버에게 메시지 전달
        CompletableFuture.allOf(
            members.stream()
                .map(memberId -> {
                    Message copy = message.toBuilder()
                        .receiverId(memberId)
                        .build();
                    return chatService.processMessage(copy);
                })
                .toArray(CompletableFuture[]::new)
        ).join();
    }
    
    // 그룹 멤버 관리
    public void addMember(String groupId, String userId) {
        redisTemplate.opsForSet().add("group:" + groupId, userId);
        notifyMemberChange(groupId, userId, "ADDED");
    }
    
    public void removeMember(String groupId, String userId) {
        redisTemplate.opsForSet().remove("group:" + groupId, userId);
        notifyMemberChange(groupId, userId, "REMOVED");
    }
}
```

## 7. 푸시 알림 시스템

```java
@Service
public class PushNotificationService {
    private final FirebaseMessaging firebaseMessaging;
    private final APNSClient apnsClient;
    private final DeviceRepository deviceRepository;
    
    public void sendPush(Message message) {
        List<Device> devices = deviceRepository
            .findByUserId(message.getReceiverId());
            
        devices.forEach(device -> {
            try {
                switch (device.getPlatform()) {
                    case ANDROID:
                        sendFirebaseNotification(device, message);
                        break;
                    case IOS:
                        sendAPNSNotification(device, message);
                        break;
                }
            } catch (Exception e) {
                log.error("Push notification failed", e);
                // 실패한 알림 재시도 큐에 추가
                retryQueue.add(new RetryNotification(device, message));
            }
        });
    }
    
    // 알림 템플릿 관리
    private Notification createNotification(Message message) {
        NotificationTemplate template = switch (message.getType()) {
            case TEXT -> new TextMessageTemplate();
            case IMAGE -> new ImageMessageTemplate();
            case FILE -> new FileMessageTemplate();
        };
        
        return template.create(message);
    }
}
```

이러한 설계를 통해:

1. 실시간 메시지 전송
2. 메시지 영속성
3. 오프라인 사용자 지원
4. 그룹 채팅 기능
5. 상태 관리
6. 알림 처리

등의 기능을 구현할 수 있습니다.

면접관: 시스템의 확장성 측면에서 잠재적인 병목 지점과 그 해결 방안은 무엇인가요?

## 8. 확장성 전략

### 8.1 WebSocket 연결 확장
```java
@Configuration
public class WebSocketScalingConfig {
    
    // 1. 연결 관리 클러스터
    @Bean
    public ConnectionCluster connectionCluster() {
        return ConnectionCluster.builder()
            .withNodes(List.of(
                new Node("ws-1", "10.0.1.1"),
                new Node("ws-2", "10.0.1.2")
            ))
            .withStickySessions(true)  // 같은 사용자는 같은 노드로
            .build();
    }
    
    // 2. 연결 부하 분산
    @Bean
    public ConnectionLoadBalancer loadBalancer() {
        return new ConsistentHashLoadBalancer(
            metric -> metric.getActiveConnections(),
            threshold -> threshold < MAX_CONNECTIONS_PER_NODE
        );
    }
}
```

### 8.2 메시지 처리 확장
```java
@Service
public class MessageProcessingService {
    
    // 1. 샤딩 전략
    private String determineShardKey(Message message) {
        if (message.isGroupMessage()) {
            return "group:" + message.getChatId();  // 그룹별 샤딩
        }
        return "user:" + message.getSenderId();     // 사용자별 샤딩
    }
    
    // 2. 병렬 처리
    public void processMessages(List<Message> messages) {
        // 메시지를 샤드별로 그룹화
        Map<String, List<Message>> shardedMessages = messages.stream()
            .collect(Collectors.groupingBy(this::determineShardKey));
            
        // 각 샤드별로 병렬 처리
        CompletableFuture.allOf(
            shardedMessages.entrySet().stream()
                .map(entry -> CompletableFuture.runAsync(() ->
                    processMessageBatch(entry.getValue()),
                    messageProcessorExecutor))
                .toArray(CompletableFuture[]::new)
        ).join();
    }
    
    // 3. 배치 처리
    @Scheduled(fixedRate = 100)  // 100ms마다
    public void processBatch() {
        List<Message> batch = messageBatcher.getBatch();
        if (!batch.isEmpty()) {
            processMessages(batch);
        }
    }
}
```

### 8.3 데이터 저장소 확장
```java
@Configuration
public class StorageScalingConfig {
    
    // 1. 읽기 복제본 관리
    @Bean
    public ReadReplicaManager replicaManager() {
        return ReadReplicaManager.builder()
            .withReplicaNodes(getReplicaNodes())
            .withLoadBalancing(LoadBalancingStrategy.LEAST_CONNECTIONS)
            .withFailover(true)
            .build();
    }
    
    // 2. 캐시 계층화
    @Bean
    public CacheManager cacheManager() {
        return CacheManager.builder()
            .withL1Cache(new LocalCache(1000))    // 로컬 캐시
            .withL2Cache(new RedisCache())        // 분산 캐시
            .withEvictionPolicy(EvictionPolicy.LRU)
            .build();
    }
    
    // 3. 핫 데이터 관리
    @Service
    public class HotDataManager {
        public void manageHotData() {
            // 활성 채팅방 식별
            Set<String> activeChats = identifyActiveChats();
            
            // 핫 데이터 프리페치
            activeChats.forEach(chatId -> 
                prefetchRecentMessages(chatId));
                
            // 콜드 데이터 아카이빙
            archiveColdData();
        }
    }
}
```

### 8.4 부하 모니터링 및 자동 스케일링
```java
@Service
public class ScalingMonitor {
    
    private final MetricRegistry metrics;
    private final AutoScaler autoScaler;
    
    // 1. 시스템 메트릭 수집
    @Scheduled(fixedRate = 5000)  // 5초마다
    public void collectMetrics() {
        metrics.gauge("websocket.connections", () -> 
            connectionManager.getActiveConnections());
            
        metrics.gauge("message.processing.rate", () ->
            messageProcessor.getProcessingRate());
            
        metrics.gauge("storage.usage", () ->
            storageManager.getUsagePercentage());
    }
    
    // 2. 자동 스케일링 결정
    @Scheduled(fixedRate = 30000)  // 30초마다
    public void checkScaling() {
        ScalingDecision decision = ScalingAnalyzer.analyze(
            metrics.getMetrics(),
            ScalingPolicy.builder()
                .withCpuThreshold(70)
                .withMemoryThreshold(80)
                .withConnectionThreshold(5000)
                .build()
        );
        
        if (decision.requiresScaling()) {
            autoScaler.scale(decision);
        }
    }
    
    // 3. 성능 병목 감지
    @Scheduled(fixedRate = 10000)  // 10초마다
    public void detectBottlenecks() {
        List<BottleneckReport> reports = BottleneckDetector
            .analyze(metrics.getMetrics());
            
        reports.forEach(report -> {
            log.warn("Bottleneck detected: {}", report);
            alertOperations(report);
            applyMitigation(report);
        });
    }
}
```

### 8.5 장애 대응 전략
```java
@Service
public class FailureHandlingService {
    
    // 1. 서킷브레이커 구성
    @Bean
    public CircuitBreakerFactory circuitBreakerFactory() {
        return new CircuitBreakerFactory()
            .withFailureThreshold(5)
            .withTimeout(Duration.ofSeconds(1))
            .withResetTimeout(Duration.ofSeconds(30));
    }
    
    // 2. 장애 격리
    @Service
    public class FailureIsolator {
        public void isolateFailure(Failure failure) {
            // 실패한 노드 격리
            nodeManager.isolateNode(failure.getNodeId());
            
            // 트래픽 재분배
            trafficManager.redistributeTraffic(
                failure.getNodeId(), 
                getHealthyNodes()
            );
            
            // 복구 프로세스 시작
            recoveryManager.startRecovery(failure);
        }
    }
}
```

이러한 확장성 전략을 통해:
1. 연결 수의 선형적 확장
2. 메시지 처리량 향상
3. 데이터 접근 최적화
4. 자동화된 스케일링
5. 효과적인 장애 처리

를 달성할 수 있습니다. 특히 중요한 점은 각 컴포넌트가 독립적으로 확장 가능하도록 설계했다는 것입니다.
