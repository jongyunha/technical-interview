# 마이크로서비스 아키텍처 설계 면접

면접관: "대규모 이커머스 시스템을 마이크로서비스 아키텍처로 설계해주세요. 특히 서비스 분리와 통신 전략에 대해 자세히 설명해주세요."

지원자: 네, 먼저 몇 가지 확인하고 싶은 사항이 있습니다.

1. 현재 시스템의 규모와 트래픽은 어느 정도인가요?
2. 가장 중요한 비즈니스 요구사항은 무엇인가요?
3. 성능이나 확장성 관련 특별한 요구사항이 있나요?
4. 현재 겪고 있는 주요 기술적 문제점은 무엇인가요?

면접관:
1. 일 평균 주문 10만 건, 피크 시간대는 평균의 5배 수준입니다.
2. 빠른 기능 출시와 서비스 안정성이 가장 중요합니다.
3. 주문 시스템은 99.99% 가용성이 필요하며, 결제는 3초 이내 응답이 필요합니다.
4. 현재 모놀리식 시스템에서 배포가 어렵고, 일부 기능의 장애가 전체 시스템에 영향을 주는 문제가 있습니다.

지원자: 이해했습니다. 마이크로서비스 아키텍처를 설계해보겠습니다.

## 1. 서비스 분리 전략

```java
// 1. 도메인 기반 서비스 분리
public class ServiceBoundaries {
    /*
    Core Services:
    - Product Service: 상품 관리
    - Order Service: 주문 처리
    - Payment Service: 결제 처리
    - Inventory Service: 재고 관리
    - User Service: 회원 관리
    
    Supporting Services:
    - Notification Service: 알림 처리
    - Analytics Service: 데이터 분석
    - Search Service: 검색 기능
    
    Infrastructure Services:
    - API Gateway
    - Service Discovery
    - Configuration Service
    */
}

// 2. 서비스 단위 예시
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentServiceClient paymentClient;
    private final InventoryServiceClient inventoryClient;
    
    @Transactional
    public OrderResult createOrder(OrderRequest request) {
        // 1. 재고 확인
        InventoryResult inventory = inventoryClient.checkInventory(
            request.getProductId(), request.getQuantity());
            
        // 2. 결제 처리
        PaymentResult payment = paymentClient.processPayment(
            request.getPaymentInfo());
            
        // 3. 주문 생성
        Order order = orderRepository.save(
            Order.from(request, payment.getTransactionId()));
            
        return OrderResult.success(order);
    }
}
```

## 2. 서비스 간 통신 전략

```java
// 1. 동기식 통신 (REST)
@FeignClient(name = "payment-service")
public interface PaymentServiceClient {
    @PostMapping("/api/payments")
    PaymentResult processPayment(@RequestBody PaymentRequest request);
}

// 2. 비동기식 통신 (Event-Driven)
@Service
public class OrderEventHandler {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreatedEvent(Order order) {
        OrderEvent event = OrderEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .status(OrderStatus.CREATED)
            .timestamp(Instant.now())
            .build();
            
        kafkaTemplate.send("order-events", event);
    }
    
    @KafkaListener(topics = "inventory-events")
    public void handleInventoryEvent(InventoryEvent event) {
        // 재고 이벤트 처리
    }
}
```

면접관: 서비스 간 데이터 일관성은 어떻게 보장하시나요?

## 3. 데이터 일관성 보장 전략

### 3.1 Saga 패턴 구현
```java
@Service
public class OrderSagaCoordinator {
    
    // 1. 주문 생성 Saga
    public OrderResult createOrder(OrderRequest request) {
        SagaInstance<OrderContext> saga = sagaManager.create(new OrderContext(request));
        
        return saga.executeSteps(List.of(
            // Step 1: 재고 확인
            new SagaStep<>(
                ctx -> inventoryService.reserve(ctx.getProductId()),
                ctx -> inventoryService.cancelReservation(ctx.getProductId())
            ),
            // Step 2: 결제 처리
            new SagaStep<>(
                ctx -> paymentService.process(ctx.getPaymentInfo()),
                ctx -> paymentService.refund(ctx.getTransactionId())
            ),
            // Step 3: 주문 생성
            new SagaStep<>(
                ctx -> orderRepository.create(ctx.getOrderDetails()),
                ctx -> orderRepository.markAsFailed(ctx.getOrderId())
            )
        ));
    }
    
    // 2. Saga 상태 관리
    @Getter
    public class SagaInstance<T> {
        private final String sagaId;
        private final T context;
        private final Map<Integer, StepStatus> stepStatuses = new HashMap<>();
        
        public void compensate(int fromStep) {
            // 보상 트랜잭션 실행
            for (int i = fromStep; i >= 0; i--) {
                steps.get(i).compensate(context);
                stepStatuses.put(i, StepStatus.COMPENSATED);
            }
        }
    }
}
```

### 3.2 이벤트 소싱
```java
@Service
public class EventSourcingHandler {
    
    private final EventStore eventStore;
    
    // 1. 이벤트 저장
    public void saveEvent(OrderEvent event) {
        EventStream stream = eventStore.getStream(event.getAggregateId());
        stream.append(event);
        eventStore.save(stream);
        
        // 이벤트 발행
        eventPublisher.publish(event);
    }
    
    // 2. 상태 재구성
    public Order reconstructOrderState(String orderId) {
        EventStream stream = eventStore.getStream(orderId);
        Order order = new Order();
        
        stream.getEvents().forEach(event -> {
            switch (event.getType()) {
                case ORDER_CREATED:
                    order.apply((OrderCreatedEvent) event);
                    break;
                case ORDER_UPDATED:
                    order.apply((OrderUpdatedEvent) event);
                    break;
                // ... 기타 이벤트 처리
            }
        });
        
        return order;
    }
}
```

### 3.3 CQRS 패턴
```java
// 1. Command 모델
@Service
public class OrderCommandService {
    private final EventSourcingHandler eventHandler;
    
    @Transactional
    public void createOrder(CreateOrderCommand command) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId(command.getOrderId())
            .userId(command.getUserId())
            .items(command.getItems())
            .build();
            
        eventHandler.saveEvent(event);
    }
}

// 2. Query 모델
@Service
public class OrderQueryService {
    private final OrderReadRepository readRepository;
    
    public OrderDetailView getOrderDetails(String orderId) {
        return readRepository.findOrderDetails(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    // 이벤트 구독하여 읽기 모델 업데이트
    @EventListener
    public void on(OrderCreatedEvent event) {
        OrderDetailView view = OrderDetailView.from(event);
        readRepository.save(view);
    }
}
```

### 3.4 장애 대응 및 복구
```java
@Service
public class ResiliencyManager {
    
    // 1. Circuit Breaker 구현
    @CircuitBreaker(name = "payment")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.processPayment(request);
    }
    
    // 2. 재시도 정책
    @Retryable(
        value = {ServiceTemporaryException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000))
    public InventoryResult checkInventory(String productId) {
        return inventoryClient.checkAvailability(productId);
    }
    
    // 3. Fallback 처리
    @Recover
    public PaymentResult paymentFallback(PaymentRequest request) {
        // 대체 결제 처리 또는 실패 처리
        return PaymentResult.failed("Service temporarily unavailable");
    }
}
```

### 3.5 모니터링 및 추적
```java
@Configuration
public class ObservabilityConfig {
    
    // 1. 분산 추적
    @Bean
    public Tracer jaegerTracer() {
        return new Tracer.Builder("order-service")
            .withSampler(new ConstSampler(true))
            .withReporter(jaegerReporter())
            .build();
    }
    
    // 2. 메트릭 수집
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }
    
    // 3. 로그 집계
    @Bean
    public LogstashAppender logstashAppender() {
        LogstashAppender appender = new LogstashAppender();
        appender.setDestination("logstash:5000");
        appender.setEncoding("UTF-8");
        return appender;
    }
}
```

이러한 전략들을 통해:
1. 트랜잭션의 일관성 보장
2. 서비스 간 느슨한 결합 유지
3. 장애 격리와 복구 가능
4. 시스템 가시성 확보

를 달성할 수 있습니다.


