# Docker 네트워킹에 대한 기술 면접 답변

## 주요 질문
"Docker의 네트워크 드라이버 종류와 각각의 사용 사례에 대해 설명해주세요. 컨테이너 간 통신은 어떻게 이루어지나요?"

## 1. Docker 네트워크 드라이버

### 1.1 기본 네트워크 드라이버
```bash
# 네트워크 목록 확인
docker network ls

# 네트워크 생성
docker network create \
  --driver bridge \
  --subnet=172.18.0.0/16 \
  --gateway=172.18.0.1 \
  my-network

# 컨테이너를 네트워크에 연결
docker run -d --network my-network --name container1 nginx
```

### 1.2 네트워크 드라이버 종류
1. Bridge Network (기본)
```bash
# 사용자 정의 브리지 네트워크 생성
docker network create --driver bridge isolated_network

# 컨테이너 실행 시 네트워크 지정
docker run -d --name app1 --network isolated_network myapp
docker run -d --name app2 --network isolated_network myapp
```

2. Host Network
```bash
# 호스트 네트워크 사용
docker run -d --network host nginx
```

3. Overlay Network (Swarm mode)
```bash
# 오버레이 네트워크 생성
docker network create \
  --driver overlay \
  --attachable \
  --subnet=10.0.0.0/16 \
  my-overlay-network
```

4. Macvlan Network
```bash
# Macvlan 네트워크 생성
docker network create \
  --driver macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net
```

## 2. 컨테이너 간 통신

### 2.1 같은 네트워크 내 통신
```yaml
# docker-compose.yml
version: '3'
services:
  web:
    image: nginx
    networks:
      - frontend
  
  app:
    image: myapp
    networks:
      - frontend
      - backend
  
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
```

### 2.2 네트워크 연결과 분리
```bash
# 실행 중인 컨테이너에 네트워크 연결
docker network connect my-network container1

# 네트워크 연결 해제
docker network disconnect my-network container1
```

## 3. 네트워크 설정과 관리

### 3.1 포트 매핑
```bash
# 단일 포트 매핑
docker run -d -p 8080:80 nginx

# 다중 포트 매핑
docker run -d \
  -p 80:80 \
  -p 443:443 \
  nginx

# 특정 IP에 대한 포트 매핑
docker run -d -p 127.0.0.1:8080:80 nginx
```

### 3.2 네트워크 검사와 디버깅
```bash
# 네트워크 상세 정보 확인
docker network inspect my-network

# 컨테이너 네트워크 설정 확인
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container1

# 네트워크 문제 해결
docker run --rm \
  --network container:container1 \
  nicolaka/netshoot \
  ping container2
```

## 4. 고급 네트워킹 기능

### 4.1 DNS와 서비스 디스커버리
```yaml
# 사용자 정의 DNS 설정
version: '3'
services:
  web:
    image: nginx
    networks:
      frontend:
        aliases:
          - web.local
    dns:
      - 8.8.8.8
      - 8.8.4.4

networks:
  frontend:
```

### 4.2 로드 밸런싱
```yaml
# HAProxy 설정 예시
version: '3'
services:
  lb:
    image: haproxy
    ports:
      - "80:80"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks:
      - frontend

  web:
    image: nginx
    deploy:
      replicas: 3
    networks:
      - frontend

networks:
  frontend:
```

## 근거 자료

### 1. 공식 문서
- [Docker Networking Overview](https://docs.docker.com/network/)
- [Docker Network Drivers](https://docs.docker.com/network/drivers/)
- [Docker Compose Networking](https://docs.docker.com/compose/networking/)

### 2. 모범 사례
- [Docker Networking Best Practices](https://docs.docker.com/network/network-tutorial-standalone/)
- [Container Network Security](https://docs.docker.com/network/security/)

### 3. 기술 블로그
- [Docker Blog - Networking](https://www.docker.com/blog/tag/networking/)
- [Network Troubleshooting Guide](https://success.docker.com/article/troubleshooting-container-networking)

## 실무 관련 추가 질문

1. "컨테이너 간 네트워크 격리를 어떻게 구현하시나요?"

2. "Docker 네트워크 성능을 모니터링하고 최적화하는 방법은?"

3. "멀티 호스트 환경에서 컨테이너 네트워킹은 어떻게 구성하시나요?"

4. "네트워크 보안을 위해 어떤 조치들을 취하시나요?"

## 실제 사용 예시

### 네트워크 격리 구현
```yaml
# 네트워크 격리 예시
version: '3'
services:
  web:
    image: nginx
    networks:
      frontend:
  
  api:
    image: myapi
    networks:
      frontend:
      backend:
  
  db:
    image: postgres
    networks:
      backend:
        aliases:
          - database
    environment:
      POSTGRES_PASSWORD: secret

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # 외부 접근 차단
```

### 네트워크 모니터링
```bash
# 네트워크 통계 확인
docker stats

# 특정 컨테이너의 네트워크 사용량
docker stats container1 --format "{{.Name}}: {{.NetIO}}"

# tcpdump를 사용한 패킷 캡처
docker run --rm --net container:container1 \
  nicolaka/netshoot \
  tcpdump -i any port 80
```

실제 면접에서는 이론적인 지식과 함께 Docker 네트워킹 관련 실제 문제 해결 경험, 네트워크 구성 설계 경험 등을 구체적으로 설명하는 것이 좋습니다.