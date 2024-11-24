# Kubernetes 스토리지에 대한 기술 면접 답변

## 주요 질문
"Kubernetes의 다양한 스토리지 옵션들과 각각의 사용 사례에 대해 설명해주세요. PV, PVC의 라이프사이클은 어떻게 되나요?"

## 1. 영구 볼륨 (PV)과 영구 볼륨 클레임 (PVC)

### 1.1 PV 정의
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-storage
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  nfs:
    path: /path/to/data
    server: nfs-server.example.com
```

### 1.2 PVC 정의
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

## 2. Storage Classes

### 2.1 기본 StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 2.2 동적 프로비저닝
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

## 3. 볼륨 타입

### 3.1 emptyDir
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      medium: Memory
      sizeLimit: 500Mi
```

### 3.2 hostPath
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: web-data
  volumes:
  - name: web-data
    hostPath:
      path: /data
      type: DirectoryOrCreate
```

## 4. 스토리지 확장 및 백업

### 4.1 볼륨 확장
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi  # 크기 증가
  storageClassName: standard
```

### 4.2 백업 설정
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: volume-backup
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-image
            volumeMounts:
            - name: data
              mountPath: /data
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: mysql-pvc
```

## 근거 자료

### 1. 공식 문서
- [Kubernetes Storage Concepts](https://kubernetes.io/docs/concepts/storage/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

### 2. 모범 사례
- [K8s Storage Best Practices](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)

## 실무 관련 추가 질문

1. "클라우드 환경에서 스토리지 선택 시 고려사항은 무엇인가요?"

2. "스테이트풀 애플리케이션의 데이터 백업 전략은 어떻게 되나요?"

3. "볼륨 성능 모니터링은 어떻게 하시나요?"

4. "다중 가용성 존(AZ)에서 스토리지 관리는 어떻게 하시나요?"

## 실제 사용 예시

### 데이터베이스 스토리지 구성
```yaml
# MySQL StatefulSet with PVC
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
      storageClassName: standard
```

### 볼륨 스냅샷
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: mysql-pvc

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  dataSource:
    name: data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### 모니터링 설정
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: storage-monitor
spec:
  selector:
    matchLabels:
      app: storage-metrics
  endpoints:
  - port: metrics
    interval: 15s
```

실제 면접에서는 이론적인 지식과 함께 스토리지 설계 경험, 성능 최적화 경험, 장애 대응 경험 등을 구체적으로 설명하는 것이 좋습니다.