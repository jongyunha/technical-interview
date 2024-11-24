# Spring Transaction 상세 동작 원리와 구현

## 1. 트랜잭션 기본 구조

### 1.1 선언적 트랜잭션
```java
@Service
public class UserService {
    
    @Transactional
    public void createUser(User user) {
        // 트랜잭션 내부 로직
    }
}
```

### 1.2 트랜잭션 프록시 생성 과정
```java
// 1. Spring AOP가 @Transactional 클래스/메서드를 스캔
// 2. TransactionInterceptor를 포함한 프록시 객체 생성
// 3. Bean 등록 시 프록시 객체로 대체

// 실제 생성되는 바이트코드 (디컴파일 예시)
public class UserService$$EnhancerBySpringCGLIB extends UserService {
    private TransactionInterceptor transactionInterceptor;
    
    public void createUser(User user) {
        try {
            // 트랜잭션 시작
            TransactionInfo txInfo = this.transactionInterceptor.createTransactionIfNecessary();
            Object retVal = super.createUser(user);
            // 트랜잭션 커밋
            this.transactionInterceptor.commitTransactionAfterReturning(txInfo);
            return retVal;
        } catch (Throwable ex) {
            // 트랜잭션 롤백
            this.transactionInterceptor.rollbackOn(ex);
            throw ex;
        }
    }
}
```

## 2. 트랜잭션 동작 상세 과정

### 2.1 트랜잭션 시작
```java
// TransactionInterceptor 내부 동작
public Object invoke(MethodInvocation invocation) throws Throwable {
    TransactionAttribute txAttr = getTransactionAttribute();
    PlatformTransactionManager tm = getTransactionManager();
    
    // 트랜잭션 시작
    TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, methodName);
    Object retVal = null;
    try {
        // 실제 비즈니스 메서드 호출
        retVal = invocation.proceed();
    } catch (Throwable ex) {
        // 예외 발생 시 롤백 처리
        completeTransactionAfterThrowing(txInfo, ex);
        throw ex;
    } finally {
        cleanupTransactionInfo(txInfo);
    }
    // 정상 처리 시 커밋
    commitTransactionAfterReturning(txInfo);
    return retVal;
}
```

### 2.2 트랜잭션 전파 속성 처리
```java
// TransactionDefinition 구현
public interface TransactionDefinition {
    // 트랜잭션 전파 속성 정의
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;
}

// 전파 속성에 따른 트랜잭션 처리 로직
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction) {
    
    switch (definition.getPropagationBehavior()) {
        case TransactionDefinition.PROPAGATION_REQUIRED:
            // 기존 트랜잭션 사용
            return new TransactionStatus(transaction);
            
        case TransactionDefinition.PROPAGATION_REQUIRES_NEW:
            // 새로운 트랜잭션 생성
            SuspendedResourcesHolder suspendedResources = suspend(transaction);
            try {
                return startTransaction(definition);
            } catch (RuntimeException | Error ex) {
                resume(transaction, suspendedResources);
                throw ex;
            }
    }
}
```

### 2.3 DB 커넥션 및 트랜잭션 동기화
```java
// TransactionSynchronizationManager 내부 동작
public abstract class TransactionSynchronizationManager {
    private static final ThreadLocal<Map<Object, Object>> resources =
            new ThreadLocal<>();
    
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
            new ThreadLocal<>();
            
    // 커넥션 바인딩
    public static void bindResource(Object key, Object value) {
        Map<Object, Object> map = resources.get();
        if (map == null) {
            map = new HashMap<>();
            resources.set(map);
        }
        map.put(key, value);
    }
}
```

## 3. 트랜잭션 격리 수준 구현

### 3.1 JDBC 레벨 격리 수준 설정
```java
// JdbcTransactionManager 내부
protected void doBegin(Object transaction, TransactionDefinition definition) {
    Connection con = ((JdbcTransactionObject)transaction).getConnectionHolder().getConnection();
    
    // 격리 수준 설정
    if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
        con.setTransactionIsolation(definition.getIsolationLevel());
    }
    
    // 트랜잭션 시작
    con.setAutoCommit(false);
}
```

### 3.2 JPA/Hibernate 트랜잭션 처리
```java
// JpaTransactionManager 내부
public class JpaTransactionManager extends AbstractPlatformTransactionManager {
    
    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        JpaTransactionObject txObject = (JpaTransactionObject) transaction;
        
        EntityManager em = entityManagerFactory.createEntityManager();
        txObject.setEntityManagerHolder(new EntityManagerHolder(em));
        
        // EntityManager 트랜잭션 시작
        EntityTransaction tx = em.getTransaction();
        tx.begin();
        
        // 격리 수준 및 읽기 전용 설정
        if (definition.isReadOnly()) {
            em.setFlushMode(FlushModeType.COMMIT);
        }
    }
}
```

## 4. 예외 처리와 롤백

### 4.1 롤백 규칙 처리
```java
public class DefaultTransactionAttribute implements TransactionAttribute {
    
    @Override
    public boolean rollbackOn(Throwable ex) {
        return (ex instanceof RuntimeException || ex instanceof Error);
    }
}

// 커스텀 롤백 규칙
@Transactional(rollbackFor = {BusinessException.class},
               noRollbackFor = {IgnorableException.class})
public void businessMethod() {
    // 비즈니스 로직
}
```

### 4.2 롤백 처리 로직
```java
// TransactionAspectSupport 내부
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.hasTransaction()) {
        if (txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
        } else {
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
        }
    }
}
```

이러한 내부 동작 이해를 통해 다음과 같은 이점을 얻을 수 있습니다:
1. 트랜잭션 문제 발생 시 효과적인 디버깅
2. 적절한 트랜잭션 설정 선택
3. 성능 최적화 가능
4. 트랜잭션 관련 버그 예방

실제 면접에서는 이러한 내부 동작에 대한 이해를 바탕으로, 실제 경험한 문제 해결 사례나 최적화 경험을 함께 설명하면 좋습니다.

# Private 메서드에서 @Transactional이 동작하지 않는 이유

## 1. 프록시 기반 AOP의 동작 원리

### 1.1 Spring AOP 프록시 생성
```java
// 원본 클래스
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        privateMethod(); // private 메서드 호출
    }
    
    @Transactional // 동작하지 않음
    private void privateMethod() {
        // 비즈니스 로직
    }
}

// Spring이 생성하는 프록시 클래스 (바이트코드)
public class UserService$$EnhancerBySpringCGLIB extends UserService {
    private TransactionInterceptor transactionInterceptor;
    
    public void createUser(User user) {
        // 프록시는 public 메서드만 오버라이딩 가능
        try {
            TransactionInfo txInfo = transactionInterceptor.createTransactionIfNecessary();
            super.createUser(user);
            transactionInterceptor.commitTransactionAfterReturning(txInfo);
        } catch (Throwable ex) {
            transactionInterceptor.rollbackOn(ex);
            throw ex;
        }
    }
    
    // private 메서드는 오버라이딩 불가능
    // private void privateMethod() { } // 컴파일 에러!
}
```

## 2. 자바 언어의 제약사항

### 2.1 상속과 private 메서드
```java
public class Parent {
    private void privateMethod() {
        // private 메서드
    }
}

public class Child extends Parent {
    // private 메서드 오버라이딩 불가
    private void privateMethod() {
        // 이것은 새로운 메서드이지, 오버라이딩이 아님
    }
}
```

### 2.2 프록시 패턴의 한계
```java
// 프록시 패턴 구현
public interface UserServiceInterface {
    void createUser(User user);
}

public class UserServiceProxy implements UserServiceInterface {
    private final UserService target;
    private final TransactionManager txManager;
    
    @Override
    public void createUser(User user) {
        // private 메서드에 대한 프록시 처리 불가능
        try {
            txManager.begin();
            target.createUser(user);
            txManager.commit();
        } catch (Exception e) {
            txManager.rollback();
            throw e;
        }
    }
}
```

## 3. 해결 방법

### 3.1 메서드 분리 및 가시성 변경
```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        processUserCreation(user); // protected 또는 public 메서드로 변경
    }
    
    @Transactional
    protected void processUserCreation(User user) {
        // 이전의 private 메서드 로직
    }
}
```

### 3.2 내부 클래스 사용
```java
@Service
public class UserService {
    @Autowired
    private UserOperations userOperations;
    
    public void createUser(User user) {
        userOperations.processUserCreation(user);
    }
}

@Component
public class UserOperations {
    @Transactional
    public void processUserCreation(User user) {
        // 비즈니스 로직
    }
}
```

## 4. AspectJ를 사용한 대안

```java
// AspectJ를 사용하면 private 메서드에도 적용 가능
// 하지만 설정이 복잡하고 컴파일 시점 위빙이 필요
@Aspect
public class TransactionAspect {
    @Around("execution(private * com.example..*.*(..))")
    public Object handleTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        // 트랜잭션 처리 로직
    }
}
```

## 5. 바이트코드 수준에서의 차이

### 5.1 일반적인 프록시 생성
```java
// 일반적인 CGLIB 프록시 바이트코드
public class UserService$$EnhancerBySpringCGLIB extends UserService {
    // 메서드 시그니처가 그대로 유지됨
    public void createUser(User user) {
        // 프록시 로직
        super.createUser(user);
    }
}
```

### 5.2 AspectJ 위빙 후 바이트코드
```java
// AspectJ 위빙 후 실제 클래스 바이트코드 변경
public class UserService {
    private void privateMethod() {
        // AspectJ가 직접 바이트코드를 수정
        Object aspectInstance = AspectRuntime.aspectOf();
        TransactionAspect.around(...);
        // 원래 메서드 코드
    }
}
```

## 6. 실무적 권장사항

1. private 메서드에 트랜잭션이 필요한 경우:
    - 메서드의 가시성을 protected로 변경
    - 별도의 클래스로 분리
    - 트랜잭션 경계 재설계

2. 트랜잭션 경계 설계 원칙:
   ```java
   @Service
   public class UserService {
       @Transactional
       public void businessOperation() {
           // 트랜잭션 경계는 public 메서드에 설정
           // 내부 private 메서드들은 트랜잭션 내에서 실행
           helperMethod1();
           helperMethod2();
       }
       
       private void helperMethod1() {
           // 트랜잭션 어노테이션 불필요
       }
       
       private void helperMethod2() {
           // 트랜잭션 어노테이션 불필요
       }
   }
   ```

이러한 제약사항을 이해하고 설계 단계에서 고려하면, 더 효과적인 트랜잭션 관리가 가능합니다.