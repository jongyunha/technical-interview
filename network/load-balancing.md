# 로드 밸런싱에 대한 기술 면접 답변

## 주요 질문
"로드 밸런싱이 무엇이며, 주요 알고리즘과 구현 방식에 대해 설명해주세요. 실제 서비스에서 어떻게 활용되나요?"

## 1. 로드 밸런싱 기본 개념

### 1.1 정의와 목적
```plaintext
[클라이언트] 
       ↓
[로드 밸런서]
    ↙  ↓  ↘
[서버1] [서버2] [서버3]
```
- 트래픽 분산
- 고가용성 확보
- 확장성 제공

### 1.2 로드 밸런서 유형
```java
public enum LoadBalancerType {
    L4, // Transport Layer (TCP/UDP)
    L7  // Application Layer (HTTP/HTTPS)
}
```

## 2. 로드 밸런싱 알고리즘

### 2.1 Round Robin
```java
public class RoundRobinLoadBalancer {
    private final List<Server> servers;
    private AtomicInteger currentIndex = new AtomicInteger(0);
    
    public Server getNextServer() {
        int index = currentIndex.getAndIncrement() % servers.size();
        return servers.get(index);
    }
}
```

### 2.2 Weighted Round Robin
```java
public class WeightedRoundRobinLoadBalancer {
    private final List<Server> servers;
    private final Map<Server, Integer> weights;
    
    public Server getNextServer() {
        int totalWeight = weights.values().stream().mapToInt(Integer::intValue).sum();
        int position = (currentPosition.getAndIncrement() % totalWeight);
        
        int sum = 0;
        for (Server server : servers) {
            sum += weights.get(server);
            if (sum > position) {
                return server;
            }
        }
        return servers.get(0);
    }
}
```

### 2.3 Least Connections
```java
public class LeastConnectionsLoadBalancer {
    private final Map<Server, AtomicInteger> connectionCounts;
    
    public Server getNextServer() {
        return connectionCounts.entrySet().stream()
            .min(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElseThrow();
    }
    
    public void incrementConnections(Server server) {
        connectionCounts.get(server).incrementAndGet();
    }
    
    public void decrementConnections(Server server) {
        connectionCounts.get(server).decrementAndGet();
    }
}
```

## 3. 헬스 체크와 장애 감지

### 3.1 헬스 체크 구현
```java
@Service
public class HealthChecker {
    private final Map<Server, Boolean> serverStatus = new ConcurrentHashMap<>();
    
    @Scheduled(fixedRate = 5000)
    public void checkHealth() {
        servers.forEach(server -> {
            try {
                HttpResponse response = HttpClient.newHttpClient()
                    .send(HttpRequest.newBuilder()
                        .uri(URI.create(server.getHealthCheckUrl()))
                        .build(),
                        HttpResponse.BodyHandlers.ofString());
                
                serverStatus.put(server, response.statusCode() == 200);
            } catch (Exception e) {
                serverStatus.put(server, false);
                log.error("Health check failed for server: " + server, e);
            }
        });
    }
}
```

### 3.2 장애 복구
```java
public class FailoverHandler {
    private final LoadBalancer loadBalancer;
    private final HealthChecker healthChecker;
    
    public Server getHealthyServer() {
        Server server = loadBalancer.getNextServer();
        int attempts = 0;
        
        while (!healthChecker.isHealthy(server) && attempts < 3) {
            server = loadBalancer.getNextServer();
            attempts++;
        }
        
        if (attempts == 3) {
            throw new NoHealthyServerAvailableException();
        }
        
        return server;
    }
}
```

## 4. 세션 관리

### 4.1 Sticky Session
```java
public class StickySessionLoadBalancer {
    private final Map<String, Server> sessionAffinity = new ConcurrentHashMap<>();
    
    public Server getServer(String sessionId) {
        return sessionAffinity.computeIfAbsent(sessionId, k -> {
            Server server = loadBalancer.getNextServer();
            log.info("New session {} assigned to server {}", sessionId, server);
            return server;
        });
    }
}
```

### 4.2 Session Clustering
```java
@Configuration
public class SessionConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        return template;
    }
    
    @Bean
    public RedisHttpSessionConfiguration sessionConfig() {
        RedisHttpSessionConfiguration config = new RedisHttpSessionConfiguration();
        config.setMaxInactiveIntervalInSeconds(3600);
        return config;
    }
}
```

## 5. 실제 구현 예시

### 5.1 Spring Cloud Load Balancer
```java
@Configuration
@LoadBalancerClient(name = "user-service")
public class LoadBalancerConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> reactorLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty("spring.application.name");
        return new RoundRobinLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
        );
    }
}
```

### 5.2 nginx 설정 예시
```nginx
upstream backend {
    least_conn;  # 최소 연결 방식
    server backend1.example.com:8080 weight=3;
    server backend2.example.com:8080 weight=2;
    server backend3.example.com:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 근거 자료

### 1. 공식 문서
- [NGINX Documentation](https://nginx.org/en/docs/)
- [HAProxy Documentation](http://www.haproxy.org/#docs)
- [AWS ELB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/)

### 2. 기술 문서
- [Spring Cloud LoadBalancer](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#spring-cloud-loadbalancer)
- [Kubernetes Load Balancing](https://kubernetes.io/docs/concepts/services-networking/service/)

### 3. 아키텍처 관련 자료
- [Microsoft Azure Architecture Guide](https://docs.microsoft.com/en-us/azure/architecture/guide/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)

## 실무 관련 추가 질문

1. "로드 밸런서 장애 시 어떻게 대응하시나요?"

2. "트래픽이 갑자기 증가할 때 어떻게 대응하시나요?"

3. "세션 클러스터링과 Sticky Session 중 어떤 것을 선호하시나요?"

4. "SSL 종료(SSL Termination)는 어디서 처리하시나요?"

## 실제 운영 시 고려사항

### 성능 모니터링
```java
@Component
public class LoadBalancerMonitor {
    private final MeterRegistry registry;
    
    public void recordResponseTime(String server, long responseTime) {
        registry.timer("lb.response.time", "server", server)
            .record(responseTime, TimeUnit.MILLISECONDS);
    }
    
    public void recordActiveConnections(String server, int connections) {
        registry.gauge("lb.active.connections", 
            Collections.singletonList(Tag.of("server", server)), 
            connections);
    }
}
```

### 자동 스케일링 설정
```yaml
# Kubernetes HPA 설정 예시
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

실제 면접에서는 이론적인 지식과 함께 로드 밸런싱 구성 경험, 장애 대응 경험, 성능 최적화 경험 등을 구체적으로 설명하는 것이 좋습니다.