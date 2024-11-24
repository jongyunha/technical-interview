# Docker 기본 개념에 대한 기술 면접 답변

## 주요 질문
"Docker의 기본 개념과 컨테이너화의 장점에 대해 설명해주세요. 가상화(VM)와 컨테이너의 차이점은 무엇인가요?"

## 1. Docker 핵심 개념

### 1.1 컨테이너 vs 가상화
```plaintext
[기존 가상화]               [Docker 컨테이너]
+----------------+         +----------------+
| App A | App B  |         | App A | App B  |
+----------------+         +----------------+
| Guest OS  A|B |         |  Container A|B |
+----------------+         +----------------+
| Hypervisor    |         | Docker Engine  |
+----------------+         +----------------+
| Host OS       |         | Host OS        |
+----------------+         +----------------+
| Infrastructure |         | Infrastructure |
+----------------+         +----------------+
```

### 1.2 Docker 기본 구성요소
```bash
# Dockerfile 예시
FROM openjdk:11-jdk
WORKDIR /app
COPY target/myapp.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]

# Docker Image 빌드
docker build -t myapp:1.0 .

# Docker Container 실행
docker run -d -p 8080:8080 myapp:1.0
```

## 2. Docker 주요 명령어와 활용

### 2.1 이미지 관리
```bash
# 이미지 조회
docker images

# 이미지 풀
docker pull nginx:latest

# 이미지 삭제
docker rmi nginx:latest

# 이미지 빌드
docker build -t myapp:latest .
```

### 2.2 컨테이너 관리
```bash
# 컨테이너 실행
docker run -d --name mycontainer -p 8080:8080 myapp:latest

# 컨테이너 목록 조회
docker ps
docker ps -a  # 중지된 컨테이너 포함

# 컨테이너 중지/시작
docker stop mycontainer
docker start mycontainer

# 컨테이너 로그 확인
docker logs mycontainer
docker logs -f mycontainer  # 실시간 로그
```

## 3. Dockerfile 작성 예시

### 3.1 Spring Boot 애플리케이션
```dockerfile
# Multi-stage build 예시
FROM maven:3.8.4-openjdk-11 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM openjdk:11-jre-slim
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

### 3.2 Node.js 애플리케이션
```dockerfile
FROM node:14-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

## 4. Docker Compose

### 4.1 다중 서비스 구성
```yaml
# docker-compose.yml
version: '3'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_HOST=db
    depends_on:
      - db
  
  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=rootpass
      - MYSQL_DATABASE=myapp
    volumes:
      - db-data:/var/lib/mysql

volumes:
  db-data:
```

## 근거 자료

### 1. 공식 문서
- [Docker Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

### 2. 모범 사례
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Container Security Best Practices](https://snyk.io/blog/10-docker-image-security-best-practices/)

### 3. 기술 블로그
- [Docker Blog](https://www.docker.com/blog/)
- [AWS Container Blog](https://aws.amazon.com/blogs/containers/)

## 실무 관련 추가 질문

1. "Docker 컨테이너의 로그 관리는 어떻게 하시나요?"

2. "컨테이너 모니터링은 어떤 도구를 사용하시나요?"

3. "Docker 이미지 크기를 최적화하는 방법은 무엇인가요?"

4. "Docker 보안을 위해 어떤 조치들을 취하시나요?"

## 실제 사용 예시

### 컨테이너 리소스 제한
```bash
docker run -d \
  --name myapp \
  --memory="512m" \
  --cpus="1.0" \
  -p 8080:8080 \
  myapp:latest
```

### 헬스체크 설정
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/health || exit 1
```

### 로그 설정
```bash
# JSON 로그 드라이버 사용
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp:latest
```

실제 면접에서는 이론적인 지식과 함께 Docker를 실제 프로젝트에서 어떻게 활용했는지, 발생한 문제들을 어떻게 해결했는지 구체적인 경험을 공유하는 것이 좋습니다.