# 인터럽트(Interrupt) 시스템

## 1. 인터럽트의 기본 구조

### 1.1 인터럽트 종류와 처리
```c
// 인터럽트 벡터 테이블 구조
struct InterruptVector {
    void (*handler)();  // 인터럽트 처리 함수 포인터
    uint32_t flags;     // 인터럽트 플래그
};

// 인터럽트 처리기 등록
void register_interrupt_handler(int vector, void (*handler)()) {
    // 현재 인터럽트 비활성화
    disable_interrupts();
    
    // 인터럽트 벡터에 핸들러 등록
    interrupt_vector_table[vector].handler = handler;
    interrupt_vector_table[vector].flags = INTERRUPT_ENABLED;
    
    // 인터럽트 재활성화
    enable_interrupts();
}
```

### 1.2 하드웨어 인터럽트 vs 소프트웨어 인터럽트
```java
public class InterruptSystem {

    // 1. 하드웨어 인터럽트 처리
    public class HardwareInterruptHandler {
        
        @InterruptHandler(type = InterruptType.HARDWARE)
        public void handleKeyboardInterrupt() {
            // 1. 현재 컨텍스트 저장
            saveContext();
            
            // 2. 키보드 버퍼 읽기
            char keyCode = readKeyboardBuffer();
            
            // 3. 키보드 이벤트 처리
            processKeyEvent(keyCode);
            
            // 4. 컨텍스트 복원
            restoreContext();
        }
        
        @InterruptHandler(type = InterruptType.HARDWARE)
        public void handleDiskInterrupt() {
            // DMA 완료 확인
            if (isDMAComplete()) {
                // 데이터 전송 완료 처리
                processDMACompletion();
            }
        }
    }

    // 2. 소프트웨어 인터럽트 처리
    public class SoftwareInterruptHandler {
        
        @InterruptHandler(type = InterruptType.SOFTWARE)
        public void handleSystemCall(int syscallNumber) {
            switch (syscallNumber) {
                case SYS_READ:
                    handleReadSystemCall();
                    break;
                case SYS_WRITE:
                    handleWriteSystemCall();
                    break;
                // 기타 시스템 콜 처리
            }
        }
    }
}
```

## 2. 인터럽트 우선순위와 중첩

```java
public class InterruptPriorityManager {
    
    // 1. 인터럽트 우선순위 관리
    private final PriorityQueue<Interrupt> interruptQueue = 
        new PriorityQueue<>((i1, i2) -> 
            Integer.compare(i2.getPriority(), i1.getPriority()));
            
    public void handleInterrupt(Interrupt interrupt) {
        // 현재 처리 중인 인터럽트보다 우선순위가 높은 경우
        if (isHigherPriority(interrupt)) {
            // 현재 처리 중인 인터럽트 중단
            suspendCurrentInterrupt();
            
            // 새로운 인터럽트 처리
            processInterrupt(interrupt);
            
            // 이전 인터럽트 재개
            resumePreviousInterrupt();
        } else {
            // 큐에 추가
            interruptQueue.offer(interrupt);
        }
    }

    // 2. 인터럽트 중첩 처리
    private class InterruptNestingManager {
        private final Stack<InterruptContext> contextStack = 
            new Stack<>();
            
        public void handleNestedInterrupt(Interrupt interrupt) {
            // 현재 컨텍스트 저장
            contextStack.push(saveCurrentContext());
            
            try {
                // 새로운 인터럽트 처리
                processInterrupt(interrupt);
            } finally {
                // 이전 컨텍스트 복원
                restoreContext(contextStack.pop());
            }
        }
    }
}
```

## 3. 인터럽트 지연과 비활성화

```java
public class InterruptController {
    
    // 1. 인터럽트 지연 메커니즘
    public class InterruptDeferral {
        private final Queue<DeferredInterrupt> deferredQueue = 
            new LinkedList<>();
            
        public void deferInterrupt(Interrupt interrupt) {
            // 중요한 작업 중인지 확인
            if (isCriticalSection()) {
                // 인터럽트 지연
                deferredQueue.offer(
                    new DeferredInterrupt(interrupt));
            } else {
                // 즉시 처리
                processInterrupt(interrupt);
            }
        }

        // 지연된 인터럽트 처리
        public void processDeferredInterrupts() {
            while (!deferredQueue.isEmpty()) {
                DeferredInterrupt deferred = deferredQueue.poll();
                if (deferred.isStillValid()) {
                    processInterrupt(deferred.getInterrupt());
                }
            }
        }
    }

    // 2. 인터럽트 마스킹
    public class InterruptMasking {
        private int interruptMask = 0;
        
        public void maskInterrupt(int interruptNumber) {
            interruptMask |= (1 << interruptNumber);
        }
        
        public void unmaskInterrupt(int interruptNumber) {
            interruptMask &= ~(1 << interruptNumber);
        }
        
        public boolean isInterruptMasked(int interruptNumber) {
            return (interruptMask & (1 << interruptNumber)) != 0;
        }
    }
}
```

## 4. 인터럽트 처리 성능 최적화

```java
public class InterruptOptimizer {
    
    // 1. 인터럽트 부하 분산
    public class InterruptLoadBalancer {
        private final int processorCount;
        private final AtomicInteger currentProcessor = 
            new AtomicInteger(0);
            
        public int getTargetProcessor(Interrupt interrupt) {
            // 라운드 로빈 방식으로 프로세서 선택
            return currentProcessor.getAndIncrement() % 
                processorCount;
        }
        
        // 인터럽트 친화도(Affinity) 설정
        public void setInterruptAffinity(
            Interrupt interrupt, 
            int processor) {
            
            // 특정 인터럽트를 특정 프로세서에 바인딩
            bindInterruptToProcessor(interrupt, processor);
        }
    }

    // 2. 인터럽트 통합
    public class InterruptCoalescing {
        private final Map<Integer, List<Interrupt>> 
            coalescingBuffers = new HashMap<>();
            
        public void handleInterrupt(Interrupt interrupt) {
            int type = interrupt.getType();
            
            coalescingBuffers.computeIfAbsent(
                type, k -> new ArrayList<>())
                .add(interrupt);
                
            // 버퍼가 충분히 찼거나 시간이 지났을 때 한 번에 처리
            if (shouldProcessBuffer(type)) {
                processInterruptBuffer(type);
            }
        }
    }
}
```

이러한 인터럽트 시스템을 통해:

1. 기본 구조
    - 하드웨어/소프트웨어 인터럽트
    - 인터럽트 벡터
    - 인터럽트 핸들러

2. 우선순위 관리
    - 우선순위 기반 처리
    - 인터럽트 중첩
    - 컨텍스트 전환

3. 지연과 마스킹
    - 인터럽트 지연
    - 인터럽트 마스킹
    - 크리티컬 섹션 보호

4. 성능 최적화
    - 부하 분산
    - 인터럽트 친화도
    - 인터럽트 통합

을 구현할 수 있습니다.

면접에서 중요한 포인트:
1. 인터럽트의 필요성
2. 하드웨어와 소프트웨어 인터럽트의 차이
3. 우선순위 처리 방식
4. 실시간 시스템에서의 인터럽트 처리
5. 성능과 지연시간 트레이드오프