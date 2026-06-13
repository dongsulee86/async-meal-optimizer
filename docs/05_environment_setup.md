# 📄 Multi-Environment Infrastructure & Setup Guide

본 문서는 로컬 PC(Windows), 외부 서브 PC, 그리고 클라우드 인프라(Linux/AWS) 등 **어떤 환경에서든 구애받지 않고 단일 커맨드로 백엔드와 데이터베이스 시스템을 동일하게 구동하기 위한 멀티 환경 아키텍처 가이드**입니다.

---

## 1. 환경 격리 전략 (Environment Isolation Strategy)

어느 환경에서나 소스 코드의 수정 없이 구동될 수 있도록, 모든 인프라 연결 설정과 자격 증명(Credential)은 소스 코드와 완전히 분리하여 **환경 변수(Environment Variables)**로 주입받는 방식을 채택합니다.

```text
[ 소스 코드 (동일) ] ──► 환경 변수(.env) 분리 ──► [ Local PC Environment ]
                                              ──► [ Remote PC Environment ]
                                              ──► [ Cloud Prod Environment ]
```

---

## 2. 어디서나 구동 가능한 Docker Compose 구성

환경에 따라 호스트 포트 충돌을 방지하고 보안을 강화하기 위해, 데이터베이스 정보를 외부 변수(`${...}`)로 동적 주입받도록 `docker-compose.yml`을 설계합니다.

### 🛠️ `docker-compose.yml`
```yaml
version: '3.8'

services:
  postgres:
    image: ankane/pgvector:v0.5.1
    container_name: meal-optimizer-db
    environment:
      POSTGRES_USER: ${DB_USER:-meal_user}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-meal_password123!}
      POSTGRES_DB: ${DB_NAME:-meal_db}
      TZ: Asia/Seoul
    ports:
      - "${DB_PORT:-5432}:5432" # 다른 PC에서 5432 포트가 이미 사용 중이라면 .env에서 변경 가능
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always

volumes:
  postgres_data:
    driver: local
```

---

## 3. 환경별 변수 설정 파일 (`.env` 스펙)

프로젝트 루트에 `.env` 파일을 생성하여 해당 장비/환경에 맞는 설정을 주입합니다. (이 파일은 보안상 깃허브 레포지토리 관리 대상에서 제외(`git rm --cached`)해야 합니다.)

### 💻 Case A: 로컬 Windows 개발 PC 용 `.env`
```text
DB_USER=meal_user
DB_PASSWORD=meal_password123!
DB_NAME=meal_db
DB_PORT=5432
DATABASE_URL=postgresql+asyncpg://meal_user:meal_password123!@localhost:5432/meal_db
APP_ENV=local
```

### ☁️ Case B: 다른 PC 또는 클라우드(Linux/AWS) 운영 환경 용 `.env`
* 다른 장비에서 포트 충돌이 나거나 운영 환경 보안을 강화할 때 소스 코드 수정 없이 `.env` 파일만 수정합니다.
```text
DB_USER=prod_secure_admin
DB_PASSWORD=strong_password_999!!
DB_NAME=prod_meal_db
DB_PORT=5433 # 포트 충돌 우회
DATABASE_URL=postgresql+asyncpg://prod_secure_admin:strong_password_999!!@localhost:5433/prod_meal_db
APP_ENV=prod
```

---

## 4. 파이썬 가상환경 이식성 통합 스크립트

윈도우와 리눅스 등 OS가 다른 환경에서도 의존성 라이브러리 버전을 100% 동일하게 고정(Locking)하기 위해 `requirements.txt` 기반으로 가상환경을 셋업합니다.

### 🛠️ 의존성 정의 파일 (`requirements.txt`)
```text
fastapi==0.110.0
uvicorn[standard]==0.28.0
sqlalchemy[asyncio]==2.0.28
asyncpg==0.29.0
pydantic==2.6.4
httpx==0.27.0
python-dotenv==1.0.1
```

### 🚀 크로스 플랫폼 구동 절차

#### ① Windows (개발 PC / 노트북) 실행 명령
```powershell
# 1. 가상환경 생성 및 활성화
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2. 패키지 설치
pip install -r requirements.txt

# 3. 도커 인프라 구동 (자동으로 .env 변수 감지)
docker-compose up -d
```

#### ② Mac / Linux / AWS EC2 실행 명령
```bash
# 1. 가상환경 생성 및 활성화
python3 -m venv .venv
source .venv/bin/activate

# 2. 패키지 설치
pip install -r requirements.txt

# 3. 도커 인프라 구동
docker-compose up -d
```

---

## 5. 데이터베이스 무결성 검증 및 확장 모듈 활성화

장비를 이동하여 도커를 새로 띄운 경우, 데이터베이스 도구(DBeaver 등)로 접속하여 아래 스크립트를 최초 1회 실행하여 외부 환경에서도 벡터 검색 알고리즘이 정상 동작하도록 인프라를 정렬합니다.

```sql
-- pgvector 확장 모듈 활성화
CREATE EXTENSION IF NOT EXISTS vector;

-- 타 장비 타임존 정상 동기화 여부 확인 (Asia/Seoul이 나와야 함)
SHOW timezone;
```
