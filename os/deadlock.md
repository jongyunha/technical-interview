# 교착상태(Deadlock)에 대한 기술 면접 답변

## 주요 질문
"교착상태(Deadlock)가 무엇이며, 발생 조건과 해결 방법에 대해 설명해주세요."

## 1. 교착상태 정의

교착상태(Deadlock)는 두 개 이상의 프로세스나 스레드가 서로가 가진 자원을 기다리며 무한히 대기하는 상태를 의미합니다.

### 1.1 교착상태 예시 코드
```java
// 교착상태 발생 가능한 코드
public class DeadlockExample {
    private static Object lock1 = new Object();
    private static Object lock2 = new Object();
    
    public static void main(String[] args) {
        Thread thread1 = new Thread(() -> {
            synchronized (lock1) {
                System.out.println("Thread 1: Holding lock1");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (lock2) {
                    System.out.println("Thread 1: Holding lock2");
                }
            }
        });
        
        Thread thread2 = new Thread(() -> {
            synchronized (lock2) {
                System.out.println("Thread 2: Holding lock2");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (lock1) {
                    System.out.println("Thread 2: Holding lock1");
                }
            }
        });
        
        thread1.start();
        thread2.start();
    }
}
```

## 2. 교착상태 발생 조건

### 2.1 4가지 필요 조건
1. 상호 배제(Mutual Exclusion)
    - 자원은 한 번에 한 프로세스만 사용 가능
   ```java
   synchronized void transfer(Account from, Account to, double amount) {
       // 계좌 잔액 이체 로직
   }
   ```

2. 점유와 대기(Hold and Wait)
    - 자원을 보유한 상태에서 다른 자원을 요청
   ```java
   synchronized (account1) {
       synchronized (account2) {
           // 두 계좌를 모두 필요로 하는 작업
       }
   }
   ```

3. 비선점(No Preemption)
    - 다른 프로세스의 자원을 강제로 빼앗을 수 없음
   ```java
   // 자원을 강제로 해제할 수 없는 상황
   public void processData() {
       lock.lock();  // 한번 획득한 락은 직접 해제하기 전까지 유지
       try {
           // 처리 로직
       } finally {
           lock.unlock();
       }
   }
   ```

4. 순환 대기(Circular Wait)
    - 프로세스간 자원 요청이 순환 형태로 발생
   ```java
   // 순환 대기 발생 가능 코드
   Thread 1: lock(A) → wait(B)
   Thread 2: lock(B) → wait(A)
   ```

## 3. 교착상태 해결 방법

### 3.1 예방(Prevention)
```java
// 순서화된 락 획득으로 교착상태 예방
public class Account {
    private static final Object tieLock = new Object();
    
    public void transfer(Account from, Account to, double amount) {
        // 계좌 ID를 기준으로 락 획득 순서 정함
        if (from.getId() < to.getId()) {
            synchronized (from) {
                synchronized (to) {
                    // 이체 로직
                }
            }
        } else if (from.getId() > to.getId()) {
            synchronized (to) {
                synchronized (from) {
                    // 이체 로직
                }
            }
        } else {
            synchronized (tieLock) {
                synchronized (from) {
                    synchronized (to) {
                        // 이체 로직
                    }
                }
            }
        }
    }
}
```

### 3.2 회피(Avoidance)
```java
// 은행원 알고리즘 구현 예시
public class BankerAlgorithm {
    private int[] available;
    private int[][] maximum;
    private int[][] allocation;
    private int[][] need;
    
    public boolean isSafeState(int[] request, int processId) {
        // 안전 상태 검사 로직
        return true; // 안전한 경우에만 자원 할당
    }
}
```

### 3.3 탐지 및 복구(Detection and Recovery)
```java
// 데드락 탐지 예시
public class DeadlockDetector {
    private Set<Thread> findDeadlockedThreads() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        long[] threadIds = threadBean.findDeadlockedThreads();
        Set<Thread> deadlockedThreads = new HashSet<>();
        if (threadIds != null) {
            ThreadInfo[] threadInfos = threadBean.getThreadInfo(threadIds);
            for (ThreadInfo threadInfo : threadInfos) {
                Thread thread = findMatchingThread(threadInfo.getThreadId());
                if (thread != null) {
                    deadlockedThreads.add(thread);
                }
            }
        }
        return deadlockedThreads;
    }
}
```

## 4. 실제 사용 예시

### 4.1 Lock 순서화
```java
public class SafeBankTransfer {
    private final Lock lock = new ReentrantLock();
    
    public void transfer(Account from, Account to, double amount) {
        Lock firstLock = from.getId() < to.getId() ? from.getLock() : to.getLock();
        Lock secondLock = from.getId() < to.getId() ? to.getLock() : from.getLock();
        
        firstLock.lock();
        try {
            secondLock.lock();
            try {
                // 안전한 이체 로직
            } finally {
                secondLock.unlock();
            }
        } finally {
            firstLock.unlock();
        }
    }
}
```

### 4.2 타임아웃 사용
```java
public class TimeoutLocking {
    private final Lock lock = new ReentrantLock();
    
    public boolean executeWithTimeout() {
        try {
            if (lock.tryLock(1000, TimeUnit.MILLISECONDS)) {
                try {
                    // 작업 수행
                    return true;
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return false;
    }
}
```

## 근거 자료

### 1. 학술 자료
- ["System Deadlocks" by Coffman et al.](https://dl.acm.org/doi/10.1145/356586.356588)
    - ACM Computing Surveys
- ["Detection of Mutual Exclusion Using Static Analysis"](https://ieeexplore.ieee.org/document/1231154)
    - IEEE Proceedings

### 2. 기술 문서
- [Java Concurrency in Practice](https://jcip.net/)
- [Oracle's Java Tutorial - Deadlock](https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html)
- [Java Thread Deadlock Detection](https://docs.oracle.com/javase/8/docs/technotes/guides/management/thread-monitoring.html)

### 3. 운영체제 교과서
- [Operating System Concepts (Silberschatz)](https://www.os-book.com/OS10/)
    - Chapter 7: Deadlocks
- [Modern Operating Systems (Tanenbaum)](https://www.pearson.com/en-us/subject-catalog/p/modern-operating-systems/P200000003295)

### 4. 실무 관련 자료
- [Spring Framework Transaction Management](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)
- [MySQL InnoDB Deadlock Documentation](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)

## 실제 면접 대비 추가 질문

1. "실제 프로젝트에서 데드락을 경험한 적이 있나요? 어떻게 해결하셨나요?"

2. "데드락과 라이브락(Livelock)의 차이점은 무엇인가요?"

3. "분산 시스템에서의 데드락은 어떻게 해결할 수 있을까요?"

4. "데이터베이스 트랜잭션에서 발생하는 데드락은 어떻게 해결하시나요?"

실제 면접에서는 이론적인 지식뿐만 아니라, 실제 경험한 데드락 상황과 해결 방법에 대해 구체적으로 설명하는 것이 좋습니다.