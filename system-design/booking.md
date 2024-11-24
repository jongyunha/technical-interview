# 온라인 예약 시스템 설계 (호텔/레스토랑)

면접관: "호텔이나 레스토랑과 같은 온라인 예약 시스템을 설계해주세요. 실시간 예약, 동시성 제어, 예약 확정 기능이 필요합니다."

지원자: 네, 몇 가지 요구사항을 확인하고 싶습니다.

1. 예상 사용자 수와 일일 예약 건수는 어느 정도인가요?
2. 예약 가능 시간 단위와 취소 정책은 어떻게 되나요?
3. 동시 예약 처리 시 우선순위 규칙이 있나요?
4. 결제 시스템 연동이 필요한가요?

면접관:
1. DAU 50만, 일일 예약 10만 건
2. 1시간 단위 예약, 24시간 전 취소 가능
3. 선착순 기본, VIP 사용자 우선순위 지원
4. 결제 시스템 연동 필요 (보증금 시스템)

## 1. 예약 시스템 핵심 로직

```java
@Service
@Slf4j
public class ReservationService {

    private final ReservationRepository reservationRepository;
    private final InventoryService inventoryService;
    private final LockService lockService;
    private final PaymentService paymentService;

    // 1. 예약 생성 프로세스
    @Transactional
    public ReservationResult createReservation(ReservationRequest request) {
        String lockKey = generateLockKey(
                request.getResourceId(),
                request.getTimeSlot()
        );

        try {
            // 분산 락 획득
            boolean locked = lockService.acquire(lockKey,
                    Duration.ofSeconds(10));

            if (!locked) {
                throw new ConcurrentBookingException(
                        "Resource is being booked by another user");
            }

            // 재고 확인
            if (!inventoryService.checkAvailability(
                    request.getResourceId(),
                    request.getTimeSlot())) {
                throw new NoAvailabilityException("No availability");
            }

            // 예약금 결제 처리
            PaymentResult payment = paymentService.processDeposit(
                    request.getUserId(),
                    request.getDepositAmount()
            );

            // 예약 생성
            Reservation reservation = Reservation.builder()
                    .userId(request.getUserId())
                    .resourceId(request.getResourceId())
                    .timeSlot(request.getTimeSlot())
                    .status(ReservationStatus.CONFIRMED)
                    .paymentId(payment.getPaymentId())
                    .build();

            // 재고 차감
            inventoryService.decrementInventory(
                    request.getResourceId(),
                    request.getTimeSlot()
            );

            return reservationRepository.save(reservation);

        } catch (Exception e) {
            // 결제 취소 등 롤백 처리
            handleReservationFailure(request, e);
            throw e;
        } finally {
            lockService.release(lockKey);
        }
    }

    // 2. 동시성 제어
    @Component
    public class DistributedLockService {
        private final RedisTemplate<String, String> redisTemplate;

        public boolean acquire(String key, Duration timeout) {
            String token = UUID.randomUUID().toString();

            return redisTemplate.opsForValue()
                    .setIfAbsent(key, token, timeout);
        }

        public void release(String key) {
            redisTemplate.delete(key);
        }
    }
}
```

## 2. 재고 관리 및 예약 확인 시스템

```java
@Service
public class InventoryService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final InventoryRepository inventoryRepository;

    // 1. 재고 관리
    public class InventoryManager {
        
        // 재고 초기화 (일별/주별 단위)
        @Scheduled(cron = "0 0 0 * * *") // 매일 자정
        public void initializeInventory() {
            List<Resource> resources = resourceRepository.findAll();
            
            resources.forEach(resource -> {
                // 향후 30일치 재고 초기화
                LocalDate startDate = LocalDate.now();
                LocalDate endDate = startDate.plusDays(30);
                
                for (LocalDate date = startDate; 
                     date.isBefore(endDate); 
                     date = date.plusDays(1)) {
                    
                    initializeDailyInventory(resource, date);
                }
            });
        }

        private void initializeDailyInventory(
            Resource resource, LocalDate date) {
            
            // 시간대별 재고 설정
            resource.getOperatingHours().forEach(hour -> {
                String inventoryKey = generateInventoryKey(
                    resource.getId(), 
                    date, 
                    hour
                );
                
                redisTemplate.opsForValue().set(
                    inventoryKey,
                    String.valueOf(resource.getCapacity()),
                    Duration.ofDays(31)
                );
            });
        }
    }

    // 2. 재고 확인 및 업데이트
    public class InventoryUpdater {
        
        public boolean checkAndDecrementInventory(
            String resourceId, 
            LocalDateTime timeSlot) {
            
            String inventoryKey = generateInventoryKey(
                resourceId, 
                timeSlot.toLocalDate(), 
                timeSlot.getHour()
            );

            // Redis Transaction 사용
            return redisTemplate.execute(new SessionCallback<Boolean>() {
                @Override
                public Boolean execute(RedisOperations operations) {
                    operations.watch(inventoryKey);
                    
                    String currentValue = operations.opsForValue()
                        .get(inventoryKey);
                        
                    int currentInventory = 
                        Integer.parseInt(currentValue);
                        
                    if (currentInventory <= 0) {
                        return false;
                    }

                    operations.multi();
                    operations.opsForValue().set(
                        inventoryKey, 
                        String.valueOf(currentInventory - 1)
                    );
                    
                    return !operations.exec().isEmpty();
                }
            });
        }

        // 재고 복구 (예약 취소 시)
        public void incrementInventory(
            String resourceId, 
            LocalDateTime timeSlot) {
            
            String inventoryKey = generateInventoryKey(
                resourceId, 
                timeSlot.toLocalDate(), 
                timeSlot.getHour()
            );

            redisTemplate.opsForValue()
                .increment(inventoryKey);
        }
    }

    // 3. 오버부킹 방지
    public class OverbookingPrevention {
        
        private final LoadingCache<String, Integer> inventoryCache;
        
        public OverbookingPrevention() {
            this.inventoryCache = Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(1))
                .build(this::loadCurrentInventory);
        }

        public boolean isOverbookingRisk(
            String resourceId, 
            LocalDateTime timeSlot) {
            
            String inventoryKey = generateInventoryKey(
                resourceId, 
                timeSlot.toLocalDate(), 
                timeSlot.getHour()
            );

            // 현재 재고 확인
            int currentInventory = inventoryCache.get(inventoryKey);
            
            // 진행 중인 예약 건수 확인
            int pendingReservations = getPendingReservationsCount(
                resourceId, 
                timeSlot
            );

            return (currentInventory - pendingReservations) <= 0;
        }
    }

    // 4. 예약 상태 관리
    @Service
    public class ReservationStateManager {
        
        private final KafkaTemplate<String, ReservationEvent> kafka;
        
        public void handleReservationStateChange(
            Reservation reservation, 
            ReservationStatus newStatus) {
            
            // 상태 변경 기록
            ReservationStatusChange statusChange = 
                ReservationStatusChange.builder()
                    .reservationId(reservation.getId())
                    .previousStatus(reservation.getStatus())
                    .newStatus(newStatus)
                    .timestamp(Instant.now())
                    .build();

            // 이벤트 발행
            ReservationEvent event = ReservationEvent.builder()
                .type(ReservationEventType.STATUS_CHANGED)
                .reservationId(reservation.getId())
                .status(newStatus)
                .timestamp(Instant.now())
                .build();

            kafka.send("reservation-events", event);

            // 필요한 후속 조치
            switch (newStatus) {
                case CANCELLED:
                    handleCancellation(reservation);
                    break;
                case NO_SHOW:
                    handleNoShow(reservation);
                    break;
                case COMPLETED:
                    handleCompletion(reservation);
                    break;
            }
        }

        private void handleCancellation(Reservation reservation) {
            // 취소 수수료 계산
            Money cancellationFee = calculateCancellationFee(
                reservation
            );
            
            if (cancellationFee.isPositive()) {
                paymentService.chargeCancellationFee(
                    reservation.getUserId(), 
                    cancellationFee
                );
            }

            // 재고 복구
            inventoryService.incrementInventory(
                reservation.getResourceId(), 
                reservation.getTimeSlot()
            );
        }
    }
}
```

이러한 설계를 통해:

1. 효율적인 재고 관리
    - Redis 기반의 실시간 재고 관리
    - 트랜잭션을 통한 동시성 제어
    - 캐시를 통한 성능 최적화

2. 오버부킹 방지
    - 실시간 재고 확인
    - 진행 중인 예약 고려
    - 안전 마진 설정

3. 상태 관리
    - 이벤트 기반 상태 관리
    - 취소 정책 적용
    - 후속 조치 자동화

를 구현할 수 있습니다.

면접관: 피크 시간대의 대량 예약 요청은 어떻게 처리하시겠습니까?

## 3. 대량 예약 처리 시스템

```java
@Service
public class HighLoadReservationService {

    // 1. 예약 요청 큐잉 시스템
    @Component
    public class ReservationQueue {
        private final PriorityBlockingQueue<ReservationRequest> requestQueue;
        private final KafkaTemplate<String, ReservationRequest> kafkaTemplate;
        
        public void enqueueReservation(ReservationRequest request) {
            // 우선순위 설정
            int priority = calculatePriority(request);
            request.setPriority(priority);

            // VIP 사용자는 Kafka 우선순위 큐로 전송
            if (isVipUser(request.getUserId())) {
                kafkaTemplate.send("vip-reservations", request);
            } else {
                // 일반 사용자는 일반 큐로 전송
                kafkaTemplate.send("regular-reservations", request);
            }
        }

        private int calculatePriority(ReservationRequest request) {
            int basePriority = 0;
            
            // VIP 상태에 따른 가중치
            basePriority += getUserPriorityWeight(request.getUserId());
            
            // 예약 시간대별 가중치
            basePriority += getTimeSlotWeight(request.getTimeSlot());
            
            return basePriority;
        }
    }

    // 2. 요청 처리 스케줄러
    @Component
    public class RequestScheduler {
        
        @KafkaListener(topics = {"vip-reservations", "regular-reservations"}, 
                      containerFactory = "batchListener")
        public void processReservationBatch(List<ReservationRequest> batch) {
            // 배치 크기에 따른 처리 조절
            int batchSize = batch.size();
            int workerThreads = calculateOptimalThreads(batchSize);
            
            ExecutorService executor = 
                Executors.newFixedThreadPool(workerThreads);
            
            List<CompletableFuture<ReservationResult>> futures = 
                batch.stream()
                    .map(request -> CompletableFuture.supplyAsync(
                        () -> processReservation(request), executor))
                    .collect(Collectors.toList());

            // 결과 수집 및 처리
            CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0]))
                .thenAccept(v -> handleBatchResults(futures));
        }

        private int calculateOptimalThreads(int batchSize) {
            return Math.min(
                batchSize, 
                Runtime.getRuntime().availableProcessors() * 2
            );
        }
    }

    // 3. 부하 제어 시스템
    @Component
    public class LoadController {
        
        private final RateLimiter rateLimiter;
        private final CircuitBreaker circuitBreaker;
        
        public LoadController() {
            this.rateLimiter = RateLimiter.create(1000); // 초당 1000개 제한
            this.circuitBreaker = CircuitBreaker.builder()
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofSeconds(10))
                .build();
        }

        public boolean shouldAcceptRequest() {
            if (!circuitBreaker.isAllowingRequests()) {
                return false;
            }

            return rateLimiter.tryAcquire();
        }

        @Scheduled(fixedRate = 1000)
        public void adjustRateLimit() {
            // 시스템 부하 지표 수집
            SystemMetrics metrics = collectSystemMetrics();
            
            // 동적으로 처리율 조정
            if (metrics.getCpuUsage() > 80) {
                rateLimiter.setRate(rateLimiter.getRate() * 0.8);
            } else if (metrics.getCpuUsage() < 50) {
                rateLimiter.setRate(rateLimiter.getRate() * 1.2);
            }
        }
    }

    // 4. 장애 복구 시스템
    @Component
    public class FailureRecoverySystem {
        
        private final ReservationRepository repository;
        private final KafkaTemplate<String, ReservationRetry> kafkaTemplate;

        @Scheduled(fixedRate = 5000)
        public void processFailedReservations() {
            List<Reservation> failedReservations = 
                repository.findByStatus(ReservationStatus.FAILED);
                
            for (Reservation reservation : failedReservations) {
                RetryContext context = buildRetryContext(reservation);
                
                if (shouldRetry(context)) {
                    kafkaTemplate.send("reservation-retries", 
                        new ReservationRetry(reservation, context));
                } else {
                    handlePermanentFailure(reservation);
                }
            }
        }

        private boolean shouldRetry(RetryContext context) {
            return context.getAttempts() < 3 && 
                   context.getLastAttempt()
                         .plus(getBackoffInterval(context.getAttempts()))
                         .isBefore(Instant.now());
        }

        private Duration getBackoffInterval(int attempts) {
            return Duration.ofSeconds((long) Math.pow(2, attempts));
        }
    }

    // 5. 모니터링 및 알림
    @Component
    @Slf4j
    public class ReservationMonitor {
        
        private final MeterRegistry registry;
        private final AlertService alertService;

        public void recordReservationMetrics(ReservationResult result) {
            // 성능 메트릭 기록
            registry.timer("reservation.processing.time")
                .record(result.getProcessingTime());
                
            // 성공/실패율 모니터링
            if (result.isSuccess()) {
                registry.counter("reservation.success").increment();
            } else {
                registry.counter("reservation.failure").increment();
            }

            // 임계치 초과 시 알림
            if (result.getProcessingTime().toMillis() > 1000) {
                alertService.sendAlert(
                    AlertLevel.WARNING,
                    "High reservation processing time detected"
                );
            }
        }

        @Scheduled(fixedRate = 60000)
        public void checkSystemHealth() {
            HealthMetrics metrics = collectHealthMetrics();
            
            if (metrics.hasAnomalies()) {
                alertService.sendAlert(
                    AlertLevel.CRITICAL,
                    "System anomalies detected: " + metrics.getDetails()
                );
            }
        }
    }
}
```

이러한 대량 예약 처리 시스템을 통해:

1. 효율적인 요청 큐잉
    - 우선순위 기반 처리
    - 배치 처리로 성능 최적화
    - VIP 사용자 우선 처리

2. 부하 제어
    - 동적 처리율 조정
    - 서킷 브레이커 패턴
    - 시스템 부하 모니터링

3. 장애 복구
    - 자동 재시도 메커니즘
    - 지수 백오프 전략
    - 영구 실패 처리

4. 모니터링
    - 실시간 성능 모니터링
    - 이상 징후 감지
    - 자동 알림 시스템

을 구현하여 피크 시간대의 대량 예약을 안정적으로 처리할 수 있습니다.