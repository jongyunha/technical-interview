# Technical Interview Guide for Backend Developers

이 저장소는 백엔드 개발자를 위한 기술 면접 가이드를 제공합니다. 실제 기업의 면접 질문과 모범 답안, 그리고 각각의 답변에 대한 참고 자료들을 포함하고 있습니다.

## 목차

### 1. Database
- [인덱스(Index)의 동작 원리와 장단점](database/index.md) 
- [트랜잭션과 ACID](database/transaction-acid.md)
- [트랜잭션 격리 수준(Isolation Level)](database/transaction-isolation.md)
- [정규화와 반정규화](database/normalization.md)
- [SQL 튜닝](database/sql-tuning.md)

### 2. Operating System
- [프로세스와 스레드](os/process-thread.md)
- [동기와 비동기](os/sync-async.md)
- [교착상태(Deadlock)](os/deadlock.md)
- [메모리 관리](os/memory-management.md)

### 3. Network
- [TCP/IP 프로토콜](network/tcp-ip.md)
- [HTTP와 HTTPS](network/http-https.md)
- [REST API](network/rest-api.md)
- [로드 밸런싱](network/load-balancing.md)

### 4. Container & Orchestration
- [Docker 기본개념](container/docker.md)
- [Docker 네트워킹](container/docker-network.md)
- [Docker 볼륨과 스토리지](container/docker-volume.md)
- [Kubernetes 아키텍쳐](container/kubernetes.md)
- [Kubernetes 워크로드](container/kubernetes-workload.md)
- [Kubernetes 서비스와 네트워킹](container/kubernetes-service.md)
- [Kubernetes 스토리지](container/kubernetes-storage.md)
- [Kubernetes 보안](container/kubernetes-security.md)

### 4. Java & Spring
- [JVM과 가비지 컬렉션](java/jvm-gc.md)
- [스프링 IoC/DI](spring/ioc-di.md)
- [스프링 AOP](spring/aop.md)
- [스프링 트랜잭션](spring/transaction.md)

### 5. 시스템 디자인
- [대용량 트래픽 처리](system-design/traffic.md)
- [캐싱 전략](system-design/caching.md)
- [데이터 샤딩](system-design/sharding.md)
- [마이크로서비스 아키텍처](system-design/msa.md)

## 기여 방법

이 프로젝트는 개발자 커뮤니티의 기여를 환영합니다. 다음과 같은 방법으로 기여하실 수 있습니다:

1. Issue 등록
    - 오타 또는 잘못된 정보 제보
    - 새로운 주제 제안
    - 질문 또는 토론 주제 제시

2. Pull Request
    - 새로운 기술 면접 질문과 답변 추가
    - 기존 내용의 개선 또는 수정
    - 새로운 카테고리 추가

## 참고 자료 형식

각 문서는 다음 형식을 따릅니다:
```markdown
# 주제명

## 질문
[실제 면접 질문]

## 답변
[모범 답안]

## 심화 질문
[예상되는 추가 질문들]

## 참고 자료
[공식 문서, 블로그, 논문 등 참고 자료 링크]
```

## 기여자

<a href="https://github.com/jongyunha/technical-interview/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=jongyunha/technical-interview" />
</a>

## 스타 히스토리

[![Star History Chart](https://api.star-history.com/svg?repos=[jongyunha]/technical-interview&type=Date)](https://star-history.com/#[username]/technical-interview&Date)

---