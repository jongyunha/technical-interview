# Spring AOP(Aspect Oriented Programming)에 대한 기술 면접 답변

## 주요 질문
"Spring AOP의 개념과 동작 원리, 그리고 실제 어떤 상황에서 활용하는지 설명해주세요."

## 1. AOP 핵심 개념

### 1.1 기본 용어 설명
```java
@Aspect
@Component
public class LoggingAspect {
    
    // Pointcut: 어디에 적용할 것인가
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}
    
    // Advice: 무엇을 적용할 것인가
    @Around("serviceLayer()")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        // Before advice 로직
        long startTime = System.currentTimeMillis();
        
        // JoinPoint: 실제 메소드 실행
        Object result = joinPoint.proceed();
        
        // After advice 로직
        long endTime = System.currentTimeMillis();
        System.out.println("실행 시간: " + (endTime - startTime) + "ms");
        
        return result;
    }
}
```

## 2. AOP 구현 방식과 장단점

### 2.1 프록시 기반 AOP
```java
// Target 객체
@Service
public class UserService {
    public void createUser(User user) {
        // 비즈니스 로직
    }
}

// Spring AOP는 프록시를 통해 구현
UserService userService = context.getBean(UserService.class);
// userService는 실제로 프록시 객체
```

장점:
- 간단한 설정으로 적용 가능
- 별도의 컴파일 과정 불필요
- 런타임에 유연한 적용 가능

단점:
- 메서드 호출에만 적용 가능
- 자가 호출(self-invocation) 불가
- 프록시 객체 생성으로 인한 경미한 성능 저하

### 2.2 AspectJ 기반 AOP
```java
@Aspect
public class SecurityAspect {
    @Before("execution(* com.example.secure.*.*(..))")
    public void checkSecurity() {
        // 보안 체크 로직
    }
}
```

장점:
- 모든 조인 포인트 지원
- 컴파일 시점 위빙으로 성능 우수
- 완전한 AOP 기능 제공

단점:
- 설정이 복잡
- 추가 컴파일 과정 필요
- 학습 곡선이 높음

## 3. 주요 사용 사례

### 3.1 로깅
```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger logger = LoggerFactory.getLogger(LoggingAspect.class);
    
    @Around("@annotation(Loggable)")
    public Object logMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        logger.info("Starting {}", methodName);
        
        Object result = joinPoint.proceed();
        
        logger.info("Completed {}", methodName);
        return result;
    }
}
```

### 3.2 트랜잭션 관리
```java
@Aspect
@Component
public class TransactionAspect {
    
    @Around("@annotation(Transactional)")
    public Object handleTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            beginTransaction();
            Object result = joinPoint.proceed();
            commit();
            return result;
        } catch (Exception e) {
            rollback();
            throw e;
        }
    }
}
```

### 3.3 보안 처리
```java
@Aspect
@Component
public class SecurityAspect {
    
    @Before("@annotation(secured)")
    public void checkSecurity(JoinPoint joinPoint, Secured secured) {
        String[] roles = secured.value();
        if (!SecurityContext.hasAnyRole(roles)) {
            throw new AccessDeniedException("접근 권한이 없습니다.");
        }
    }
}
```

## 4. 실제 활용 예시

### 4.1 성능 모니터링
```java
@Aspect
@Component
public class PerformanceAspect {
    
    @Around("@annotation(Monitored)")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        Object result = joinPoint.proceed();
        long end = System.nanoTime();
        
        MetricRegistry.recordMethodExecution(
            joinPoint.getSignature().toLongString(),
            end - start
        );
        
        return result;
    }
}
```

### 4.2 API 요청 응답 로깅
```java
@Aspect
@Component
public class ApiLoggingAspect {
    
    @Around("@within(org.springframework.web.bind.annotation.RestController)")
    public Object logApiCall(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = 
            ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes())
            .getRequest();
            
        logger.info("API Call: {} {}", request.getMethod(), request.getRequestURI());
        
        Object result = joinPoint.proceed();
        
        logger.info("API Response: {}", result);
        return result;
    }
}
```

## 5. Best Practices

### 5.1 적절한 Pointcut 설계
```java
@Aspect
@Component
public class OptimizedAspect {
    // 너무 광범위한 Pointcut은 피하기
    @Pointcut("execution(* com.example..*.*(..))")  // BAD
    
    // 구체적인 Pointcut 사용
    @Pointcut("execution(* com.example.service.*Service.*(..))")  // GOOD
    public void serviceLayer() {}
}
```

### 5.2 예외 처리
```java
@Around("serviceLayer()")
public Object handleErrors(ProceedingJoinPoint joinPoint) throws Throwable {
    try {
        return joinPoint.proceed();
    } catch (BusinessException e) {
        logger.error("Business error occurred", e);
        throw e;
    } catch (Exception e) {
        logger.error("Unexpected error", e);
        throw new SystemException("시스템 오류가 발생했습니다", e);
    }
}
```

## 6. 주의사항과 제약사항

### 6.1 자가 호출 문제
```java
@Service
public class UserService {
    public void createUser(User user) {
        // 외부에서 호출 시 AOP 동작
        this.validateUser(user);  // 내부 호출은 AOP 동작 안함
    }
    
    @Validated
    public void validateUser(User user) {
        // 검증 로직
    }
}
```

### 6.2 프록시 제한사항
```java
// private 메서드는 AOP 적용 불가
@Service
public class OrderService {
    @Transactional  // 동작 안함
    private void processOrder() {
        // 비즈니스 로직
    }
}
```

AOP를 사용할 때는 이러한 장단점과 제약사항을 잘 이해하고, 적절한 상황에서 활용하는 것이 중요합니다.