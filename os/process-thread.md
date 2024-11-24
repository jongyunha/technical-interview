# 프로세스와 스레드에 대한 기술 면접 답변

## 주요 질문
"프로세스와 스레드의 차이점에 대해 설명해주세요. 멀티프로세스와 멀티스레드의 장단점을 비교해주세요."

## 답변 구조

### 1. 프로세스(Process)

#### 1.1 정의
- 실행 중인 프로그램의 인스턴스
- 독립된 메모리 공간(Code, Data, Stack, Heap)을 할당받음
- OS로부터 시스템 자원을 할당받는 작업의 단위

#### 1.2 프로세스의 메모리 구조
```
+------------------+
|       Stack      | → 지역변수, 함수호출 정보
+------------------+
|        ↓         | → 스택 증가 방향
+------------------+
|        ↑         | → 힙 증가 방향
+------------------+
|       Heap       | → 동적 할당 메모리
+------------------+
|       Data       | → 전역변수, static 변수
+------------------+
|       Code       | → 프로그램 코드
+------------------+
```

### 2. 스레드(Thread)

#### 2.1 정의
- 프로세스 내에서 실행되는 작업의 단위
- 스택을 제외한 나머지 메모리 영역(Code, Data, Heap)을 공유
- Light Weight Process(LWP)라고도 함

#### 2.2 스레드의 메모리 구조
```
Process
+------------------+
|    Thread 1      |     Thread 2     |
|  +----------+    |    +----------+  |
|  |  Stack   |    |    |  Stack   |  |
|  +----------+    |    +----------+  |
|       ↓          |         ↓        |
+------------------+------------------+
|              Heap (공유)           |
+----------------------------------+
|              Data (공유)           |
+----------------------------------+
|              Code (공유)           |
+----------------------------------+
```

### 3. 멀티프로세스 vs 멀티스레드

#### 3.1 멀티프로세스
장점:
- 프로세스 간 독립된 메모리로 안정성 높음
- 하나의 프로세스가 죽어도 다른 프로세스에 영향 없음

단점:
- 프로세스 생성과 컨텍스트 스위칭 비용이 큼
- IPC(Inter-Process Communication)가 필요하며 구현이 복잡

```c
// 프로세스 생성 예시 (Unix/Linux)
pid_t pid = fork();
if (pid == 0) {
    // 자식 프로세스
    execv("/path/to/program", args);
} else if (pid > 0) {
    // 부모 프로세스
    wait(NULL);
}
```

#### 3.2 멀티스레드
장점:
- 메모리 공유로 자원 효율성 높음
- 컨텍스트 스위칭 비용이 적음
- 스레드 간 통신이 쉬움

단점:
- 스레드 하나가 죽으면 전체 프로세스에 영향
- 동기화 문제(Race Condition)가 발생할 수 있음

```java
// Java에서의 스레드 생성 예시
Thread thread = new Thread(() -> {
    // 스레드에서 실행할 코드
    System.out.println("New thread running");
});
thread.start();
```

### 4. 동기화 문제와 해결

#### 4.1 Race Condition 예시
```java
public class Counter {
    private int count = 0;
    
    // 동기화되지 않은 메서드
    public void increment() {
        count++;  // Race Condition 발생 가능
    }
    
    // 동기화된 메서드
    public synchronized void safeIncrement() {
        count++;  // Thread Safe
    }
}
```

## 근거 자료

### 1. 운영체제 교과서 및 문서
- [Operating System Concepts, 10th Edition](https://www.os-book.com/OS10/)
    - Chapter 3: Processes
    - Chapter 4: Threads & Concurrency
- [Modern Operating Systems, 4th Edition](https://www.pearson.com/en-us/subject-catalog/p/modern-operating-systems/P200000003295)
    - Chapter 2: Processes and Threads

### 2. 공식 문서 및 표준
- [POSIX Threads Programming](https://computing.llnl.gov/tutorials/pthreads/)
- [Java Thread Documentation](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/Thread.html)
- [Linux man pages - fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html)

### 3. 학술 논문
- ["Processes and Threads in Android"](https://doi.org/10.1109/ACCESS.2019.2926623)
    - IEEE Access Journal
- ["An Implementation of POSIX Threads under UNIX"](https://doi.org/10.1145/506954.506955)
    - USENIX Winter Conference

### 4. 실무 관련 자료
- [Java Concurrency in Practice](https://jcip.net/)
- [Linux Kernel Documentation - Process Management](https://www.kernel.org/doc/html/latest/admin-guide/pm/)

## 실제 면접에서 나올 수 있는 추가 질문

1. "스레드의 컨텍스트 스위칭이 프로세스의 컨텍스트 스위칭보다 빠른 이유는 무엇인가요?"

2. "프로세스 간 통신(IPC) 방법들에 대해 설명해주세요."

3. "멀티코어 환경에서 멀티스레드 프로그래밍시 주의해야 할 점은 무엇인가요?"

4. "데드락(Deadlock)의 발생 조건과 해결 방법에 대해 설명해주세요."

## 실무 경험 예시

실제 면접에서는 다음과 같은 경험을 공유하면 좋습니다:

1. 멀티스레드 환경에서 발생한 동기화 문제 해결 경험
```java
// 문제가 있는 코드
private Map<String, User> userCache = new HashMap<>();

// 개선된 코드
private Map<String, User> userCache = new ConcurrentHashMap<>();
```

2. 성능 개선을 위해 멀티스레드를 적용한 경험
```java
// 단일 스레드 처리
for (String file : files) {
    processFile(file);
}

// 멀티스레드 처리로 개선
ExecutorService executor = Executors.newFixedThreadPool(4);
for (String file : files) {
    executor.submit(() -> processFile(file));
}
```

3. 프로세스 간 통신을 구현한 경험
```java
// Socket을 이용한 IPC 구현 예시
ServerSocket server = new ServerSocket(8080);
Socket client = server.accept();
// 데이터 통신 처리
```

이러한 실무 경험을 공유하면서, 발생했던 문제점과 해결 과정, 그리고 배운 점을 함께 설명하면 더욱 효과적인 답변이 될 수 있습니다.