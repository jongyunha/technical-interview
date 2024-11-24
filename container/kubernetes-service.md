# Kubernetes 서비스와 네트워킹에 대한 기술 면접 답변

## 주요 질문
"Kubernetes의 서비스 타입들과 각각의 용도에 대해 설명해주세요. 또한 Pod 간 통신이 어떻게 이루어지는지 설명해주세요."

## 1. 서비스 타입

### 1.1 ClusterIP
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

### 1.2 NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30007  # 30000-32767 범위
```

### 1.3 LoadBalancer
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-lb-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  externalTrafficPolicy: Local
```

### 1.4 ExternalName
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-database
spec:
  type: ExternalName
  externalName: database.example.com
```

## 2. 네트워크 정책

### 2.1 기본 네트워크 정책
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              role: database
      ports:
        - protocol: TCP
          port: 5432
```

### 2.2 CIDR 기반 정책
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-traffic
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
```

## 3. Ingress 설정

### 3.1 기본 Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### 3.2 TLS 설정
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

## 4. 서비스 디스커버리와 DNS

### 4.1 CoreDNS 설정
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

## 근거 자료

### 1. 공식 문서
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

### 2. 모범 사례
- [Service Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

## 실무 관련 추가 질문

1. "서비스 메시(Service Mesh)를 사용해보셨나요? 장단점은 무엇인가요?"

2. "멀티 클러스터 환경에서 서비스 디스커버리는 어떻게 구현하시나요?"

3. "Ingress 컨트롤러 선택 시 고려사항은 무엇인가요?"

4. "네트워크 정책으로 마이크로서비스 간 통신을 어떻게 제어하시나요?"

## 실제 사용 예시

### 서비스 모니터링
```bash
# 서비스 연결성 테스트
kubectl run -i --tty --rm debug \
  --image=busybox --restart=Never \
  -- wget -O- http://my-service:80

# 엔드포인트 확인
kubectl get endpoints my-service
```

### 트래픽 제어
```yaml
# 트래픽 분할 예시
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service-split
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service-v1
        subset: v1
      weight: 90
    - destination:
        host: my-service-v2
        subset: v2
      weight: 10
```

실제 면접에서는 이론적인 지식과 함께 네트워크 구성 경험, 트러블슈팅 경험, 성능 최적화 경험 등을 구체적으로 설명하는 것이 좋습니다.