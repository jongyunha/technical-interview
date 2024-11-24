# Kubernetes 보안에 대한 기술 면접 답변

## 주요 질문
"Kubernetes의 주요 보안 구성 요소와 보안 모범 사례에 대해 설명해주세요. RBAC은 어떻게 구성하나요?"

## 1. RBAC (Role-Based Access Control)

### 1.1 Role과 RoleBinding
```yaml
# Role 정의
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 1.2 ClusterRole과 ClusterRoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## 2. Pod 보안 정책

### 2.1 SecurityContext
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
```

### 2.2 PodSecurityPolicy
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
```

## 3. 네트워크 보안

### 3.1 NetworkPolicy
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

## 4. 시크릿 관리

### 4.1 Secret 생성과 사용
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm

---
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
  - name: test-container
    image: nginx
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secret-volume"
  volumes:
  - name: secret-volume
    secret:
      secretName: mysecret
```

## 5. 보안 모니터링과 감사

### 5.1 AuditPolicy
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]

- level: Request
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["create", "delete"]
```

## 근거 자료

### 1. 공식 문서
- [Kubernetes Security Concepts](https://kubernetes.io/docs/concepts/security/)
- [RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

### 2. 보안 모범 사례
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NIST Kubernetes Security Guide](https://csrc.nist.gov/publications/detail/sp/800-190/final)

## 실무 관련 추가 질문

1. "컨테이너 이미지 보안은 어떻게 관리하시나요?"

2. "Kubernetes 클러스터의 보안 감사는 어떻게 수행하시나요?"

3. "시크릿 관리에서 외부 시스템(Vault 등)을 사용해보셨나요?"

4. "제로 트러스트 보안을 K8s에서 어떻게 구현하시나요?"

## 실제 사용 예시

### 이미지 보안 정책
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ImagePolicyWebhook
metadata:
  name: image-policy
webhooks:
- name: image-policy.k8s.io
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
```

### 감사 로깅 설정
```bash
# kube-apiserver 설정
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

실제 면접에서는 이론적인 지식과 함께 보안 구성 경험, 취약점 대응 경험, 보안 감사 경험 등을 구체적으로 설명하는 것이 좋습니다.