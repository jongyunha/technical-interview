# JVM Performance Optimization Guide

## 1. JIT(Just-In-Time) 컴파일러

### 1.1 Interpretation vs JIT Compilation

```java
public class CompilationExample {
    // 인터프리터 실행
    public void interpretedMethod() {
        // 매번 바이트코드를 해석하여 실행
        int result = 1 + 2;
    }

    // JIT 컴파일 대상
    public int hotMethod() {
        int sum = 0;
        // 자주 호출되는 메소드 (Hot Spot)
        for (int i = 0; i < 10000; i++) {
            sum += i;
        }
        return sum;
    }
}
```

### 1.2 Hot Spot Detection

```java
public class HotSpotExample {
    private static final int THRESHOLD = 10000;

    // JIT 컴파일러가 감지하는 Hot Spot 패턴들

    // 1. 반복적인 메소드 호출
    public void repeatedCall() {
        for (int i = 0; i < THRESHOLD; i++) {
            computeValue();  // 빈번한 호출로 인한 Hot Spot
        }
    }

    // 2. 루프 내부 코드
    public long loopComputation() {
        long sum = 0;
        for (int i = 0; i < THRESHOLD; i++) {
            sum += i * i;  // 루프 내부가 Hot Spot
        }
        return sum;
    }

    private int computeValue() {
        return 42;
    }
}
```

### 1.3 Optimization Techniques

```java
public class OptimizationExample {
    // 1. Method Inlining
    public int inlineExample(int x) {
        return computeSimple(x);  // 작은 메소드는 인라인화 됨
    }

    private int computeSimple(int x) {
        return x * 2;
    }

    // 2. Loop Unrolling
    public int sumArray(int[] array) {
        int sum = 0;
        // JIT가 최적화하여 여러 반복을 한번에 처리
        for (int i = 0; i < array.length; i++) {
            sum += array[i];
        }
        return sum;
    }

    // 3. Null Check Elimination
    public int processValue(Integer value) {
        // JIT는 불필요한 null 체크를 제거
        return value + 10;
    }
}
```

## 2. JVM Profiling

### 2.1 CPU Profiling

```java

@Service
@Slf4j
public class CPUProfilingExample {
    // CPU 사용량이 높은 작업 예시
    public void cpuIntensiveTask() {
        try {
            ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
            long startCpuTime = threadBean.getCurrentThreadCpuTime();

            // CPU 집중 작업 수행
            performHeavyComputation();

            long endCpuTime = threadBean.getCurrentThreadCpuTime();
            log.info("CPU Time used: {} ms", (endCpuTime - startCpuTime) / 1_000_000);

        } catch (Exception e) {
            log.error("Error during CPU intensive task", e);
        }
    }

    private void performHeavyComputation() {
        // CPU 집중 연산
        for (int i = 0; i < 1000000; i++) {
            Math.sqrt(i);
        }
    }
}
```

### 2.2 Memory Profiling

```java

@Component
public class MemoryProfilingExample {
    @Scheduled(fixedRate = 60000)
    public void memoryUsageMonitoring() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();

        log.info("Memory Usage: {}/{} MB",
                heapUsage.getUsed() / 1024 / 1024,
                heapUsage.getMax() / 1024 / 1024);

        // Heap Histogram 분석
        // jmap -histo <pid>
    }
}
```

## 3. 성능 최적화 기법

### 3.1 Object Pool

```java
public class ObjectPoolExample {
    public static class ExpensiveObject {
        // 생성 비용이 높은 객체
    }

    public class ObjectPool {
        private final Queue<ExpensiveObject> pool;
        private static final int MAX_POOL_SIZE = 100;

        public ObjectPool() {
            pool = new ConcurrentLinkedQueue<>();
            // 풀 초기화
            for (int i = 0; i < MAX_POOL_SIZE; i++) {
                pool.offer(new ExpensiveObject());
            }
        }

        public ExpensiveObject borrow() {
            ExpensiveObject object = pool.poll();
            return object != null ? object : new ExpensiveObject();
        }

        public void release(ExpensiveObject object) {
            if (pool.size() < MAX_POOL_SIZE) {
                pool.offer(object);
            }
        }
    }
}
```

### 3.2 String Interning

```java
public class StringInterningExample {
    public void stringPoolExample() {
        // String Pool 사용
        String s1 = "hello";
        String s2 = "hello";
        System.out.println(s1 == s2);  // true

        // 명시적 인터닝
        String s3 = new String("hello").intern();
        System.out.println(s1 == s3);  // true
    }
}
```

### 3.3 Escape Analysis

```java
public class EscapeAnalysisExample {
    // 스택 할당 가능한 객체
    public int computeSum() {
        Point point = new Point(10, 20);  // 메소드 범위를 벗어나지 않음
        return point.getX() + point.getY();
    }

    private static class Point {
        private final int x;
        private final int y;

        public Point(int x, int y) {
            this.x = x;
            this.y = y;
        }

        public int getX() {
            return x;
        }

        public int getY() {
            return y;
        }
    }
}
```

## 핵심 포인트

1. JIT 컴파일러:
    - 자주 실행되는 코드를 식별하여 네이티브 코드로 최적화
    - 런타임 성능 향상을 위한 다양한 최적화 기법 적용
    - 적절한 최적화 시점 결정이 중요

2. 프로파일링:
    - CPU, 메모리 사용량 모니터링이 필수
    - 병목 지점 식별 및 최적화
    - 적절한 프로파일링 도구 선택

3. 최적화 기법:
    - 상황에 맞는 최적화 전략 선택
    - 과도한 최적화 주의
    - 측정 기반의 최적화 진행
