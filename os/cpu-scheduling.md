# CPU 스케줄링

## 1. 주요 CPU 스케줄링 알고리즘

### 1.1 비선점 스케줄링
```c
// FCFS (First-Come, First-Served)
struct Process {
    int pid;
    int arrival_time;
    int burst_time;
    int waiting_time;
    int turnaround_time;
};

void fcfs_scheduling(Process processes[], int n) {
    int current_time = 0;
    
    for (int i = 0; i < n; i++) {
        // 프로세스가 도착하지 않았다면 대기
        if (current_time < processes[i].arrival_time) {
            current_time = processes[i].arrival_time;
        }
        
        processes[i].waiting_time = 
            current_time - processes[i].arrival_time;
            
        current_time += processes[i].burst_time;
        
        processes[i].turnaround_time = 
            current_time - processes[i].arrival_time;
    }
}

// SJF (Shortest Job First)
void sjf_scheduling(Process processes[], int n) {
    sort(processes, n); // burst_time 기준으로 정렬
    
    fcfs_scheduling(processes, n);
}
```

### 1.2 선점 스케줄링
```java
public class RoundRobinScheduler {
    private final Queue<Process> readyQueue = new LinkedList<>();
    private final int timeQuantum;
    
    public void schedule() {
        while (!readyQueue.isEmpty()) {
            Process currentProcess = readyQueue.poll();
            
            // 타임 퀀텀만큼 실행
            int executionTime = Math.min(
                timeQuantum, 
                currentProcess.getRemainingTime()
            );
            
            executeProcess(currentProcess, executionTime);
            
            // 프로세스가 완료되지 않았다면 다시 큐에 삽입
            if (currentProcess.getRemainingTime() > 0) {
                readyQueue.offer(currentProcess);
            }
        }
    }
}

// Priority Scheduling
public class PriorityScheduler {
    private final PriorityQueue<Process> readyQueue;
    
    public PriorityScheduler() {
        this.readyQueue = new PriorityQueue<>((p1, p2) -> 
            p2.getPriority() - p1.getPriority());
    }
    
    public void addProcess(Process process) {
        readyQueue.offer(process);
        // 우선순위 역전 현상 체크
        checkPriorityInversion();
    }
    
    private void checkPriorityInversion() {
        // 우선순위 상속 구현
        Process highPriorityProcess = readyQueue.peek();
        if (highPriorityProcess != null && 
            highPriorityProcess.isBlocked()) {
            Process blockingProcess = 
                highPriorityProcess.getBlockingProcess();
            if (blockingProcess != null) {
                blockingProcess.setTemporaryPriority(
                    highPriorityProcess.getPriority()
                );
            }
        }
    }
}
```

## 2. 스케줄링 성능 메트릭

```java
public class SchedulingMetrics {
    
    // 1. 평균 대기 시간 계산
    public double calculateAverageWaitingTime(List<Process> processes) {
        return processes.stream()
            .mapToDouble(Process::getWaitingTime)
            .average()
            .orElse(0.0);
    }
    
    // 2. 평균 반환 시간 계산
    public double calculateAverageTurnaroundTime(
        List<Process> processes) {
        
        return processes.stream()
            .mapToDouble(p -> 
                p.getCompletionTime() - p.getArrivalTime())
            .average()
            .orElse(0.0);
    }
    
    // 3. CPU 사용률 계산
    public double calculateCpuUtilization(
        List<Process> processes, 
        double totalTime) {
        
        double busyTime = processes.stream()
            .mapToDouble(Process::getBurstTime)
            .sum();
            
        return (busyTime / totalTime) * 100;
    }
    
    // 4. 처리량 계산
    public double calculateThroughput(
        List<Process> processes, 
        double totalTime) {
        
        return processes.size() / totalTime;
    }
}
```

## 3. 고급 스케줄링 기법

```java
public class AdvancedScheduler {
    
    // 1. 다단계 큐 스케줄링
    public class MultilevelQueueScheduler {
        private final List<Queue<Process>> queues;
        private final List<Integer> timeQuantums;
        
        public void schedule() {
            for (int i = 0; i < queues.size(); i++) {
                Queue<Process> currentQueue = queues.get(i);
                int timeQuantum = timeQuantums.get(i);
                
                // 상위 큐가 비어있을 때만 하위 큐 처리
                if (!currentQueue.isEmpty()) {
                    Process process = currentQueue.poll();
                    executeProcess(process, timeQuantum);
                    
                    // 프로세스가 완료되지 않았다면
                    if (process.getRemainingTime() > 0) {
                        // 다음 하위 큐로 이동
                        if (i < queues.size() - 1) {
                            queues.get(i + 1).offer(process);
                        } else {
                            // 최하위 큐면 같은 큐에 다시 삽입
                            currentQueue.offer(process);
                        }
                    }
                }
            }
        }
    }
    
    // 2. 실시간 스케줄링
    public class RealTimeScheduler {
        private final PriorityQueue<Process> readyQueue;
        
        public RealTimeScheduler() {
            this.readyQueue = new PriorityQueue<>((p1, p2) -> 
                p1.getDeadline().compareTo(p2.getDeadline()));
        }
        
        public void scheduleEDF() { // Earliest Deadline First
            while (!readyQueue.isEmpty()) {
                Process process = readyQueue.poll();
                
                if (canMeetDeadline(process)) {
                    executeProcess(process);
                } else {
                    handleDeadlineMiss(process);
                }
            }
        }
        
        private boolean canMeetDeadline(Process process) {
            return System.currentTimeMillis() + 
                   process.getExecutionTime() <= 
                   process.getDeadline().getTime();
        }
    }
    
    // 3. 공정 스케줄링
    public class FairScheduler {
        private final Map<String, Double> shares;
        private final Map<String, Double> usage;
        
        public Process selectNextProcess(List<Process> runnable) {
            String selectedGroup = runnable.stream()
                .map(Process::getGroupId)
                .min((g1, g2) -> 
                    Double.compare(
                        usage.get(g1) / shares.get(g1),
                        usage.get(g2) / shares.get(g2)
                    ))
                .orElse(null);
                
            return runnable.stream()
                .filter(p -> p.getGroupId().equals(selectedGroup))
                .findFirst()
                .orElse(null);
        }
    }
}
```

이러한 CPU 스케줄링 메커니즘을 통해:

1. 스케줄링 알고리즘
    - 비선점 스케줄링 (FCFS, SJF)
    - 선점 스케줄링 (Round Robin, Priority)
    - 혼합 스케줄링 (다단계 큐)

2. 성능 평가
    - 평균 대기 시간
    - CPU 활용도
    - 처리량
    - 반환 시간

3. 고급 기법
    - 실시간 스케줄링
    - 공정 스케줄링
    - 우선순위 상속

을 구현하여 효율적인 프로세스 관리를 할 수 있습니다.

면접에서 중요한 포인트:
1. 각 알고리즘의 장단점
2. 적합한 사용 사례
3. 성능 트레이드오프
4. 실제 시스템에서의 구현 방식