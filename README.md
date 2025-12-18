# n8n Docker Setup

Docker Compose를 사용한 n8n 워크플로우 자동화 플랫폼 설정.

## 구성 요소

- **Traefik**: 리버스 프록시 + HTTPS (v3.2)
- **n8n**: 워크플로우 자동화 플랫폼 (메인)
- **n8n Worker**: 워크플로우 실행 워커 (스케일 가능)
- **PostgreSQL**: 데이터베이스 (v16 Alpine)
- **Redis**: 캐시 및 큐 관리 (v7 Alpine)

## 주요 기능

- HTTPS 자동 발급 (Let's Encrypt)
- HTTP → HTTPS 자동 리다이렉트
- Traefik 대시보드 (Basic Auth 보호)
- PostgreSQL 데이터 영구 저장
- Redis 기반 큐 모드 및 캐싱
- Worker 수평 확장 (Horizontal Scaling)

## 빠른 시작

### 1. 환경변수 설정

```bash
cp .env.example .env
```

`.env` 파일 수정:
```env
N8N_DOMAIN=n8n.yourdomain.com
TRAEFIK_DOMAIN=traefik.yourdomain.com
ACME_EMAIL=your-email@example.com
POSTGRES_PASSWORD=your_secure_password_here
REDIS_PASSWORD=your_redis_password_here
N8N_WORKER_REPLICAS=2
```

### 2. Traefik 인증 설정

Basic Auth 비밀번호 생성:
```bash
# htpasswd 설치 (Ubuntu/Debian)
sudo apt-get install apache2-utils

# 비밀번호 해시 생성
echo $(htpasswd -nb admin yourpassword) | sed -e s/\\$/\\$\\$/g
```

생성된 값을 `.env`의 `TRAEFIK_AUTH`에 입력.

### 3. 실행

```bash
docker compose up -d
```

### 4. 접속

- **n8n**: https://n8n.yourdomain.com
- **Traefik Dashboard**: https://traefik.yourdomain.com

> ⚠️ Let's Encrypt는 실제 도메인과 공인 IP가 필요합니다.
> 로컬 테스트 시에는 자체 서명 인증서가 사용됩니다.

## 명령어

| 명령어 | 설명 |
|--------|------|
| `docker compose up -d` | 백그라운드 실행 |
| `docker compose down` | 중지 |
| `docker compose logs -f` | 전체 로그 확인 |
| `docker compose logs -f n8n` | n8n 로그 확인 |
| `docker compose logs -f n8n-worker` | Worker 로그 확인 |
| `docker compose logs -f traefik` | Traefik 로그 확인 |
| `docker compose logs -f redis` | Redis 로그 확인 |
| `docker compose ps` | 상태 확인 |
| `docker compose restart` | 재시작 |

## 환경변수

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `N8N_DOMAIN` | n8n.localhost | n8n 도메인 |
| `N8N_PROTOCOL` | https | 프로토콜 |
| `TRAEFIK_DOMAIN` | traefik.localhost | Traefik 대시보드 도메인 |
| `TRAEFIK_AUTH` | - | Traefik 대시보드 인증 |
| `ACME_EMAIL` | - | Let's Encrypt 이메일 |
| `GENERIC_TIMEZONE` | Asia/Seoul | 타임존 |
| `POSTGRES_DB` | n8n | DB 이름 |
| `POSTGRES_USER` | n8n | DB 사용자 |
| `POSTGRES_PASSWORD` | - | DB 비밀번호 |
| `REDIS_PASSWORD` | - | Redis 비밀번호 |
| `N8N_WORKER_REPLICAS` | 2 | Worker 인스턴스 수 |

## Worker 설정

### 개요

n8n Worker는 Redis 큐에서 워크플로우 실행 작업을 가져와 처리합니다. 메인 n8n 인스턴스는 UI와 Webhook을 담당하고, Worker가 실제 워크플로우를 실행합니다.

**장점**:
- 워크플로우 실행 부하 분산
- 수평 확장으로 처리량 증가
- 메인 인스턴스 안정성 향상
- 무중단 Worker 스케일링

### 스케일링

```bash
# .env 파일에서 Worker 수 변경
N8N_WORKER_REPLICAS=4

# 변경 적용
docker compose up -d

# 또는 명령어로 직접 스케일링
docker compose up -d --scale n8n-worker=4
```

### Worker 모니터링

```bash
# Worker 상태 확인
docker compose ps n8n-worker

# 모든 Worker 로그 확인
docker compose logs -f n8n-worker

# 특정 Worker 로그 확인
docker logs n8n-n8n-worker-1 -f

# Worker 리소스 사용량
docker stats $(docker compose ps -q n8n-worker)
```

### 권장 Worker 수

| 워크플로우 규모 | 권장 Worker 수 |
|----------------|----------------|
| 소규모 (일 100건 미만) | 1-2 |
| 중규모 (일 1,000건) | 2-4 |
| 대규모 (일 10,000건+) | 4-8+ |

> Worker 수는 서버 CPU 코어 수와 메모리를 고려하여 설정하세요.

## Redis 설정

### 큐 모드

n8n은 Redis를 사용하여 워크플로우 실행을 큐로 관리합니다.

**장점**:
- 워크플로우 실행의 안정성 향상
- 서버 재시작 시에도 실행 상태 유지
- 대규모 워크플로우 처리에 적합

**관련 환경변수**:
```env
EXECUTIONS_MODE=queue
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_BULL_REDIS_PASSWORD=${REDIS_PASSWORD}
```

### 캐싱

Redis 캐시를 통해 n8n 성능을 향상시킵니다.

**장점**:
- API 응답 속도 향상
- 데이터베이스 부하 감소
- 반복 조회 최적화

**관련 환경변수**:
```env
N8N_CACHE_ENABLED=true
N8N_CACHE_BACKEND=redis
N8N_CACHE_REDIS_HOST=redis
N8N_CACHE_REDIS_PORT=6379
N8N_CACHE_REDIS_PASSWORD=${REDIS_PASSWORD}
```

### Redis 모니터링

```bash
# Redis 연결 테스트
docker compose exec redis redis-cli -a $REDIS_PASSWORD ping

# Redis 정보 확인
docker compose exec redis redis-cli -a $REDIS_PASSWORD info

# 큐 상태 확인
docker compose exec redis redis-cli -a $REDIS_PASSWORD keys 'bull:*'

# 메모리 사용량 확인
docker compose exec redis redis-cli -a $REDIS_PASSWORD info memory
```

## HTTPS 설정

### Let's Encrypt (기본 활성화)

현재 설정은 Let's Encrypt를 통한 자동 HTTPS 인증서 발급이 기본 활성화되어 있습니다.

**요구사항**:
- 실제 도메인 (DNS 설정 완료)
- 서버 공인 IP
- 80, 443 포트 오픈

**동작 방식**:
1. HTTP 요청이 들어오면 자동으로 HTTPS로 리다이렉트
2. Let's Encrypt에서 자동으로 인증서 발급
3. 인증서 자동 갱신

### 로컬 개발 환경

로컬에서 테스트할 경우 `*.localhost` 도메인 사용:
- 브라우저에서 자체 서명 인증서 경고 발생 (무시 가능)
- Let's Encrypt 인증서 발급 불가 (정상 동작)

## 데이터 백업

### PostgreSQL 백업
```bash
docker compose exec postgres pg_dump -U n8n n8n > backup.sql
```

### PostgreSQL 복원
```bash
docker compose exec -T postgres psql -U n8n n8n < backup.sql
```

### Redis 백업
```bash
# RDB 스냅샷 강제 생성
docker compose exec redis redis-cli -a $REDIS_PASSWORD bgsave

# RDB 파일 복사
docker compose cp redis:/data/dump.rdb ./redis-backup.rdb
```

### 인증서 백업
```bash
docker compose cp traefik:/etc/traefik/acme.json ./acme.json
```

## 파일 구조

```
.
├── docker-compose.yml  # Docker Compose 설정
├── .env                # 환경변수 (Git 제외)
├── .env.example        # 환경변수 템플릿
├── .gitignore          # Git 제외 목록
└── README.md           # 이 문서
```

## 아키텍처

```
            인터넷
               │
               ▼
        ┌─────────────┐
        │   Traefik   │ ← Let's Encrypt HTTPS
        │  :80 / :443 │
        └──────┬──────┘
               │
        ┌──────┴──────┐
        │             │
        ▼             ▼
   ┌────────┐   ┌───────────┐
   │Traefik │   │    n8n    │ ← UI / Webhooks
   │Dashboard│   │   :5678   │
   └────────┘   └─────┬─────┘
                      │
                      ▼
               ┌─────────────┐
               │    Redis    │ ← Job Queue
               │    :6379    │
               └──────┬──────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Worker  │  │ Worker  │  │ Worker  │
   │   #1    │  │   #2    │  │   #N    │
   └────┬────┘  └────┬────┘  └────┬────┘
        │             │             │
        └─────────────┼─────────────┘
                      │
                      ▼
               ┌─────────────┐
               │ PostgreSQL  │ ← Data Storage
               │    :5432    │
               └─────────────┘
```

## 문제 해결

### 인증서 발급 실패
```bash
# Traefik 로그 확인
docker compose logs traefik | grep -i acme

# acme.json 권한 확인 (600 필요)
docker compose exec traefik ls -la /etc/traefik/
```

### 502 Bad Gateway
```bash
# n8n 상태 확인
docker compose ps
docker compose logs n8n
```

### 데이터베이스 연결 실패
```bash
# PostgreSQL 상태 확인
docker compose exec postgres pg_isready -U n8n -d n8n
```

### Redis 연결 실패
```bash
# Redis 상태 확인
docker compose exec redis redis-cli -a $REDIS_PASSWORD ping

# Redis 로그 확인
docker compose logs redis
```

### Worker 문제
```bash
# Worker 상태 확인
docker compose ps n8n-worker

# Worker 로그 확인
docker compose logs n8n-worker

# Worker 재시작
docker compose restart n8n-worker

# 특정 Worker만 재시작
docker restart n8n-n8n-worker-1
```

### 큐 적체 (Queue Backlog)
```bash
# 큐 대기 작업 수 확인
docker compose exec redis redis-cli -a $REDIS_PASSWORD llen bull:jobs:wait

# Worker 스케일 업
docker compose up -d --scale n8n-worker=4
```

## 참고

- [n8n 공식 문서](https://docs.n8n.io/)
- [n8n Queue Mode](https://docs.n8n.io/hosting/scaling/queue-mode/)
- [n8n Scaling Guide](https://docs.n8n.io/hosting/scaling/)
- [Traefik 공식 문서](https://doc.traefik.io/traefik/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Redis 공식 문서](https://redis.io/docs/)
