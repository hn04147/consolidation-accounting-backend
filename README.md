# consolidation-accounting-backend

## FastAPI 백엔드 서버 개발 및 AWS EC2 배포 가이드 (pyproject.toml 포함)

이 문서는 FastAPI, PostgreSQL (SQLAlchemy, Alembic), Docker를 활용하여 백엔드 서버를 개발하고 AWS EC2에 배포하는 과정을 `pyproject.toml`을 통한 패키지 관리를 포함하여 단계별로 설명합니다.

---

### 1. 프로젝트 구조

클린 아키텍처 원칙을 적용하여 **모듈성과 확장성**을 고려한 프로젝트 구조입니다.

```

your\_project\_name/
├── .uv/                        \# UV 가상 환경 파일 (UV 관리)
├── .gitignore                  \# Git 추적 제외 파일
├── pyproject.toml              \# 프로젝트 메타데이터 및 의존성 목록 (PEP 621 표준)
├── main.py                     \# FastAPI 애플리케이션 진입점
├── app/
│   ├── **init**.py             \# Python 모듈로 인식
│   ├── core/                   \# 핵심 설정 및 유틸리티
│   │   ├── config.py           \# 환경 변수 및 애플리케이션 설정
│   │   ├── database.py         \# DB 세션 및 엔진 초기화
│   │   └── security.py         \# 인증/인가 관련 유틸리티 (JWT 등)
│   ├── api/                    \# API 엔드포인트
│   │   ├── v1/
│   │   │   ├── endpoints/
│   │   │   │   ├── users.py    \# 사용자 관련 API
│   │   │   │   └── items.py    \# 아이템 관련 API
│   │   │   └── routers.py      \# v1 라우터 통합
│   ├── crud/                   \# DB CRUD 로직
│   │   ├── base.py             \# 제네릭 CRUD 기본 클래스
│   │   ├── users.py            \# 사용자 CRUD
│   │   └── items.py            \# 아이템 CRUD
│   ├── models/                 \# SQLAlchemy ORM 모델 정의
│   │   ├── base.py             \# 모든 모델의 공통 베이스
│   │   ├── user.py             \# User 모델
│   │   └── item.py             \# Item 모델
│   ├── schemas/                \# Pydantic 모델 (요청/응답 유효성 검사)
│   │   ├── user.py             \# User 스키마
│   │   └── item.py             \# Item 스키마
│   └── dependencies/           \# API 종속성 주입 (DB 세션, 현재 사용자 등)
│       └── common.py
├── migrations/                 \# Alembic 마이그레이션 파일
│   ├── versions/               \# 마이그레이션 스크립트
│   ├── env.py                  \# Alembic 환경 설정
│   └── alembic.ini             \# Alembic 전역 설정
├── tests/                      \# 테스트 코드
│   └── ...
├── Dockerfile                  \# Docker 이미지 빌드 파일
├── docker-compose.yml          \# 개발 환경용 Docker Compose 파일
├── .env.example                \# 환경 변수 예시
└── README.md                   \# 프로젝트 설명

```

---

### 2. 프로젝트 개발 및 로컬 환경 설정 절차

1.  **프로젝트 초기화 및 UV 설정**:
    * 프로젝트 디렉토리를 생성하고 이동합니다: `mkdir your_project_name && cd your_project_name`
    * **UV 설치**: 시스템 Python 환경에 `pip install uv`로 UV를 설치합니다.
    * **UV 가상 환경 생성 및 활성화**: `uv venv`를 실행한 후, 운영체제에 맞춰 가상 환경을 활성화합니다 (Windows: `.\.uv\Scripts\activate`, macOS/Linux: `source ./.uv/bin/activate`).
    * `.gitignore` 파일을 생성하고 `.uv/`, `__pycache__/`, `.env` 등을 추가하여 Git에서 추적되지 않도록 합니다.

2.  **`pyproject.toml` 작성**:
    * `pyproject.toml` 파일을 생성하고 프로젝트의 메타데이터와 의존성을 정의합니다.

    ``` toml
    [project]
    name = "your_project_name"
    version = "0.1.0"
    description = "My FastAPI Backend Project with pyproject.toml"
    dependencies = [
        "fastapi",
        "uvicorn[standard]",
        "sqlalchemy",
        "psycopg2-binary",
        "alembic",
        "pydantic-settings",
    ]

    [project.optional-dependencies]
    dev = [
        "pytest",
        "httpx", # FastAPI 테스트 클라이언트
        "ruff", # 코드 린터
        "black", # 코드 포매터
    ]

    [build-system]
    requires = ["setuptools>=61.0"]
    build-backend = "setuptools.build_meta"
    ```

3.  **패키지 설치 (UV + `pyproject.toml`)**:
    * **기본 의존성 설치**: 가상 환경이 활성화된 상태에서 `uv pip install -e .` 명령어를 실행합니다. 이 명령은 `pyproject.toml`의 `[project].dependencies`에 정의된 패키지들을 설치합니다. `-e .`는 editable 모드로, 개발 중 코드 변경 시 바로 반영됩니다.
    * **개발 의존성 설치 (선택 사항)**: `uv pip install -e ".[dev]"` 명령어를 실행하여 `[project.optional-dependencies].dev`에 정의된 개발용 패키지들도 함께 설치합니다.

4.  **프로젝트 기본 파일 생성**:
    * `main.py`, `app/` 디렉토리와 그 하위 구조를 위에 제시된 형태로 생성합니다.

5.  **`app/core/config.py` 작성**:
    * `pydantic-settings`를 활용하여 환경 변수를 관리하는 클래스를 정의합니다.

    ``` python
    from pydantic_settings import BaseSettings, SettingsConfigDict

    class Settings(BaseSettings):
        DATABASE_URL: str
        SECRET_KEY: str
        ALGORITHM: str = "HS256"
        ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

        model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    settings = Settings()
    ```

6.  **`app/core/database.py` 작성**:
    * SQLAlchemy 엔진, 세션 및 ORM `Base` 객체를 설정합니다.

    ``` python
    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker, declarative_base
    from app.core.config import settings

    SQLALCHEMY_DATABASE_URL = settings.DATABASE_URL
    engine = create_engine(SQLALCHEMY_DATABASE_URL)
    SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
    Base = declarative_base()

    def get_db():
        db = SessionLocal()
        try:
            yield db
        finally:
            db.close()
    ```

7.  **`app/models/` 디렉토리 및 ORM 모델 정의**:
    * `app/models/base.py`에서 `Base`를 임포트하여 SQLAlchemy ORM 모델의 기본 클래스로 사용합니다.
    * 예시: `app/models/user.py`

    ```python
    from sqlalchemy import Column, Integer, String
    from app.models.base import Base

    class User(Base):
        __tablename__ = "users"
        id = Column(Integer, primary_key=True, index=True)
        email = Column(String, unique=True, index=True, nullable=False)
        hashed_password = Column(String, nullable=False)
    ```

8.  **Alembic 설정 및 마이그레이션**:
    * **Alembic 초기화**: 프로젝트 루트에서 `alembic init migrations` 명령어를 실행하여 `migrations/` 디렉토리와 `alembic.ini` 파일을 생성합니다.
    * **`alembic.ini` 수정**: `sqlalchemy.url =` 부분을 주석 처리하고 `script_location = migrations`가 올바른지 확인합니다.
    * **`migrations/env.py` 수정**:
        * 프로젝트 루트를 `sys.path`에 추가하여 `app` 모듈을 임포트할 수 있도록 합니다.
        * `target_metadata = None`을 **`target_metadata = Base.metadata`** 로 변경합니다 (반드시 `app.models.base.Base`를 임포트해야 합니다).
        * `config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)`를 사용하여 `DATABASE_URL`을 `config.py`에서 가져오도록 설정합니다.
    * **첫 마이그레이션 생성**: `alembic revision --autogenerate -m "create initial tables"` 명령어로 초기 테이블 생성을 위한 마이그레이션 스크립트를 생성합니다.
    * **마이그레이션 적용**: `alembic upgrade head` 명령어를 실행하여 데이터베이스에 변경사항을 반영합니다.

9.  **`app/schemas/` 및 Pydantic 스키마 정의**:
    * API 요청 및 응답 데이터를 위한 Pydantic 모델을 정의합니다 (예: `UserCreate`, `UserResponse`).

10. **`app/crud/` 및 CRUD 로직 정의**:
    * 데이터베이스와의 상호작용(Create, Read, Update, Delete) 로직을 정의합니다. `base.py`를 통해 제네릭 CRUD를 구현할 수 있습니다.

11. **`app/api/v1/endpoints/` 및 API 엔드포인트 정의**:
    * 각 리소스별 `APIRouter`를 사용하여 API 엔드포인트를 정의합니다 (예: `users.py`, `items.py`).

12. **`app/api/v1/routers.py`에서 라우터 통합**:
    * 모든 `APIRouter`들을 한곳에 모아 `main.py`에서 `app.include_router`로 한 번에 등록할 수 있도록 합니다.

    ``` python
    from fastapi import APIRouter
    from app.api.v1.endpoints import users, items

    api_router = APIRouter()
    api_router.include_router(users.router, prefix="/users", tags=["users"])
    api_router.include_router(items.router, prefix="/items", tags=["items"])
    # 다른 라우터가 있다면 여기에 추가합니다.
    ```

13. **`main.py` 작성**:
    * FastAPI 애플리케이션의 메인 진입점입니다.

    ``` python
    from fastapi import FastAPI
    from app.api.v1.routers import api_router

    app = FastAPI(
        title="My FastAPI Project",
        description="A sample FastAPI project with Docker, SQLAlchemy, Alembic",
        version="1.0.0",
    )

    app.include_router(api_router, prefix="/api/v1")

    # 애플리케이션 시작/종료 이벤트 처리 (선택 사항)
    @app.on_event("startup")
    async def startup_event():
        print("Application startup")

    @app.on_event("shutdown")
    async def shutdown_event():
        print("Application shutdown")

    if __name__ == "__main__":
        import uvicorn
        uvicorn.run(app, host="0.0.0.0", port=8000)
    ```

14. **`.env` 파일 생성 (로컬 개발용)**:
    * 데이터베이스 연결 정보 및 시크릿 키 등을 포함하는 `.env` 파일을 생성합니다.
    * **절대 Git에 올리지 마세요!** 반드시 `.gitignore`에 추가해야 합니다.

    ``` env
    DATABASE_URL=postgresql://user:password@localhost:5432/your_database_name
    SECRET_KEY=your_super_secret_key_for_jwt
    ```

15. **`Dockerfile` 작성**:
    * Docker 이미지를 빌드하는 방법을 정의하는 파일입니다. `pyproject.toml`을 활용하여 의존성을 설치하도록 변경됩니다.

    ``` dockerfile
    # Python 3.11 Slim 이미지 사용 (더 가볍습니다)
    FROM python:3.11-slim-buster

    # 작업 디렉토리 설정
    WORKDIR /app

    # 시스템 패키지 업데이트 및 필요한 의존성 설치
    # psycopg2-binary를 위한 libpq-dev, 그 외 빌드 도구 설치
    RUN apt-get update && \
        apt-get install -y --no-install-recommends \
            gcc \
            postgresql-client \
            libpq-dev && \
        rm -rf /var/lib/apt/lists/*

    # UV 설치 (빌드 시 필요)
    RUN pip install uv

    # pyproject.toml 복사 및 의존성 설치
    COPY pyproject.toml .
    RUN uv pip install . # pyproject.toml의 [project].dependencies를 설치합니다.

    # 애플리케이션 코드 복사 (의존성 설치 후)
    COPY . .

    # 포트 노출
    EXPOSE 8000

    # 시작 명령어 (Gunicorn + Uvicorn Worker 권장, 여기서는 Uvicorn 직접 실행)
    # CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "main:app", "--bind", "0.0.0.0:8000"]
    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```

16. **`docker-compose.yml` 작성 (로컬 개발용)**:
    * FastAPI 애플리케이션 컨테이너와 PostgreSQL 데이터베이스 컨테이너를 함께 띄울 때 사용합니다.

    ``` yaml
    version: '3.8'

    services:
      db:
        image: postgres:15-alpine
        restart: always
        environment:
          POSTGRES_DB: your_database_name
          POSTGRES_USER: user
          POSTGRES_PASSWORD: password
        ports:
          - "5432:5432"
        volumes:
          - db_data:/var/lib/postgresql/data # 데이터 영속성을 위한 볼륨

      web:
        build: . # 현재 디렉토리의 Dockerfile 사용
        command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload # 개발 시 --reload 추가
        volumes:
          - .:/app # 코드 변경 시 즉시 반영 (개발용)
        ports:
          - "8000:8000"
        env_file:
          - .env # .env 파일에서 환경 변수 로드
        depends_on:
          - db # DB 컨테이너가 먼저 시작되도록 설정

    volumes:
      db_data: # 볼륨 정의
    ```

17. **로컬에서 Docker로 실행**:
    * DB 컨테이너와 앱 컨테이너를 함께 시작하려면 `docker-compose up --build` 명령어를 사용합니다.
    * 백그라운드에서 실행하려면 `-d` 옵션을 추가하여 `docker-compose up -d --build`를 실행합니다.
    * 이제 웹 브라우저에서 `http://localhost:8000/docs`로 접속하여 FastAPI Swagger UI를 확인할 수 있습니다.

---

### 3. AWS EC2 배포 절차

이제 준비된 Docker 이미지를 AWS EC2에 배포하는 절차입니다.

#### 3.1. AWS 사전 준비

1.  **AWS 계정 생성 및 로그인**:
    * 아직 AWS 계정이 없다면, [AWS 웹사이트](https://aws.amazon.com/)에서 계정을 생성하고 로그인합니다.
2.  **AWS RDS (Relational Database Service) for PostgreSQL 인스턴스 생성**:
    * **DB 엔진**: `PostgreSQL`을 선택합니다.
    * **버전 및 템플릿**: 적절한 버전과 개발/테스트 또는 프로덕션 템플릿을 선택합니다.
    * **DB 인스턴스 식별자, 마스터 사용자 이름, 마스터 암호**를 설정합니다.
    * **VPC 보안 그룹** 생성: EC2 인스턴스가 RDS에 접근할 수 있도록 인바운드 규칙에 EC2의 보안 그룹을 추가하고 포트는 `5432`로 설정합니다.
    * **퍼블릭 액세스**: 보안상 '아니요'로 설정하는 것이 좋습니다. 이 경우 EC2 인스턴스가 RDS와 같은 VPC 내에 있어야 합니다.
    * 생성된 RDS 인스턴스의 **엔드포인트**를 기록해 둡니다. 이 엔드포인트는 `DATABASE_URL`에 사용될 호스트 주소가 됩니다.
3.  **AWS ECR (Elastic Container Registry) 리포지토리 생성**:
    * Docker Hub 대신 AWS 내부 레지스트리인 ECR을 사용하는 것이 보안 및 성능 면에서 이점이 있습니다.
    * **리포지토리 이름**을 지정하고 생성합니다.
    * 이미지를 푸시하기 위한 **로그인/푸시 명령어**를 기록해 둡니다.
4.  **AWS Systems Manager Parameter Store 또는 Secrets Manager 설정**:
    * 데이터베이스 연결 정보, 시크릿 키 등 민감한 정보를 안전하게 저장하고 관리합니다.
    * **Parameter Store (비교적 덜 민감한 정보)**: `/your-project-name/db-url`, `/your-project-name/secret-key` 등으로 파라미터를 생성하고 값을 입력합니다 (`SecureString` 타입 권장).
    * **Secrets Manager (더 민감한 정보)**: 비밀번호, API 키 등을 저장합니다.
    * EC2 인스턴스에 이 파라미터/시크릿에 접근할 수 있는 **IAM Role**을 부여해야 합니다.

#### 3.2. Docker 이미지 빌드 및 푸시

1.  **FastAPI 프로젝트의 최신 코드 확인**: 로컬에서 작업한 코드가 모두 Git에 커밋되었는지 확인합니다.
2.  **Docker 이미지 빌드**:
    * 프로젝트 루트 디렉토리에서 다음 명령어를 실행합니다.
        `docker build -t your-project-name:latest .`
        (`your-project-name`은 실제 프로젝트 이름으로 변경)
3.  **ECR 레지스트리에 로그인 (터미널에서)**:
    * AWS 콘솔의 ECR 리포지토리 페이지에서 "푸시 명령 보기"를 클릭하여 나오는 로그인 명령어를 복사하여 실행합니다. 일반적으로 다음과 같은 형식입니다.
        `aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.<your-region>.amazonaws.com`
4.  **이미지 태그 지정**:
    * 빌드한 로컬 이미지를 ECR 리포지토리 URL로 태그합니다.
        `docker tag your-project-name:latest <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/your-project-name:latest`
5.  **ECR로 이미지 푸시**:
    * `docker push <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/your-project-name:latest` 명령어를 실행하여 이미지를 ECR에 업로드합니다.

#### 3.3. AWS EC2 인스턴스 설정

1.  **EC2 인스턴스 시작**:
    * **AMI (Amazon Machine Image)**: `Amazon Linux 2023` 또는 `Ubuntu` LTS 버전을 권장합니다.
    * **인스턴스 유형**: `t2.micro` (프리티어) 또는 적절한 인스턴스 유형을 선택합니다.
    * **키 페어**: SSH 접속을 위한 키 페어를 생성하거나 기존 키 페어를 사용합니다.
    * **네트워크 설정**:
        * **VPC**: 위에서 설정한 VPC 또는 기본 VPC를 사용합니다.
        * **서브넷**: 공용 서브넷 (인터넷 접근 가능)을 선택합니다.
        * **퍼블릭 IP 자동 할당**: '활성화'로 설정하여 SSH 및 웹 접근이 가능하도록 합니다.
        * **보안 그룹 생성**:
            * **SSH (Port 22)**: 자신의 IP 주소에서만 접근을 허용합니다.
            * **HTTP (Port 80)**: `0.0.0.0/0` (모든 IP)를 허용합니다.
            * **HTTPS (Port 443)**: `0.0.0.0/0` (모든 IP)를 허용합니다 (Nginx 사용 시).
            * **FastAPI 포트 (Port 8000)**: 외부에서 8000번 포트로 직접 접근하는 것은 권장하지 않습니다. Nginx와 같은 리버스 프록시를 사용할 경우, Nginx가 실행되는 EC2 인스턴스의 보안 그룹 ID만 허용하도록 설정합니다. (즉, 자기 자신의 보안 그룹에서 8000번 포트 인바운드를 허용).
    * **스토리지**: 기본값으로 충분합니다.
    * **고급 세부 정보 (IAM 인스턴스 프로필)**: EC2 인스턴스가 ECR에서 이미지를 당겨오고, Parameter Store/Secrets Manager에서 파라미터를 읽어올 수 있도록 권한을 부여하는 **IAM 역할**을 생성하여 인스턴스에 할당합니다 (예: `ECR Pull`, `SSM Read`, `Secrets Manager Read` 권한 포함).

2.  **EC2 인스턴스 접속**:
    * SSH 클라이언트(예: PuTTY, Git Bash, macOS/Linux 터미널)를 사용하여 접속합니다.
    * `ssh -i /path/to/your-key.pem ec2-user@<your-ec2-public-ip-or-dns>`

3.  **Docker 설치 및 설정 (EC2 인스턴스 내부에서)**:
    * **Amazon Linux 2023**:
        `sudo yum update -y`
        `sudo yum install docker -y`
        `sudo service docker start`
        `sudo usermod -a -G docker ec2-user` (ec2-user가 docker 명령어를 sudo 없이 실행하도록)
        `newgrp docker` (그룹 변경 즉시 적용)
    * **Ubuntu**:
        `sudo apt update`
        `sudo apt install docker.io -y`
        `sudo systemctl start docker`
        `sudo systemctl enable docker`
        `sudo usermod -aG docker $USER`
        `newgrp docker`

4.  **Nginx 설치 및 설정 (리버스 프록시)**:
    * Nginx를 설치하고 (`sudo yum install nginx -y` 또는 `sudo apt install nginx -y`) 설정 파일을 수정합니다.
    * `sudo nano /etc/nginx/nginx.conf` 또는 `sudo nano /etc/nginx/conf.d/default.conf` 파일을 열어 `http` 블록 안에 `server` 블록을 추가하거나 수정합니다.

    ``` nginx
    server {
        listen 80;
        server_name your_domain_name_or_ec2_public_ip; # 도메인 연결 시 도메인명, 아니면 EC2 퍼블릭 IP

        location / {
            proxy_pass [http://127.0.0.1:8000](http://127.0.0.1:8000); # FastAPI 컨테이너가 8000번 포트에서 실행 중이라고 가정
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
    * Nginx 시작 및 자동 시작 설정: `sudo systemctl start nginx`, `sudo systemctl enable nginx`.
    * **방화벽 설정 (Ubuntu의 경우)**:
        `sudo ufw allow 'Nginx HTTP'`
        `sudo ufw enable`

#### 3.4. Docker 컨테이너 실행

1.  **ECR 레지스트리에 로그인 (EC2 인스턴스 내부에서)**:
    * 앞서 ECR 리포지토리에서 복사한 로그인 명령어를 EC2 인스턴스 SSH 터미널에서 다시 실행합니다.
2.  **환경 변수 설정**:
    * **Parameter Store/Secrets Manager 사용 시 (권장)**: 애플리케이션 시작 스크립트에서 AWS CLI/SDK (`boto3` 라이브러리 사용)를 사용하여 환경 변수를 동적으로 읽어오도록 설정합니다. 이는 가장 안전하고 권장되는 방법입니다.
    * **Docker `run` 명령으로 직접 주입 시 (간단한 경우)**:
        `docker run -d --name fastapi_app -p 8000:8000 \`
        `-e DATABASE_URL="postgresql://<user>:<password>@<rds-endpoint>:5432/<db_name>" \`
        `-e SECRET_KEY="<your_jwt_secret_key>" \`
        `<your-account-id>.dkr.ecr.<your-region>.amazonaws.com/your-project-name:latest`
        **주의:** `-e`로 직접 주입하는 것은 민감 정보 보안에 취약하므로 AWS 서비스를 활용하는 것을 강력히 권장합니다.
3.  **Alembic 마이그레이션 적용**:
    * 데이터베이스 스키마 변경 시 마이그레이션을 적용해야 합니다.
    * **방법 1 (서비스 시작 전 별도 컨테이너로 마이그레이션)**:
        ```bash
        # 마이그레이션 전용 컨테이너를 일회성으로 실행합니다.
        docker run --rm \
        -e DATABASE_URL="postgresql://<user>:<password>@<rds-endpoint>:5432/<db_name>" \
        <your-account-id>.dkr.ecr.<your-region>[.amazonaws.com/your-project-name:latest](https://.amazonaws.com/your-project-name:latest) \
        alembic upgrade head
        ```
    * **방법 2 (CI/CD 파이프라인에서 마이그레이션)**: 배포 파이프라인의 특정 단계에서 마이그레이션 컨테이너를 실행하여 DB를 최신 상태로 유지하는 것이 **가장 권장되는 프로덕션 방법**입니다.
4.  **FastAPI Docker 컨테이너 실행**:
    * `docker run -d --name fastapi_app -p 8000:8000 --restart always \`
    * 위에서 설정한 방식으로 환경 변수를 전달합니다.
    * `<your-account-id>.dkr.ecr.<your-region>.amazonaws.com/your-project-name:latest`

#### 3.5. 배포 확인

1.  **EC2 퍼블릭 IP 또는 도메인으로 접속**:
    * 웹 브라우저에서 `http://<EC2_Public_IP>/docs` 또는 `http://<EC2_Public_IP>/api/v1/docs` (Nginx 설정에 따라 접두사가 필요할 수 있음)로 접속하여 FastAPI Swagger UI가 정상적으로 표시되는지 확인합니다.
2.  **로그 확인**:
    * Nginx 로그 (`sudo tail -f /var/log/nginx/access.log`)와 Docker 컨테이너 로그 (`docker logs fastapi_app`)를 확인하여 문제가 없는지 점검합니다.

---

### 다음 단계 (선택 사항 및 권장)

* **HTTPS 설정**: AWS ACM(Certificate Manager)으로 SSL 인증서를 발급받고 Nginx 또는 AWS ALB에 적용하여 안전한 HTTPS 통신을 구현합니다.
* **CI/CD 파이프라인 구축**: GitHub Actions, AWS CodePipeline/CodeBuild 등을 사용하여 코드 푸시부터 테스트, Docker 이미지 빌드, ECR 푸시, EC2 배포까지의 과정을 자동화하여 개발 및 배포 효율성을 극대화합니다.
* **모니터링**: AWS CloudWatch 등을 사용하여 EC2 인스턴스와 애플리케이션의 로그, 메트릭을 수집하고 모니터링하여 문제 발생 시 신속하게 대응할 수 있도록 합니다.
* **로드 밸런싱 및 오토 스케일링**: 트래픽이 많아질 경우 AWS ALB (Application Load Balancer)와 Auto Scaling Group을 사용하여 확장성을 확보합니다.
* **보안 강화**: EC2 보안 그룹 및 네트워크 ACL 규칙을 최소한으로 설정하고, IAM 역할을 세분화하여 최소 권한 원칙을 준수합니다.