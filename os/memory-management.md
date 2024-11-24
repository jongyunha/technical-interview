# 메모리 관리(Memory Management)에 대한 기술 면접 답변

## 주요 질문
"운영체제의 메모리 관리 기법에 대해 설명해주세요. 가상 메모리와 페이징 시스템은 어떻게 작동하나요?"

## 1. 메모리 관리 기본 개념

### 1.1 물리적 메모리와 가상 메모리
```
[가상 메모리 주소 공간]     [페이지 테이블]      [물리적 메모리]
+------------------+      +------------+     +----------------+
|    Code (0x000)  | ---> | Frame 5    | --> |    Frame 0     |
+------------------+      +------------+     +----------------+
|    Data (0x400)  | ---> | Frame 2    | --> |    Frame 1     |
+------------------+      +------------+     +----------------+
|    Heap (0x800)  | ---> | Frame 7    | --> |    Frame 2     |
+------------------+      +------------+     +----------------+
|    Stack (0xFFF) | ---> | Frame 1    | --> |    Frame 3     |
+------------------+      +------------+     +----------------+
```

### 1.2 주소 변환 과정
```c
// 가상 주소를 물리 주소로 변환하는 의사 코드
struct VirtualAddress {
    unsigned int pageNumber;
    unsigned int offset;
};

unsigned int translateAddress(VirtualAddress va) {
    unsigned int frameNumber = pageTable[va.pageNumber];
    return (frameNumber << OFFSET_BITS) | va.offset;
}
```

## 2. 메모리 관리 기법

### 2.1 페이징(Paging)
```java
public class PageTable {
    private static final int PAGE_SIZE = 4096; // 4KB
    private static final int PAGE_BITS = 12;   // 2^12 = 4096
    
    public long getPhysicalAddress(long virtualAddress) {
        long pageNumber = virtualAddress >>> PAGE_BITS;
        long offset = virtualAddress & (PAGE_SIZE - 1);
        long frameNumber = pageTable[pageNumber];
        return (frameNumber << PAGE_BITS) | offset;
    }
}
```

### 2.2 세그멘테이션(Segmentation)
```c
struct Segment {
    unsigned int base;    // 시작 주소
    unsigned int limit;   // 세그먼트 크기
    unsigned int flags;   // 접근 권한 등
};

// 세그먼트 테이블
Segment segmentTable[MAX_SEGMENTS];

bool checkAccess(int segmentNumber, unsigned int offset) {
    if (offset > segmentTable[segmentNumber].limit) {
        return false;  // 세그먼트 범위 초과
    }
    return true;
}
```

## 3. 페이지 교체 알고리즘

### 3.1 LRU (Least Recently Used)
```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

### 3.2 Clock Algorithm
```java
public class ClockPageReplacement {
    private boolean[] referenceBits;
    private int clockHand = 0;
    
    public int findVictimPage() {
        while (true) {
            if (referenceBits[clockHand]) {
                referenceBits[clockHand] = false;  // second chance
            } else {
                int victimPage = clockHand;
                clockHand = (clockHand + 1) % referenceBits.length;
                return victimPage;
            }
            clockHand = (clockHand + 1) % referenceBits.length;
        }
    }
}
```

## 4. 메모리 할당 기법

### 4.1 연속 할당
```java
public class ContinuousMemoryAllocation {
    private class MemoryBlock {
        int start;
        int size;
        boolean allocated;
    }
    
    // First Fit 알고리즘
    public MemoryBlock firstFit(int size) {
        for (MemoryBlock block : memoryBlocks) {
            if (!block.allocated && block.size >= size) {
                return allocateMemory(block, size);
            }
        }
        return null;
    }
}
```

### 4.2 페이지 프레임 할당
```java
public class PageFrameAllocation {
    private final int totalFrames;
    private final Map<Integer, Set<Integer>> processToFrames;
    
    public boolean allocateFrames(int processId, int frameCount) {
        if (getAvailableFrames() < frameCount) {
            return false;  // 메모리 부족
        }
        // 프레임 할당 로직
        return true;
    }
}
```

## 5. 메모리 보호

### 5.1 경계 레지스터 사용
```java
public class MemoryProtection {
    private final int baseRegister;
    private final int limitRegister;
    
    public boolean checkAccess(int address) {
        if (address < baseRegister || 
            address > baseRegister + limitRegister) {
            throw new SegmentationFault();
        }
        return true;
    }
}
```

## 근거 자료

### 1. 운영체제 교과서
- [Operating System Concepts](https://www.os-book.com/OS10/)
    - Chapter 8: Memory Management
    - Chapter 9: Virtual Memory
- [Modern Operating Systems](https://www.pearson.com/en-us/subject-catalog/p/modern-operating-systems/P200000003295)
    - Chapter 3: Memory Management

### 2. 기술 문서
- [Linux Memory Management Documentation](https://www.kernel.org/doc/html/latest/admin-guide/mm/)
- [Windows Memory Management](https://docs.microsoft.com/en-us/windows/win32/memory/memory-management)

### 3. 학술 논문
- ["The Working Set Model for Program Behavior"](https://dl.acm.org/doi/10.1145/363095.363141)
    - Peter J. Denning
- ["A Survey of Page Replacement Algorithms"](https://ieeexplore.ieee.org/document/987595)
    - IEEE Transactions

### 4. 실무 자료
- [Java Memory Management Whitepaper](https://www.oracle.com/technical-resources/articles/javase/gc-tuning-6.html)
- [Linux Memory Management Guide](https://www.kernel.org/doc/gorman/html/understand/)

## 실무 관련 추가 질문

1. "JVM의 메모리 구조와 가비지 컬렉션에 대해 설명해주세요."

2. "메모리 누수(Memory Leak)를 탐지하고 해결한 경험이 있나요?"

3. "큰 데이터를 처리할 때 메모리 관리는 어떻게 하시나요?"

4. "페이지 폴트(Page Fault)가 성능에 미치는 영향과 최적화 방법은?"

## 실제 사용 예시

### Out of Memory 문제 해결
```java
public class MemoryLeakExample {
    private static final List<byte[]> leakedList = new ArrayList<>();
    
    // 메모리 누수 발생
    public void causeMemoryLeak() {
        while (true) {
            leakedList.add(new byte[1024 * 1024]); // 1MB
        }
    }
    
    // 해결 방법
    public void fixedImplementation() {
        try (BufferedReader reader = new BufferedReader(new FileReader("large.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) {
                processLine(line);
            }
        }
    }
}
```

### 메모리 모니터링
```java
public class MemoryMonitor {
    public static void printMemoryStats() {
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("Used Memory: " + 
            (usedMemory / 1024 / 1024) + " MB");
        System.out.println("Free Memory: " + 
            (freeMemory / 1024 / 1024) + " MB");
        System.out.println("Total Memory: " + 
            (totalMemory / 1024 / 1024) + " MB");
    }
}
```

실제 면접에서는 이론적인 지식과 함께 실제 프로젝트에서 겪은 메모리 관련 문제와 해결 경험을 구체적으로 설명하는 것이 좋습니다.