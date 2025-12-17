# n8n Docker Setup

Docker Compose를 사용한 n8n 워크플로우 자동화 플랫폼 설정.

## 구성 요소

- **Traefik**: 리버스 프록시 + HTTPS (v3.2)
- **n8n**: 워크플로우 자동화 플랫폼
- **PostgreSQL**: 데이터베이스 (v16 Alpine)

## 주요 기능

- HTTPS 자동 발급 (Let's Encrypt)
- HTTP → HTTPS 자동 리다이렉트
- Traefik 대시보드 (Basic Auth 보호)
- PostgreSQL 데이터 영구 저장

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
| `docker compose logs -f traefik` | Traefik 로그 확인 |
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
           │ HTTP → HTTPS 리다이렉트
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
┌────────┐  ┌─────────────┐
│Traefik │  │     n8n     │
│Dashboard│  │    :5678    │
└────────┘  └──────┬──────┘
                   │
                   ▼
            ┌─────────────┐
            │  PostgreSQL │
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

## 참고

- [n8n 공식 문서](https://docs.n8n.io/)
- [Traefik 공식 문서](https://doc.traefik.io/traefik/)
- [Let's Encrypt](https://letsencrypt.org/)
- [n8n Docker 가이드](https://docs.n8n.io/hosting/installation/docker/)
