# JVM 아키텍처와 실행 과정

## 핵심 질문

"JVM의 아키텍처를 구성하는 주요 컴포넌트들의 동작 방식과 Java 애플리케이션의 실행 프로세스를 상세히 설명해주세요."

## 1. 클래스 로더 서브시스템 (Class Loader Subsystem)

### 1.1 로딩 (Loading)

```java
public class LoadingPhaseExample {
    public static void main(String[] args) throws ClassNotFoundException {
        // 1. Bootstrap Class Loader
        // - JDK의 core classes 로드 (rt.jar, JAVA_HOME/jre/lib)
        URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
        for (URL url : urls) {
            System.out.println(url.toExternalForm());
        }

        // 2. Extension Class Loader
        // - JAVA_HOME/jre/lib/ext 디렉토리의 클래스 로드
        ClassLoader extClassLoader = ClassLoader.getSystemClassLoader().getParent();

        // 3. Application Class Loader
        // - 애플리케이션 클래스패스의 클래스들을 로드
        ClassLoader appClassLoader = LoadingPhaseExample.class.getClassLoader();

        // 동적 클래스 로딩 예시
        Class<?> dynamicClass = Class.forName("com.example.DynamicClass");
    }
}
```

### 1.2 링킹 (Linking)

```java
public class LinkingPhaseExample {
    // 1. Verification Phase
    // - 바이트코드 검증
    // - 포맷 검사, 액세스 규칙 확인 등

    // 2. Preparation Phase
    static int staticVar;  // 기본값 0으로 초기화
    static final int CONSTANT = 100;  // 상수는 이 단계에서 할당

    // 3. Resolution Phase
    static String message = "Hello";  // 심볼릭 레퍼런스를 실제 레퍼런스로 변환

    public static void main(String[] args) {
        System.out.println("Static Variable: " + staticVar);  // 0 출력
        System.out.println("Constant: " + CONSTANT);  // 100 출력
        System.out.println("Message: " + message);  // "Hello" 출력
    }
}
```

### 1.3 초기화 (Initialization)

```java
public class InitializationPhaseExample {
    // 정적 초기화 블록과 정적 변수 초기화
    static int counter;
    static final String APP_NAME;
    static List<String> items = new ArrayList<>();

    // 정적 초기화 블록
    static {
        counter = 1;
        APP_NAME = "MyApp";
        items.add("First Item");
        System.out.println("Static initialization completed");
    }

    public static void main(String[] args) {
        System.out.println("Counter: " + counter);
        System.out.println("App Name: " + APP_NAME);
        System.out.println("Items: " + items);
    }
}
```

## 2. Runtime Data Area

### 2.1 Method Area (Static Area)

```java
public class MethodAreaExample {
    // 메소드 영역에 저장되는 정보

    // 1. 클래스 구조 정보
    static class InnerClass {
    }

    // 2. 상수 풀 (Constant Pool)
    static final String CONSTANT = "This is constant";
    static final int MAX_VALUE = 100;

    // 3. 정적 변수
    static int counter = 0;

    // 4. 메소드 데이터
    public static void staticMethod() {
        System.out.println("Static method");
    }

    // 5. 생성자 데이터
    public MethodAreaExample() {
        counter++;
    }
}
```

### 2.2 Heap Area

```java
public class HeapAreaExample {
    public static void main(String[] args) {
        // 1. Young Generation

        // Eden Space
        Object newObject = new Object();  // 새로운 객체는 Eden에 할당

        // Survivor Spaces (S0, S1)
        // Minor GC 발생 시 살아남은 객체들이 이동

        // 2. Old Generation
        // Long-lived 객체들이 저장되는 영역
        byte[] largeArray = new byte[1024 * 1024 * 10];  // 10MB

        // 3. String Pool
        String str1 = "Hello";  // String Pool에 저장
        String str2 = new String("Hello");  // Heap에 저장
        String str3 = str2.intern();  // String Pool에서 참조
    }
}
```

### 2.3 Stack Area

```java
public class StackAreaExample {
    public static void main(String[] args) {
        // 1. Stack Frame for main method
        int mainVar = 10;

        // 2. Method Invocation
        calculateSum(5);
    }

    private static int calculateSum(int n) {
        // New Stack Frame
        int result = 0;  // Local Variable

        // Operand Stack
        for (int i = 1; i <= n; i++) {
            result += i;
        }

        return result;  // Frame Popped after return
    }
}
```

### 2.4 PC Register & Native Method Stack

```java
public class PCRegisterExample {
    public static void main(String[] args) {
        // PC Register: 현재 실행 중인 JVM 명령의 주소를 저장
        Thread thread = new Thread(() -> {
            // 각 스레드마다 별도의 PC Register 보유
            for (int i = 0; i < 1000; i++) {
                // JVM 명령어 실행
            }
        });

        // Native Method Stack
        thread.start();  // Native method 호출
    }

    // Native 메소드 예시
    public native void nativeOperation();
}
```

## 3. 실행 엔진 (Execution Engine)

### 3.1 인터프리터

```java
public class InterpreterExample {
    public static void main(String[] args) {
        // 인터프리터에 의해 한 줄씩 해석되고 실행
        int a = 10;
        int b = 20;
        int sum = a + b;
        System.out.println("Sum: " + sum);
    }
}
```

### 3.2 JIT 컴파일러

```java
public class JITCompilerExample {
    public static void main(String[] args) {
        long startTime = System.nanoTime();

        // Hot spot: JIT 컴파일 대상
        for (int i = 0; i < 10000; i++) {
            calculateFibonacci(10);
        }

        long endTime = System.nanoTime();
        System.out.println("Execution time: " + (endTime - startTime));
    }

    private static int calculateFibonacci(int n) {
        if (n <= 1) return n;
        return calculateFibonacci(n - 1) + calculateFibonacci(n - 2);
    }
}
```

### 3.3 가비지 컬렉터

```java
public class GCExample {
    public static void main(String[] args) {
        // 1. 객체 생성 (Eden 영역)
        Object obj1 = new Object();
        Object obj2 = new Object();

        // 2. 참조 제거 (GC 대상)
        obj1 = null;

        // 3. GC 제안 (강제성 없음)
        System.gc();

        // 4. Finalization
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Cleanup before JVM shutdown");
        }));
    }

    @Override
    protected void finalize() {
        // 객체 소멸 전 호출
        System.out.println("Finalize method called");
    }
}
```

## 4. JVM 모니터링과 성능 최적화

### 4.1 JVM 모니터링

```java

@Component
@Slf4j
public class JVMMonitorExample {
    @Scheduled(fixedRate = 60000)
    public void monitorJVM() {
        // 1. Memory 모니터링
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();

        log.info("Heap Memory: {}/{}",
                heapUsage.getUsed() / 1048576,
                heapUsage.getMax() / 1048576);

        // 2. Thread 모니터링
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        log.info("Thread Count: {}", threadBean.getThreadCount());

        // 3. Class Loading 모니터링
        ClassLoadingMXBean classBean = ManagementFactory.getClassLoadingMXBean();
        log.info("Loaded Class Count: {}", classBean.getLoadedClassCount());

        // 4. GC 모니터링
        List<GarbageCollectorMXBean> gcBeans =
                ManagementFactory.getGarbageCollectorMXBeans();
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            log.info("GC {}: Count={}, Time={}ms",
                    gcBean.getName(),
                    gcBean.getCollectionCount(),
                    gcBean.getCollectionTime());
        }
    }
}
```

## 5. JVM 튜닝 파라미터

### 5.1 메모리 관련 설정

```bash
# 힙 메모리 설정
-Xms2g  # 초기 힙 크기
-Xmx2g  # 최대 힙 크기

# Young Generation 크기 설정
-XX:NewSize=512m
-XX:MaxNewSize=512m

# 메타스페이스 설정
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
```

### 5.2 GC 관련 설정

```bash
# G1 GC 사용
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=8m

# GC 로깅
-Xlog:gc*:file=gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

## 참고 자료

1. [Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se11/html/index.html)
2. [JVM Internals](https://blog.jamesdbloom.com/JVMInternals.html)
3. [Understanding Java Garbage Collection](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
4. Oracle JVM Tuning Guide

## 실무 연관 질문

1. "JVM 튜닝 경험이 있다면, 어떤 문제가 있었고 어떻게 해결했나요?"

2. "OOM (Out of Memory) 에러를 경험한 적이 있나요? 어떻게 해결했나요?"

3. "프로덕션 환경에서 GC 튜닝은 어떤 방식으로 진행했나요?"

4. "Thread Dump를 분석해본 경험이 있나요?"

## 면접 답변 핵심 포인트

1. JVM 아키텍처의 각 컴포넌트 역할 이해
2. 클래스 로딩 프로세스의 단계별 이해
3. 메모리 영역의 특징과 용도
4. GC 동작 방식과 튜닝 방법
5. 실무에서의 문제 해결 경험 공유

# JVM 튜닝 경험 사례

## 1. 문제 상황: 주기적인 성능 저하와 응답 지연

### 1.1 증상

```java
// 모니터링에서 발견된 패턴
public class PerformanceMonitoring {
    public void checkMetrics() {
        // 매 4시간마다 응답시간 증가
        // API 응답시간: 평소 200ms -> 급격히 2000ms로 증가
        // CPU 사용률: 일시적으로 90% 이상 스파이크
    }
}
```

### 1.2 원인 분석

```java

@Service
@Slf4j
public class ProblemAnalysis {
    public void analyzeGCLogs() {
        // 1. GC 로그 분석 결과
        // - Full GC가 자주 발생 (4시간마다)
        // - Old Generation이 빠르게 차는 현상

        // 2. Heap Dump 분석
        // - 대용량 캐시 객체가 Old Generation에 누적
        // - 캐시 데이터의 생명주기 관리 부재
    }
}
```

## 2. 해결 과정

### 2.1 즉각적인 조치

```java
// 1. JVM 옵션 조정
-XX:+UseG1GC
-Xms4g
-Xmx4g
-XX:MaxGCPauseMillis=200
        -XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xlog:gc*:file=gc.log

// 2. GC 로깅 강화
@Component

public class GCMonitoring {
    @Scheduled(fixedRate = 300000) // 5분마다
    public void logGCMetrics() {
        for (GarbageCollectorMXBean gc :
                ManagementFactory.getGarbageCollectorMXBeans()) {
            log.info("GC {}: Count={}, Time={}ms",
                    gc.getName(),
                    gc.getCollectionCount(),
                    gc.getCollectionTime());
        }
    }
}
```

### 2.2 근본적인 해결

```java

@Service
public class CacheService {
    // 기존 코드
    private static final Map<String, Object> cache = new HashMap<>();

    // 개선된 코드: 캐시 라이브러리 도입
    @Autowired
    private CaffeineCacheManager cacheManager;

    public void configureCacheManager() {
        // 캐시 설정
        Cache<String, Object> cache = Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofHours(1))
                .recordStats()
                .build();
    }
}
```

## 3. 결과 및 개선사항

### 3.1 성능 메트릭스

```java
public class PerformanceResults {
    // 개선 전
    // - Full GC 빈도: 6회/일
    // - 평균 GC 시간: 800ms
    // - 응답시간: p99 2000ms

    // 개선 후
    // - Full GC 빈도: 1회/일 미만
    // - 평균 GC 시간: 200ms
    // - 응답시간: p99 300ms
}
```

### 3.2 모니터링 체계 구축

```java

@Configuration
public class MonitoringConfig {
    @Bean
    public MeterRegistry meterRegistry() {
        // Prometheus + Grafana 설정
        return new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
    }

    @Bean
    public GCMonitor gcMonitor(MeterRegistry registry) {
        // GC 메트릭 수집
        return new GCMonitor(registry);
    }
}
```

## 4. 교훈 및 배운 점

1. 선제적 모니터링의 중요성

```java

@Component
public class PreemptiveMonitoring {
    // 1. 임계치 기반 알람 설정
    private static final double MEMORY_THRESHOLD = 0.8; // 80%

    @Scheduled(fixedRate = 60000)
    public void checkMemoryUsage() {
        Runtime runtime = Runtime.getRuntime();
        double memoryUsage =
                (double) (runtime.totalMemory() - runtime.freeMemory())
                        / runtime.maxMemory();

        if (memoryUsage > MEMORY_THRESHOLD) {
            alertOperations();
        }
    }
}
```

2. 튜닝 프로세스 정립

```java
public class TuningProcess {
    public void tuningSteps() {
        // 1. 기준 메트릭 설정
        // 2. 데이터 수집
        // 3. 분석 및 가설
        // 4. 테스트 환경 검증
        // 5. 단계적 적용
        // 6. 모니터링 및 피드백
    }
}
```

## 5. 향후 계획

1. 자동화된 모니터링 체계

```java

@Configuration
public class AutomatedMonitoring {
    @Bean
    public AlertManager alertManager() {
        return AlertManager.builder()
                .withGCThreshold(Duration.ofMillis(500))
                .withMemoryThreshold(0.8)
                .withResponseTimeThreshold(Duration.ofMillis(1000))
                .build();
    }
}
```

2. 성능 테스트 자동화

```java

@SpringBootTest
public class PerformanceTest {
    @Test
    public void loadTest() {
        // JMeter 또는 Gatling 기반 부하 테스트
        // 정기적인 성능 지표 수집
        // 자동화된 리포트 생성
    }
}
```

## 핵심 포인트

1. 문제 상황을 파악할 수 있는 정확한 모니터링이 중요
2. 임시 방편이 아닌 근본적인 원인 해결 필요
3. 점진적이고 안전한 적용이 중요
4. 변경 후 모니터링과 피드백이 필수
5. 문서화와 지식 공유를 통한 재발 방지

# Out of Memory Error 경험과 해결 과정

## 1. 문제 상황 발생

### 1.1 현상

```java
// Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
@RestController
@Slf4j
public class DataProcessingController {
    @GetMapping("/process-large-data")
    public ResponseEntity<List<ResultDTO>> processData() {
        // 대용량 데이터 처리 중 OOM 발생
        // 수백만 건의 데이터를 한 번에 메모리에 로드
        List<ResultDTO> results = new ArrayList<>();
        try {
            results = dataService.processLargeDataSet();
        } catch (OutOfMemoryError e) {
            log.error("OOM occurred while processing data", e);
            // 에러 처리
        }
        return ResponseEntity.ok(results);
    }
}
```

### 1.2 초기 분석

```java
public class MemoryAnalysis {
    public void analyzeHeapDump() {
        // heap dump 분석 결과
        // 1. 대량의 ResultDTO 객체가 heap에 적재
        // 2. 컬렉션 객체가 지속적으로 증가
        // 3. 메모리 회수가 제대로 되지 않는 상황
    }
}
```

## 2. 문제 해결 과정

### 2.1 임시 조치

```java
// 1. JVM 힙 메모리 증가
-Xms4g
-Xmx8g

// 2. GC 로깅 활성화
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

### 2.2 근본적인 해결책

#### 2.2.1 페이지네이션 도입

```java

@Service
public class DataService {
    private static final int PAGE_SIZE = 1000;

    public List<ResultDTO> processLargeDataSetWithPagination() {
        List<ResultDTO> results = new ArrayList<>();
        long totalCount = repository.count();

        for (int page = 0; page < (totalCount / PAGE_SIZE) + 1; page++) {
            Pageable pageable = PageRequest.of(page, PAGE_SIZE);
            List<Data> pageData = repository.findAll(pageable).getContent();
            results.addAll(processDataBatch(pageData));
        }
        return results;
    }
}
```

#### 2.2.2 스트림 처리 도입

```java

@Service
public class StreamProcessingService {
    @Autowired
    private DataRepository repository;

    public void processWithStream() {
        try (Stream<Data> dataStream = repository.streamAll()) {
            dataStream
                    .filter(this::isValid)
                    .map(this::transform)
                    .forEach(this::save);
        }
    }
}
```

#### 2.2.3 배치 처리 구현

```java

@Configuration
@EnableBatchProcessing
public class BatchConfig {
    @Bean
    public Job processDataJob(JobBuilderFactory jobBuilderFactory,
                              StepBuilderFactory stepBuilderFactory) {
        return jobBuilderFactory.get("processDataJob")
                .start(processDataStep(stepBuilderFactory))
                .build();
    }

    @Bean
    public Step processDataStep(StepBuilderFactory stepBuilderFactory) {
        return stepBuilderFactory.get("processDataStep")
                .<Data, ResultDTO>chunk(1000)
                .reader(dataItemReader())
                .processor(dataProcessor())
                .writer(resultWriter())
                .build();
    }
}
```

## 3. 모니터링 체계 구축

### 3.1 메모리 사용량 모니터링

```java

@Component
@Slf4j
public class MemoryMonitor {
    @Scheduled(fixedRate = 60000)
    public void monitorMemoryUsage() {
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        long maxMemory = runtime.maxMemory();

        double memoryUsagePercent = ((double) usedMemory / maxMemory) * 100;

        log.info("Memory Usage: {}%, Used: {}MB, Max: {}MB",
                String.format("%.2f", memoryUsagePercent),
                usedMemory / (1024 * 1024),
                maxMemory / (1024 * 1024));
    }
}
```

### 3.2 알림 시스템 구축

```java

@Service
public class AlertService {
    private static final double MEMORY_THRESHOLD = 0.85; // 85%

    public void checkMemoryThreshold() {
        double memoryUsage = getMemoryUsage();
        if (memoryUsage > MEMORY_THRESHOLD) {
            sendAlert("High memory usage detected: " +
                    String.format("%.2f%%", memoryUsage * 100));
        }
    }

    private void sendAlert(String message) {
        // Slack, Email 등으로 알림 발송
    }
}
```

## 4. 예방을 위한 조치

### 4.1 메모리 누수 방지

```java
public class ResourceManagement {
    // try-with-resources 사용
    public void processStream() {
        try (InputStream is = new FileInputStream("large-file.txt")) {
            // 처리 로직
        } catch (IOException e) {
            log.error("Error processing file", e);
        }
    }

    // 캐시 관리
    @Bean
    public CacheManager cacheManager() {
        return new CaffeineCacheManager()
                .builder()
                .maximumSize(1000)
                .expireAfterWrite(Duration.ofHours(1))
                .build();
    }
}
```

### 4.2 리소스 관리

```java

@Configuration
public class ResourceConfig {
    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        return executor;
    }
}
```

## 5. 교훈과 모범 사례

1. 대용량 데이터 처리 원칙:
    - 페이지네이션 필수
    - 스트림 처리 활용
    - 배치 처리 고려

2. 모니터링 중요성:
    - 메모리 사용량 상시 모니터링
    - 임계치 기반 알림 설정
    - 히스토리 트래킹

3. 설계 단계 고려사항:
    - 메모리 사용량 예측
    - 확장성 고려
    - 리소스 제한 설정

# Thread Dump 분석 경험

## 1. 문제 상황 발견

### 1.1 시스템 증상

```java

@RestController
public class OrderController {
    @GetMapping("/orders")
    public ResponseEntity<List<Order>> getOrders() {
        // API 응답 지연 현상 발생
        // 평소 응답시간 200ms -> 5000ms로 증가
        // 특정 시간대에 CPU 사용률 급증
        return ResponseEntity.ok(orderService.findAll());
    }
}
```

### 1.2 Thread Dump 획득

```java
public class ThreadDumpUtil {
    public static void generateThreadDump() {
        // 1. jstack 사용
        // jstack <pid> > thread_dump.txt

        // 2. JMX를 통한 획득
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(true, true);

        // 3. Spring Actuator 사용
        // /actuator/threaddump 엔드포인트
    }
}
```

## 2. Thread Dump 분석 과정

### 2.1 데드락 확인

```java

@Service
public class DeadlockAnalysis {
    public void analyzeLocks() {
        // Thread Dump 예시
        /*
        "Thread-1" #42 prio=5 os_prio=0 tid=0x00007f6c0c063000 nid=0x1234 waiting for monitor entry [0x00007f6c0c063000]
           java.lang.Thread.State: BLOCKED (on object monitor)
            at com.example.Service.methodA(Service.java:15)
            - locked <0x00000007d58f5e70> (a java.lang.Object)
            at com.example.Service.methodB(Service.java:30)
        
        "Thread-2" #43 prio=5 os_prio=0 tid=0x00007f6c0c064000 nid=0x1235 waiting for monitor entry [0x00007f6c0c064000]
           java.lang.Thread.State: BLOCKED (on object monitor)
            at com.example.Service.methodB(Service.java:30)
            - locked <0x00000007d58f5e71> (a java.lang.Object)
            at com.example.Service.methodA(Service.java:15)
        */
    }
}
```

### 2.2 스레드 상태 분석

```java

@Component
public class ThreadStateAnalyzer {
    public void analyzeThreadStates() {
        Map<Thread.State, Integer> stateCount = new HashMap<>();

        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadBean.dumpAllThreads(true, true);

        for (ThreadInfo info : threadInfos) {
            Thread.State state = info.getThreadState();
            stateCount.merge(state, 1, Integer::sum);
        }

        // Thread 상태별 카운트 출력
        stateCount.forEach((state, count) ->
                log.info("{}: {}", state, count));
    }
}
```

### 2.3 Hot Spot 식별

```java
public class HotSpotAnalysis {
    public void analyzeHotSpots() {
        // CPU 사용률이 높은 스레드 확인
        /*
        "http-nio-8080-exec-5" #25 daemon prio=5 tid=0x00007f6c0c065000 nid=0x1236 runnable [0x00007f6c0c065000]
           java.lang.Thread.State: RUNNABLE
            at com.example.service.HeavyComputationService.compute(HeavyComputationService.java:45)
            at com.example.controller.ApiController.process(ApiController.java:30)
        */

        // 반복적으로 발생하는 패턴 분석
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        long[] threadIds = threadBean.findMonitorDeadlockedThreads();
        if (threadIds != null) {
            ThreadInfo[] threadInfos = threadBean.getThreadInfo(threadIds);
            // 데드락 상태의 스레드 정보 분석
        }
    }
}
```

## 3. 문제 해결 과정

### 3.1 동시성 이슈 해결

```java

@Service
public class ImprovedService {
    // 기존 코드
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    // 개선된 코드: Lock 순서 정규화
    private final ReentrantLock lock = new ReentrantLock();

    public void processWithLock() {
        try {
            if (lock.tryLock(5, TimeUnit.SECONDS)) {
                try {
                    // 크리티컬 섹션
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 3.2 스레드 풀 최적화

```java

@Configuration
public class ThreadPoolConfig {
    @Bean
    public ThreadPoolTaskExecutor applicationTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // CPU 코어 수 기반 설정
        int corePoolSize = Runtime.getRuntime().availableProcessors();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(corePoolSize * 2);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("custom-exec-");

        // 포화 상태 정책 설정
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());

        return executor;
    }
}
```

## 4. 모니터링 체계 구축

### 4.1 자동화된 Thread Dump 수집

```java

@Component
@Slf4j
public class ThreadDumpCollector {
    @Scheduled(fixedRate = 300000) // 5분마다
    public void collectThreadDump() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadBean.dumpAllThreads(true, true);

        // 임계치 초과 시 Thread Dump 저장
        if (isThresholdExceeded(threadInfos)) {
            saveThreadDump(threadInfos);
        }
    }

    private boolean isThresholdExceeded(ThreadInfo[] threadInfos) {
        long blockedThreads = Arrays.stream(threadInfos)
                .filter(info -> info.getThreadState() == Thread.State.BLOCKED)
                .count();

        return blockedThreads > 10; // 임계치
    }
}
```

### 4.2 Thread 상태 시각화

```java

@RestController
@RequestMapping("/monitoring")
public class ThreadMonitoringController {
    @GetMapping("/threads/status")
    public Map<String, Object> getThreadStatus() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadBean.dumpAllThreads(false, false);

        Map<Thread.State, Long> stateCount = Arrays.stream(threadInfos)
                .collect(Collectors.groupingBy(
                        ThreadInfo::getThreadState,
                        Collectors.counting()
                ));

        Map<String, Object> status = new HashMap<>();
        status.put("totalThreads", threadBean.getThreadCount());
        status.put("stateDistribution", stateCount);

        return status;
    }
}
```

## 5. 예방적 조치

### 5.1 코드 리뷰 체크리스트

```java
public class ThreadSafetyChecklist {
    // 1. synchronized 블록 최소화
    // 2. Lock 순서 일관성 유지
    // 3. try-finally에서 lock 해제
    // 4. 데드락 가능성 검토
    // 5. 스레드 풀 설정 적절성 검토
}
```

## 핵심 교훈

1. Thread Dump는 주기적으로 수집하여 패턴 분석
2. 동시성 이슈는 설계 단계에서 고려
3. 모니터링 자동화로 선제적 대응
4. 스레드 풀 설정은 시스템 특성에 맞게 조정
5. 로킹 전략은 최소화하고 명확하게 구현