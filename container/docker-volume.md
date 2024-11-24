# Docker 볼륨과 스토리지에 대한 기술 면접 답변

## 주요 질문
"Docker의 데이터 저장 방식과 볼륨의 종류, 사용 사례에 대해 설명해주세요. 컨테이너의 데이터 영속성은 어떻게 보장하나요?"

## 1. Docker 스토리지 종류

### 1.1 스토리지 드라이버 시스템
```plaintext
[레이어 구조]
읽기/쓰기 레이어 (컨테이너)
│
이미지 레이어 3 (읽기 전용)
│
이미지 레이어 2 (읽기 전용)
│
이미지 레이어 1 (읽기 전용)
│
베이스 이미지 레이어 (읽기 전용)
```

### 1.2 데이터 저장 방식
```bash
# 1. 볼륨 (Volumes)
docker volume create myvolume
docker run -v myvolume:/app/data myapp

# 2. 바인드 마운트 (Bind Mounts)
docker run -v /host/path:/container/path myapp

# 3. tmpfs 마운트 (임시 저장소)
docker run --tmpfs /app/temp myapp
```

## 2. Docker 볼륨 관리

### 2.1 볼륨 생성과 관리
```bash
# 볼륨 생성
docker volume create --name mydata

# 볼륨 조회
docker volume ls

# 볼륨 상세 정보
docker volume inspect mydata

# 볼륨 삭제
docker volume rm mydata

# 사용되지 않는 볼륨 정리
docker volume prune
```

### 2.2 Docker Compose에서의 볼륨 설정
```yaml
version: '3'
services:
  db:
    image: postgres:13
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:6
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf:ro

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
```

## 3. 볼륨 백업과 복구

### 3.1 볼륨 백업
```bash
# 볼륨 백업
docker run --rm \
  -v myvolume:/source \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/myvolume-backup.tar.gz -C /source .

# 볼륨 복구
docker run --rm \
  -v myvolume:/target \
  -v $(pwd):/backup \
  ubuntu tar xzf /backup/myvolume-backup.tar.gz -C /target
```

### 3.2 데이터베이스 볼륨 백업
```bash
# PostgreSQL 백업
docker exec -t my-postgres \
  pg_dumpall -c -U postgres > dump_`date +%Y-%m-%d"_"%H_%M_%S`.sql

# MySQL 백업
docker exec my-mysql \
  sh -c 'exec mysqldump --all-databases -uroot -p"$MYSQL_ROOT_PASSWORD"' \
  > backup.sql
```

## 4. 볼륨 드라이버와 플러그인

### 4.1 볼륨 드라이버 사용
```bash
# NFS 볼륨 생성
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/dir \
  nfs_data

# AWS EBS 볼륨 사용
docker volume create \
  --driver rexray/ebs \
  --opt size=20 \
  --name ebs_volume
```

### 4.2 클라우드 스토리지 연동
```yaml
# AWS EFS 사용 예시
version: '3'
services:
  app:
    image: myapp
    volumes:
      - efs_data:/app/data

volumes:
  efs_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=fs-xxx.efs.region.amazonaws.com,nfsvers=4.1,rsize=1048576,wsize=1048576
      device: :/
```

## 근거 자료

### 1. 공식 문서
- [Docker Storage Overview](https://docs.docker.com/storage/)
- [Docker Volumes](https://docs.docker.com/storage/volumes/)
- [Docker Storage Drivers](https://docs.docker.com/storage/storagedriver/)

### 2. 모범 사례
- [Docker Storage Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Volume Plugins](https://docs.docker.com/engine/extend/plugins_volume/)

### 3. 클라우드 제공자 문서
- [AWS EBS with Docker](https://aws.amazon.com/blogs/compute/docker-volumes-and-amazon-ecs/)
- [Azure Storage with Docker](https://docs.microsoft.com/azure/container-instances/container-instances-volume-azure-files)

## 실무 관련 추가 질문

1. "데이터베이스 컨테이너의 데이터 영속성은 어떻게 보장하시나요?"

2. "볼륨 성능 모니터링은 어떻게 하시나요?"

3. "클라우드 환경에서 볼륨 관리 전략은 어떻게 되나요?"

4. "대용량 데이터를 다루는 컨테이너의 볼륨 설계는 어떻게 하시나요?"

## 실제 사용 예시

### 데이터베이스 볼륨 구성
```yaml
# PostgreSQL with Volume
version: '3'
services:
  db:
    image: postgres:13
    volumes:
      - type: volume
        source: pgdata
        target: /var/lib/postgresql/data
      - type: bind
        source: ./backup
        target: /backup
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    deploy:
      resources:
        limits:
          memory: 1G
    configs:
      - source: pg_config
        target: /etc/postgresql/postgresql.conf

volumes:
  pgdata:
    driver: local
    driver_opts:
      type: none
      device: /mnt/data/postgres
      o: bind

configs:
  pg_config:
    file: ./postgresql.conf
```

### 볼륨 모니터링 스크립트
```bash
#!/bin/bash
# 볼륨 사용량 모니터링
docker system df -v | grep "VOLUME NAME"
docker ps -q | xargs docker inspect \
  -f '{{ .Name }}{{ range .Mounts }}{{ if eq .Type "volume" }}
    Volume: {{ .Name }}
    Used By: {{ $.Name }}
    Mount: {{ .Destination }}
  {{ end }}{{ end }}'
```

실제 면접에서는 이론적인 지식과 함께 볼륨 관리 경험, 데이터 백업/복구 경험, 성능 최적화 경험 등을 구체적으로 설명하는 것이 좋습니다.