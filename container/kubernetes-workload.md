# Kubernetes 워크로드에 대한 기술 면접 답변

## 주요 질문
"Kubernetes의 주요 워크로드 리소스들(Pod, Deployment, StatefulSet 등)의 특징과 각각의 사용 사례에 대해 설명해주세요."

## 1. Pod

### 1.1 기본 Pod 구성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
```

### 1.2 Multi-Container Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: web
    image: nginx
  - name: log-aggregator
    image: fluentd
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  volumes:
  - name: shared-logs
    emptyDir: {}
```

## 2. Deployment

### 2.1 Deployment 설정
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
```

### 2.2 배포 전략
```yaml
# Blue/Green 배포
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: green
  template:
    metadata:
      labels:
        app: nginx
        version: green
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.0
```

## 3. StatefulSet

### 3.1 StatefulSet 구성
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

## 4. DaemonSet과 Job

### 4.1 DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.7
```

### 4.2 CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool
            command: ["/backup.sh"]
          restartPolicy: OnFailure
```

## 근거 자료

### 1. 공식 문서
- [Kubernetes Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

### 2. 모범 사례
- [Production-Grade Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#use-case)
- [StatefulSet Best Practices](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#best-practices)

## 실무 관련 추가 질문

1. "무상태(Stateless)와 상태유지(Stateful) 애플리케이션의 배포 전략 차이는?"

2. "Pod의 리소스 요청과 제한을 어떻게 결정하시나요?"

3. "롤링 업데이트 중 장애가 발생하면 어떻게 대응하시나요?"

4. "StatefulSet을 사용할 때의 주의사항은 무엇인가요?"

## 실제 사용 예시

### 카나리 배포
```yaml
# 기존 버전 (90%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: app
        image: myapp:1.0

# 새 버전 (10%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: app
        image: myapp:1.1
```

### 리소스 관리
```bash
# 리소스 사용량 모니터링
kubectl top pods
kubectl top nodes

# HPA (Horizontal Pod Autoscaling)
kubectl autoscale deployment nginx-deployment \
  --min=3 --max=10 --cpu-percent=80
```

### 롤백 처리
```bash
# 배포 이력 확인
kubectl rollout history deployment/nginx-deployment

# 특정 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment \
  --to-revision=2

# 배포 상태 확인
kubectl rollout status deployment/nginx-deployment
```

실제 면접에서는 이론적인 지식과 함께 워크로드 운영 경험, 장애 대응 경험, 스케일링 경험 등을 구체적으로 설명하는 것이 좋습니다.