# Spring IoC/DI에 대한 기술 면접 답변

## 주요 질문
"Spring IoC/DI의 개념과 동작 원리에 대해 설명해주세요. 어떤 장점이 있나요?"

## 1. IoC(Inversion of Control)의 개념

### 1.1 기존 방식 vs IoC
```java
// 기존 방식 - 직접 객체 생성과 의존성 관리
public class UserService {
    private UserRepository userRepository = new UserRepository();  // 강한 결합
}

// IoC 방식 - 컨테이너가 객체 생성과 의존성 관리
@Service
public class UserService {
    private final UserRepository userRepository;  // 느슨한 결합
    
    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

## 2. DI(Dependency Injection)의 방식

### 2.1 생성자 주입
```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    // 생성자 주입 (권장 방식)
    @Autowired
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

### 2.2 Setter 주입
```java
@Service
public class UserService {
    private UserRepository userRepository;
    
    // Setter 주입
    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

### 2.3 필드 주입
```java
@Service
public class UserService {
    @Autowired  // 필드 주입 (권장하지 않음)
    private UserRepository userRepository;
}
```

## 3. Spring Bean의 생명주기

### 3.1 Bean 생명주기 콜백
```java
@Component
public class MyBean implements InitializingBean, DisposableBean {
    
    @PostConstruct
    public void init() {
        System.out.println("Bean 초기화");
    }
    
    @PreDestroy
    public void cleanup() {
        System.out.println("Bean 소멸");
    }
    
    @Override
    public void afterPropertiesSet() {
        System.out.println("InitializingBean: 프로퍼티 설정 후");
    }
    
    @Override
    public void destroy() {
        System.out.println("DisposableBean: 빈 소멸");
    }
}
```

## 4. Configuration과 Bean 등록

### 4.1 Java Config 방식
```java
@Configuration
public class AppConfig {
    
    @Bean
    public UserRepository userRepository() {
        return new UserRepository();
    }
    
    @Bean
    public UserService userService(UserRepository userRepository) {
        return new UserService(userRepository);
    }
}
```

### 4.2 Component Scan
```java
@SpringBootApplication
@ComponentScan(basePackages = "com.example")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 5. Bean Scope

### 5.1 Singleton Scope
```java
@Component
@Scope("singleton")
public class SingletonBean {
    // 기본 스코프: 애플리케이션당 하나의 인스턴스
}
```

### 5.2 Prototype Scope
```java
@Component
@Scope("prototype")
public class PrototypeBean {
    // 요청할 때마다 새로운 인스턴스 생성
}
```

## 근거 자료

### 1. 공식 문서
- [Spring Core Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)

### 2. 기술 서적
- "Spring in Action" by Craig Walls
- "Expert Spring MVC and Web Flow" by Seth Ladd

## 실무 관련 추가 질문

1. "생성자 주입을 권장하는 이유는 무엇인가요?"
- 불변성 보장
- 필수 의존성 명시
- 순환 참조 방지
- 테스트 용이성

2. "Bean의 생명주기는 어떻게 관리하나요?"
- 생성
- 의존성 주입
- 초기화
- 사용
- 소멸

3. "Singleton Bean은 Thread Safe한가요?"
```java
@Service
public class UserService {
    // Singleton Bean에서 상태를 가지면 Thread Safe하지 않음
    private User currentUser;  // 위험!
    
    // 대신 메서드 스코프나 파라미터로 상태 관리
    public void processUser(User user) {
        // 메서드 스코프에서 처리
    }
}
```

## 실제 사용 예시

### 의존성 주입 예시
```java
@Service
@RequiredArgsConstructor  // Lombok을 사용한 생성자 주입
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    public void createOrder(OrderRequest request) {
        // 각 의존성을 활용한 비즈니스 로직
        Order order = orderRepository.save(request.toOrder());
        paymentService.processPayment(order);
        notificationService.sendOrderConfirmation(order);
    }
}
```

### 설정 클래스 예시
```java
@Configuration
@Profile("production")
public class ProductionConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "app.datasource")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            EntityManagerFactoryBuilder builder, DataSource dataSource) {
        return builder
            .dataSource(dataSource)
            .packages("com.example.domain")
            .build();
    }
}
```

실제 면접에서는 이론적인 지식과 함께 Spring IoC/DI를 활용한 실제 문제 해결 경험, 아키텍처 설계 경험 등을 구체적으로 설명하는 것이 좋습니다.

# Spring IoC/DI의 장점

## 1. 결합도(Coupling) 감소

### 1.1 강한 결합의 문제점
```java
// 강한 결합의 예
public class UserService {
    // UserRepository 구현체에 직접 의존
    private UserRepository userRepository = new JpaUserRepository();
}
```

### 1.2 느슨한 결합의 장점
```java
// 느슨한 결합의 예
public class UserService {
    // 인터페이스에 의존
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// 구현체 교체가 용이
@Configuration
public class AppConfig {
    @Bean
    public UserRepository userRepository() {
        // JPA, MyBatis, MongoDB 등 구현체 변경이 쉬움
        return new MongoUserRepository();
    }
}
```

## 2. 테스트 용이성 향상

### 2.1 테스트가 어려운 코드
```java
public class OrderService {
    private final PaymentGateway paymentGateway = new RealPaymentGateway();
    
    public void processOrder(Order order) {
        // 실제 결제가 발생하여 테스트하기 어려움
        paymentGateway.processPayment(order.getAmount());
    }
}
```

### 2.2 테스트하기 쉬운 코드
```java
public class OrderService {
    private final PaymentGateway paymentGateway;
    
    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
    
    @Test
    public void processOrder_Should_Success() {
        // Mock 객체를 주입하여 테스트 가능
        PaymentGateway mockGateway = mock(PaymentGateway.class);
        OrderService service = new OrderService(mockGateway);
        service.processOrder(new Order());
        verify(mockGateway).processPayment(any());
    }
}
```

## 3. 코드 재사용성 향상

```java
// 여러 서비스에서 재사용 가능한 컴포넌트
@Component
public class EmailService {
    public void sendEmail(String to, String subject, String content) {
        // 이메일 발송 로직
    }
}

@Service
public class UserService {
    private final EmailService emailService;
    
    public UserService(EmailService emailService) {
        this.emailService = emailService;
    }
}

@Service
public class OrderService {
    private final EmailService emailService;
    
    public OrderService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

## 4. 유지보수성 향상

### 4.1 변경의 국소화
```java
// 구현체 변경 시 설정 클래스만 수정
@Configuration
public class EmailConfig {
    @Bean
    public EmailSender emailSender() {
        // AWS SES에서 Gmail로 변경 시 이 부분만 수정
        return new GmailSender();
    }
}
```

## 5. 생명주기 관리 용이

```java
@Component
public class DatabaseConnection {
    @PostConstruct
    public void init() {
        // 커넥션 풀 초기화
    }
    
    @PreDestroy
    public void cleanup() {
        // 리소스 정리
    }
}
```

## 6. 객체 생성 및 구성의 중앙화

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityService securityService(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder,
            JwtTokenProvider jwtTokenProvider) {
        // 복잡한 의존성 그래프도 Spring이 관리
        return new SecurityService(
            userDetailsService, 
            passwordEncoder, 
            jwtTokenProvider
        );
    }
}
```

## 7. AOP 적용 용이성

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        // 메서드 실행 시간 로깅 등 횡단 관심사 처리
        long start = System.currentTimeMillis();
        Object proceed = joinPoint.proceed();
        long executionTime = System.currentTimeMillis() - start;
        System.out.println(joinPoint.getSignature() + " executed in " + executionTime + "ms");
        return proceed;
    }
}
```

## 8. 환경별 설정 관리 용이

```java
@Configuration
@Profile("development")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@Profile("production")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://production-db:3306/myapp")
            .username("prod_user")
            .password("prod_pass")
            .build();
    }
}
```

이러한 장점들로 인해 Spring IoC/DI는 대규모 엔터프라이즈 애플리케이션 개발에 있어 필수적인 요소가 되었습니다. 특히 마이크로서비스 아키텍처에서는 이러한 장점들이 더욱 부각됩니다.