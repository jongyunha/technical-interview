# JVM과 가비지 컬렉션에 대한 기술 면접 답변

## 주요 질문
"JVM의 메모리 구조와 가비지 컬렉션의 동작 방식에 대해 설명해주세요. 각 GC 알고리즘의 장단점은 무엇인가요?"

## 1. JVM 메모리 구조

### 1.1 Heap 영역
```java
public class HeapStructure {
    // Young Generation
    // - Eden
    // - Survivor 0
    // - Survivor 1
    
    // Old Generation
    
    public static void main(String[] args) {
        // Eden 영역에 객체 생성
        Object obj = new Object();
        
        // 큰 객체는 바로 Old Generation으로
        byte[] bigArray = new byte[1024 * 1024 * 10]; // 10MB
    }
}
```

### 1.2 Non-Heap 영역
```java
public class NonHeapStructure {
    // Method Area (Metaspace in Java 8+)
    static final String CONSTANT = "This is constant";
    
    // Stack
    public void stackExample() {
        int localVar = 42;  // Stack에 저장
        Object obj = new Object();  // 참조는 Stack, 객체는 Heap
    }
}
```

## 2. 가비지 컬렉션 알고리즘

### 2.1 Serial GC
```java
// Serial GC 사용 설정
// -XX:+UseSerialGC
public class SerialGCExample {
    public static void main(String[] args) {
        System.out.println("GC Algorithm: " + 
            ManagementFactory.getGarbageCollectorMXBeans()
            .stream()
            .map(GarbageCollectorMXBean::getName)
            .collect(Collectors.joining(", ")));
    }
}
```

### 2.2 Parallel GC
```java
// Parallel GC 설정
// -XX:+UseParallelGC
// -XX:ParallelGCThreads=4
public class ParallelGCExample {
    private static final int MB = 1024 * 1024;
    
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            list.add(new byte[MB]); // 메모리 할당
        }
    }
}
```

### 2.3 G1 GC
```java
// G1 GC 설정
// -XX:+UseG1GC
// -XX:MaxGCPauseMillis=200
public class G1GCExample {
    public static void main(String[] args) {
        // G1 GC 모니터링
        for (GarbageCollectorMXBean gc : 
            ManagementFactory.getGarbageCollectorMXBeans()) {
            long count = gc.getCollectionCount();
            long time = gc.getCollectionTime();
            String name = gc.getName();
            System.out.printf("GC %s: %d collections, %dms%n", 
                name, count, time);
        }
    }
}
```

## 3. GC 모니터링과 튜닝

### 3.1 GC 로깅 설정
```bash
# GC 로깅 옵션
-verbose:gc
-Xlog:gc*:file=gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

### 3.2 메모리 릭 탐지
```java
public class MemoryLeakDetector {
    private static final List<Object> leakyList = new ArrayList<>();
    
    public static void detectLeak() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        
        System.out.printf("Memory used: %d MB%n", 
            heapUsage.getUsed() / 1024 / 1024);
        
        if ((double) heapUsage.getUsed() / heapUsage.getMax() > 0.8) {
            System.out.println("Potential memory leak detected!");
        }
    }
}
```

## 4. 실제 튜닝 예시

### 4.1 힙 크기 설정
```bash
# 힙 크기 설정
-Xms4g  # 초기 힙 크기
-Xmx4g  # 최대 힙 크기
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
```

### 4.2 GC 튜닝
```bash
# G1 GC 튜닝
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=8m
-XX:InitiatingHeapOccupancyPercent=45
```

## 근거 자료

### 1. 공식 문서
- [JVM Specification](https://docs.oracle.com/javase/specs/jvms/se11/html/index.html)
- [HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/en/java/javase/11/gctuning/index.html)
- [JDK Tools and Utilities](https://docs.oracle.com/javase/8/docs/technotes/tools/)

### 2. 기술 문서
- [Understanding Java Garbage Collection](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
- [Memory Management in the Java HotSpot Virtual Machine](https://www.oracle.com/technetwork/java/javase/tech/memorymanagement-whitepaper-1-150020.pdf)

## 실무 관련 추가 질문

1. "실제 프로덕션 환경에서 GC 튜닝을 해보신 경험이 있나요?"

2. "메모리 릭을 발견하고 해결한 경험을 설명해주세요."

3. "어떤 GC 알고리즘을 선호하시나요? 그 이유는?"

4. "GC 로그를 어떻게 분석하시나요?"

## 실제 사용 예시

### GC 모니터링 코드
```java
public class GCMonitor {
    public static void monitorGC() {
        List<GarbageCollectorMXBean> gcBeans = 
            ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println(gcBean.getName());
            System.out.println("Collection Count: " + 
                gcBean.getCollectionCount());
            System.out.println("Collection Time: " + 
                gcBean.getCollectionTime() + "ms");
        }
    }
}
```

### 메모리 사용량 모니터링
```java
@Component
@Slf4j
public class MemoryMonitor {
    @Scheduled(fixedRate = 60000) // 1분마다 실행
    public void checkMemory() {
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        log.info("Memory Usage: {}MB / {}MB", 
            usedMemory/1024/1024, 
            totalMemory/1024/1024);
    }
}
```

실제 면접에서는 이론적인 지식과 함께 GC 튜닝 경험, 메모리 문제 해결 경험, 성능 최적화 경험 등을 구체적으로 설명하는 것이 좋습니다.

# JVM의 각 GC(Garbage Collection) 프로세스와 장단점

## 1. Serial GC

### 프로세스
1. Young Generation (Minor GC)
   ```plaintext
   1. Eden 영역에서 객체 생성
   2. Eden 영역이 가득 차면 GC 발생
   3. 살아있는 객체를 Survivor 영역으로 이동
   4. Eden 영역을 비움
   ```

2. Old Generation (Major GC)
   ```plaintext
   1. Mark-Sweep-Compact 알고리즘 사용
   2. Mark: 살아있는 객체 식별
   3. Sweep: 죽은 객체 제거
   4. Compact: 살아있는 객체들을 한쪽으로 모음
   ```

### 장단점
- 장점:
    - 단순한 알고리즘으로 구현이 간단
    - 싱글 스레드 환경에서 효율적
    - 작은 메모리에서 효과적
- 단점:
    - Stop-the-World 시간이 김
    - 멀티코어 환경에서 비효율적
    - 큰 힙 메모리에서 성능 저하

## 2. Parallel GC

### 프로세스
1. Young Generation
   ```plaintext
   1. 여러 스레드가 동시에 GC 수행
   2. Eden 영역의 살아있는 객체들을 병렬로 Survivor로 이동
   3. 빈 Eden 영역을 한 번에 비움
   ```

2. Old Generation
   ```plaintext
   1. Mark 단계: 여러 스레드가 동시에 살아있는 객체 탐색
   2. Sweep 단계: 병렬로 죽은 객체 제거
   3. Compact 단계: 살아있는 객체들을 병렬로 재배치
   ```

### 장단점
- 장점:
    - Minor GC, Major GC 모두 병렬 처리로 빠른 처리
    - 높은 처리량(Throughput)
    - 멀티코어 환경에서 효율적
- 단점:
    - Stop-the-World 시간은 여전히 존재
    - 메모리와 CPU 사용량이 증가
    - 일시 정지 시간이 예측 불가능

## 3. CMS(Concurrent Mark Sweep) GC

### 프로세스
```plaintext
1. Initial Mark (STW)
   - GC Root에서 직접 참조하는 객체만 마킹
   
2. Concurrent Mark
   - 애플리케이션 실행과 동시에 마킹
   - 참조된 객체들을 추적
   
3. Remark (STW)
   - Concurrent Mark 단계에서 변경된 부분 확인
   - 최종 마킹 작업
   
4. Concurrent Sweep
   - 애플리케이션 실행과 동시에 가비지 제거
```

### 장단점
- 장점:
    - Stop-the-World 시간이 매우 짧음
    - 응답 시간(Latency)이 중요한 애플리케이션에 적합
    - 실시간 처리가 필요한 시스템에 적합
- 단점:
    - CPU와 메모리 사용량이 높음
    - 메모리 단편화 문제 발생
    - Compaction 과정이 없음

## 4. G1(Garbage First) GC

### 프로세스
```plaintext
1. Young GC
   1) Eden 영역의 살아있는 객체를 Survivor 영역으로 이동
   2) 여러 스레드가 병렬로 작업
   
2. Mixed GC
   1) Initial Mark (STW)
      - GC Root 스캔
   2) Concurrent Mark
      - 전체 힙의 살아있는 객체 추적
   3) Remark (STW)
      - 최종 마킹 작업
   4) Cleanup (STW/Concurrent)
      - 빈 리전 회수 및 회수할 리전 계산
```

### 장단점
- 장점:
    - 큰 힙 메모리에서도 효율적
    - 예측 가능한 Stop-the-World 시간
    - 메모리 단편화 최소화
    - Region 단위의 메모리 관리로 유연함
- 단점:
    - CPU와 메모리 사용량이 더 높음
    - 복잡한 알고리즘으로 인한 튜닝의 어려움
    - 작은 힙에서는 오버헤드 발생

## 5. ZGC (Java 11+)

### 프로세스
```plaintext
1. Mark Start (STW - μs 단위)
   - GC Root에서 시작하는 객체 마킹 시작

2. Concurrent Mark and Remap
   - 애플리케이션 실행과 동시에 마킹
   - 메모리 페이지 재매핑

3. Mark End (STW - μs 단위)
   - 마킹 작업 완료

4. Concurrent Reset and Remap
   - 메모리 정리 및 재구성
```

### 장단점
- 장점:
    - 매우 짧은 Stop-the-World 시간(< 10ms)
    - 큰 힙 크기(수 TB)에서도 효율적
    - 메모리 단편화 문제 해결
- 단점:
    - 더 높은 CPU와 메모리 사용량
    - 아직 안정화 단계
    - 메모리 오버헤드 존재

이러한 각 GC의 특성을 이해하고 애플리케이션의 요구사항(응답시간, 처리량, 메모리 크기 등)에 맞는 적절한 GC를 선택하는 것이 중요합니다.