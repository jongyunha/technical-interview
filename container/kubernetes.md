# Kubernetes 아키텍처에 대한 기술 면접 답변

## 주요 질문
"Kubernetes의 주요 컴포넌트들과 그들의 역할에 대해 설명해주세요. Control Plane과 Worker Node의 차이점은 무엇인가요?"

## 1. Kubernetes 아키텍처 개요

### 1.1 Control Plane 컴포넌트
```yaml
# Control Plane 구성요소
1. kube-apiserver:
   - API 서버
   - 모든 작업의 중심점

2. etcd:
   - 분산 키-값 저장소
   - 클러스터 데이터 저장

3. kube-scheduler:
   - 파드 스케줄링
   - 노드 할당 결정

4. kube-controller-manager:
   - 컨트롤러 실행
   - 상태 관리

5. cloud-controller-manager:
   - 클라우드 서비스 연동
```

### 1.2 Worker Node 컴포넌트
```yaml
# Worker Node 구성요소
1. kubelet:
   - 컨테이너 실행 관리
   - 노드 에이전트

2. kube-proxy:
   - 네트워크 프록시
   - 서비스 구현

3. Container Runtime:
   - 컨테이너 실행 환경
   - Docker, containerd 등
```

## 2. 통신 흐름

### 2.1 API 요청 흐름
```plaintext
[사용자/도구] → [kube-apiserver] → [인증/인가] → [admission controller] → [etcd]
```

### 2.2 파드 생성 흐름
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    
# 실행 흐름:
1. kubectl apply → API Server
2. API Server → etcd (저장)
3. Scheduler → 노드 선택
4. Kubelet → 컨테이너 실행
```

## 3. 고가용성 구성

### 3.1 Control Plane HA
```yaml
# Control Plane HA 구성
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
metadata:
  name: ha-cluster
kubernetesVersion: v1.24.0
controlPlaneEndpoint: "172.16.1.100:6443"
etcd:
  external:
    endpoints:
      - https://10.0.1.10:2379
      - https://10.0.1.11:2379
      - https://10.0.1.12:2379
```

### 3.2 etcd 클러스터 구성
```bash
# etcd 클러스터 상태 확인
etcdctl member list \
  --endpoints=https://10.0.1.10:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## 4. 네트워킹

### 4.1 클러스터 네트워킹
```yaml
# Calico CNI 설정 예시
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  ipipMode: Always
  natOutgoing: true
```

### 4.2 서비스 네트워킹
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

## 근거 자료

### 1. 공식 문서
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/nodes/)

### 2. 모범 사례
- [Kubernetes Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [Production-Grade Container Orchestration](https://kubernetes.io/docs/setup/production-environment/)

### 3. 커뮤니티 자료
- [CNCF Documentation](https://www.cncf.io/certification/cka/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

## 실무 관련 추가 질문

1. "Kubernetes 클러스터 장애 시 어떻게 대응하시나요?"

2. "Control Plane 컴포넌트 중 하나가 실패할 경우 어떻게 되나요?"

3. "etcd 백업과 복구 전략은 어떻게 되나요?"

4. "클러스터 업그레이드는 어떻게 진행하시나요?"

## 실제 사용 예시

### 클러스터 모니터링
```bash
# 클러스터 상태 확인
kubectl get nodes
kubectl get componentstatuses
kubectl get pods -n kube-system

# 로그 확인
kubectl logs -n kube-system kube-apiserver-master
```

### 백업 및 복구
```bash
# etcd 백업
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# etcd 복구
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --data-dir=/var/lib/etcd-backup
```

### 클러스터 업그레이드
```bash
# 컨트롤 플레인 업그레이드
kubeadm upgrade plan
kubeadm upgrade apply v1.24.0

# 노드 업그레이드
kubectl drain node-1
apt-get upgrade -y kubeadm=1.24.0-00
kubeadm upgrade node
apt-get upgrade -y kubelet=1.24.0-00
systemctl restart kubelet
kubectl uncordon node-1
```

실제 면접에서는 이론적인 지식과 함께 클러스터 운영 경험, 장애 대응 경험, 업그레이드 경험 등을 구체적으로 설명하는 것이 좋습니다.