# FastAPI 백엔드 서버 개발 및 AWS EC2 배포 가이드 (pyproject.toml 포함)

이 문서는 FastAPI, PostgreSQL (SQLAlchemy, Alembic), Docker를 활용하여 백엔드 서버를 개발하고 AWS EC2에 배포하는 과정을 `pyproject.toml`을 통한 패키지 관리를 포함하여 단계별로 설명합니다.

-----

## 1\. 프로젝트 구조

클린 아키텍처 원칙을 적용하여 모듈성과 확장성을 고려한 프로젝트 구조입니다. `src` 디렉토리를 사용하여 실제 Python 패키지들을 관리하는 "src 레이아웃"을 채택합니다.

```
your_project_name/
├── .uv/                        # UV 가상 환경 파일 (UV 관리)
├── .gitignore                  # Git 추적 제외 파일
├── pyproject.toml              # 프로젝트 메타데이터 및 의존성 목록 (PEP 621 표준)
├── alembic.ini                 # Alembic 전역 설정 (프로젝트 루트에 위치)
├── main.py                     # FastAPI 애플리케이션 진입점
├── src/                        # Python 패키지들을 담는 컨테이너 (중요!)
│   ├── app/                    # 주 애플리케이션 패키지
│   │   ├── __init__.py         # Python 모듈로 인식
│   │   ├── core/               # 핵심 설정 및 유틸리티
│   │   │   ├── config.py       # 환경 변수 및 애플리케이션 설정
│   │   │   ├── database.py     # DB 세션 및 엔진 초기화
│   │   │   └── security.py     # 인증/인가 관련 유틸리티 (JWT 등)
│   │   ├── api/                # API 엔드포인트
│   │   │   ├── v1/
│   │   │   │   ├── endpoints/
│   │   │   │   │   ├── users.py# 사용자 관련 API
│   │   │   │   │   └── items.py# 아이템 관련 API
│   │   │   │   └── routers.py  # v1 라우터 통합
│   │   ├── crud/               # DB CRUD 로직
│   │   │   ├── base.py         # 제네릭 CRUD 기본 클래스
│   │   │   ├── users.py        # 사용자 CRUD
│   │   │   └── items.py        # 아이템 CRUD
│   │   ├── models/             # SQLAlchemy ORM 모델 정의
│   │   │   ├── base.py         # 모든 모델의 공통 베이스
│   │   │   ├── user.py         # User 모델
│   │   │   └── item.py         # Item 모델
│   │   ├── schemas/            # Pydantic 모델 (요청/응답 유효성 검사)
│   │   │   ├── user.py         # User 스키마
│   │   │   └── item.py         # Item 스키마
│   │   └── dependencies/       # API 종속성 주입 (DB 세션, 현재 사용자 등)
│   │       └── common.py
│   └── migrations/             # Alembic 마이그레이션 파일 (src 아래에 위치!)
│       ├── versions/           # 마이그레이션 스크립트
│       ├── env.py              # Alembic 환경 설정
│       └── __init__.py         # Python 패키지로 인식시키기 위함 (필수)
├── tests/                      # 테스트 코드
│   └── ...
├── Dockerfile                  # Docker 이미지 빌드 파일
├── docker-compose.yml          # 개발 환경용 Docker Compose 파일
├── .env.example                # 환경 변수 예시
└── README.md                   # 프로젝트 설명
```

-----

## 2\. 프로젝트 개발 및 로컬 환경 설정 절차

### 2.1. 프로젝트 초기화 및 UV 설정

1.  **프로젝트 디렉토리 생성 및 이동:**

    ```bash
    mkdir consolidation-accounting-backend
    cd consolidation-accounting-backend
    ```

2.  **UV 설치:**
    시스템 Python 환경에 `uv`를 설치합니다.

    ```bash
    pip install uv
    ```

3.  **UV 가상 환경 생성 및 활성화:**
    `uv venv`를 실행한 후, 운영체제에 맞춰 가상 환경을 활성화합니다.

    ```bash
    uv venv
    ```

      * **macOS/Linux:**
        ```bash
        source ./.uv/bin/activate
        ```
      * **Windows (PowerShell):**
        ```powershell
        ./.uv/Scripts/Activate.ps1
        ```
      * **Windows (Command Prompt):**
        ```cmd
        .\.uv\Scripts\activate.bat
        ```

4.  `.gitignore` **파일 생성:**
    프로젝트 루트에 `.gitignore` 파일을 생성하고 다음 내용을 추가하여 Git에서 추적되지 않도록 합니다.

    ```gitignore
    # .gitignore
    .uv/
    __pycache__/
    .env
    *.pyc
    *.sqlite3
    .pytest_cache/
    .mypy_cache/
    *.log
    .DS_Store
    ```

### 2.2. `pyproject.toml` 작성

프로젝트 루트에 `pyproject.toml` 파일을 생성하고 프로젝트의 메타데이터와 의존성을 정의합니다.

```toml
# pyproject.toml
[project]
name = "consolidation-accounting-backend"
version = "0.1.0"
description = "FastAPI Backend for Consolidation Accounting"
readme = "README.md"
requires-python = ">=3.9" # 또는 사용하는 Python 버전
license = { file = "LICENSE" } # 선택 사항, 있다면 LICENSE 파일 생성

dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "sqlalchemy",
    "psycopg2-binary", # PostgreSQL 드라이버
    "alembic",
    "pydantic-settings",
]

[project.optional-dependencies]
dev = [
    "pytest",
    "httpx", # FastAPI 테스트 클라이언트
    "ruff", # 코드 린터 (black 대신 ruff format 사용 시)
    "black", # 코드 포매터 (ruff format을 사용하지 않는 경우)
]

[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"] # src/ 디렉토리 안에서 패키지를 찾도록 설정 (매우 중요!)

[tool.pytest.ini_options]
pythonpath = "src" # pytest가 src 디렉토리의 모듈을 찾도록 설정

[tool.ruff]
line-length = 120 # ruff 사용 시 (선택 사항)
target-version = "py39" # 또는 사용하는 Python 버전
select = ["E", "F", "I", "C", "N", "D"] # 린트 규칙 설정
ignore = ["D100", "D104"] # 특정 규칙 무시

[tool.ruff.format]
# ruff format을 사용한다면 여기에 설정
# 2024년 6월 현재 ruff format은 아직 실험적 기능이므로, black을 쓰는 것을 권장
```

### 2.3. 패키지 설치 (UV + `pyproject.toml`)

가상 환경이 활성화된 상태에서 의존성을 설치합니다.

1.  **기본 의존성 설치:**

    ```bash
    uv pip install -e .
    ```

      * `-e .`는 editable 모드로, 개발 중 코드 변경 시 바로 반영됩니다. `pyproject.toml`의 `[project].dependencies`에 정의된 패키지들을 설치합니다.

2.  **개발 의존성 설치 (선택 사항):**

    ```bash
    uv pip install -e ".[dev]"
    ```

      * `[project.optional-dependencies].dev`에 정의된 개발용 패키지들도 함께 설치합니다.

### 2.4. 프로젝트 기본 디렉토리 및 파일 생성

위에 제시된 프로젝트 구조에 맞춰 필요한 디렉토리와 빈 `__init__.py` 파일들을 생성합니다.

```bash
mkdir -p src/app/core src/app/api/v1/endpoints src/app/crud src/app/models src/app/schemas src/app/dependencies src/migrations
touch src/app/__init__.py
touch src/app/core/__init__.py
touch src/app/api/__init__.py
touch src/app/api/v1/__init__.py
touch src/app/api/v1/endpoints/__init__.py
touch src/app/crud/__init__.py
touch src/app/models/__init__.py
touch src/app/schemas/__init__.py
touch src/app/dependencies/__init__.py
touch src/migrations/__init__.py # migrations 폴더를 Python 패키지로 인식시키기 위함 (중요!)
touch main.py
```

### 2.5. `app/core/config.py` 작성

`pydantic-settings`를 활용하여 환경 변수를 관리하는 클래스를 정의합니다.

```python
# src/app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
import os # os 모듈 임포트 추가

class Settings(BaseSettings):
    # 데이터베이스 연결 URL (postgresql://user:password@host:port/dbname)
    DATABASE_URL: str

    # JWT 토큰 서명에 사용될 시크릿 키
    SECRET_KEY: str

    # JWT 토큰 암호화 알고리즘 (기본값 HS256)
    ALGORITHM: str = "HS256"

    # 액세스 토큰 만료 시간 (분 단위)
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # Pydantic Settings 설정
    # .env 파일에서 환경 변수를 로드하며, 찾을 수 없는 추가 필드는 무시
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

# Settings 인스턴스를 생성하여 애플리케이션 전역에서 사용
settings = Settings()
```

### 2.6. `app/core/database.py` 작성

SQLAlchemy 엔진, 세션 및 ORM `Base` 객체를 설정합니다.

```python
# src/app/core/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
from app.core.config import settings # config 모듈에서 settings 임포트

# 데이터베이스 연결 URL 설정
SQLALCHEMY_DATABASE_URL = settings.DATABASE_URL

# SQLAlchemy 엔진 생성
# echo=True는 SQL 쿼리를 로깅하여 디버깅에 유용합니다 (프로덕션에서는 False).
engine = create_engine(SQLALCHEMY_DATABASE_URL)

# 데이터베이스 세션 생성을 위한 sessionmaker 설정
# autocommit=False: 트랜잭션이 수동으로 커밋될 때까지 변경사항을 보류
# autoflush=False: 쿼리를 실행할 때마다 변경사항을 자동으로 플러시하지 않음
# bind=engine: 생성된 엔진에 세션 바인딩
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# SQLAlchemy ORM 모델의 기본 클래스 선언
# 모든 ORM 모델은 이 Base를 상속받아야 합니다.
Base = declarative_base()

# 의존성 주입을 위한 DB 세션 제너레이터 함수
# FastAPI의 Depends에 사용되어 각 요청마다 DB 세션을 제공하고,
# 요청이 완료되면 세션을 자동으로 닫습니다.
def get_db():
    db = SessionLocal()
    try:
        yield db # DB 세션을 제공
    finally:
        db.close() # 요청 완료 후 DB 세션 닫기
```

### 2.7. `app/models/` 디렉토리 및 ORM 모델 정의

`src/app/models/base.py`에서 `Base`를 임포트하여 SQLAlchemy ORM 모델의 기본 클래스로 사용합니다.

1.  **`src/app/models/base.py`:**

    ```python
    # src/app/models/base.py
    from app.core.database import Base # database.py에서 정의한 Base 객체를 임포트
    ```

2.  **`src/app/models/user.py` (예시):**

    ```python
    # src/app/models/user.py
    from sqlalchemy import Column, Integer, String
    from sqlalchemy.orm import relationship # 관계형 매핑을 위해 추가될 수 있음

    from app.models.base import Base # models/base.py에서 Base 임포트

    class User(Base):
        __tablename__ = "users" # 이 클래스가 매핑될 데이터베이스 테이블 이름

        id = Column(Integer, primary_key=True, index=True)
        email = Column(String, unique=True, index=True, nullable=False)
        hashed_password = Column(String, nullable=False)

        # 예를 들어, User와 Item이 1:N 관계라면 여기에 relationship 추가 가능
        # items = relationship("Item", back_populates="owner")
    ```

3.  **`src/app/models/item.py` (예시):**

    ```python
    # src/app/models/item.py
    from sqlalchemy import Column, Integer, String, ForeignKey
    from sqlalchemy.orm import relationship

    from app.models.base import Base
    # from app.models.user import User # User 모델이 필요하면 임포트 (아래 주석 처리된 관계 예시 사용 시)

    class Item(Base):
        __tablename__ = "items"

        id = Column(Integer, primary_key=True, index=True)
        title = Column(String, index=True, nullable=False)
        description = Column(String, nullable=True)
        # owner_id = Column(Integer, ForeignKey("users.id")) # 사용자 ID를 외래 키로 사용 (주석 해제 시)

        # owner = relationship("User", back_populates="items") # 관계형 매핑 (주석 해제 시)
    ```

### 2.8. Alembic 설정 및 마이그레이션

이 부분이 가장 중요하며, 앞에서 논의했던 `sys.path`와 `alembic.ini` 설정 변경이 포함됩니다.

1.  **Alembic 초기화:**
    **프로젝트 루트 디렉토리**에서 다음 명령어를 실행하여 `alembic.ini` 파일과 `src/migrations/` 디렉토리 구조를 생성합니다.

    ```bash
    alembic init src/migrations
    ```

      * 이 명령을 실행하면 `your_project_name/alembic.ini` 파일과 `your_project_name/src/migrations` 디렉토리(및 그 하위 `versions`, `env.py`)가 생성됩니다.

2.  **`alembic.ini` 수정:**
    프로젝트 루트에 생성된 `alembic.ini` 파일을 엽니다.

      * `sqlalchemy.url =` 부분을 주석 처리 (`#` 붙임)합니다. 이제 DB URL은 `.env` 파일을 통해 `env.py`에서 동적으로 로드됩니다.
      * `script_location =` 부분을 `src/migrations`로 변경합니다.

    <!-- end list -->

    ```ini
    # alembic.ini
    [alembic]
    # ... (다른 설정들) ...

    # Path to the revision scripts directory. Relative to this file.
    script_location = %(here)s/src/migrations # <-- 이 부분을 이렇게 변경합니다.

    # ... (다른 설정들) ...

    # A database URL or connection string.
    # Set to empty to use app.core.config.py's settings.DATABASE_URL
    # sqlalchemy.url =
    # Example: postgresql://user:password@localhost/dbname
    # <-- 이 부분을 주석 처리하거나 비워둡니다.

    # ... (다른 설정들) ...
    ```

3.  **`src/migrations/env.py` 수정:**
    `src/migrations/env.py` 파일을 열고 다음과 같이 수정합니다. (앞서 설명드린 내용 반영)

    ```python
    # src/migrations/env.py

    import os
    import sys
    from pathlib import Path

    from logging.config import fileConfig

    from sqlalchemy import engine_from_config
    from sqlalchemy import pool

    from alembic import context

    # ----------------------------------------------------------------------
    # 프로젝트 루트 디렉토리를 sys.path에 추가 (매우 중요!)
    # 현재 스크립트(env.py)의 절대 경로
    current_dir = Path(__file__).parent.resolve()
    # project_root = current_dir.parent.parent  # src/migrations -> src -> project_root
    # sys.path.insert(0, str(project_root))

    # NOTE: 만약 `alembic init .`으로 migrations 폴더가 루트에 생성되었다면 위 코드는 달라져야 함
    # 현재 가이드는 `alembic init src/migrations` 이므로 위 주석 처리된 코드가 맞지만
    # 더 일반적인 사용을 위해 `os.getcwd()`를 사용하고 `src`를 포함한 경로를 직접 추가하는 방식을 권장합니다.

    # 프로젝트의 'src' 디렉토리를 sys.path에 추가합니다.
    # 이렇게 하면 'src' 안의 'app' 모듈을 임포트할 수 있습니다.
    sys.path.insert(0, str(Path(os.getcwd()) / "src"))
    # ----------------------------------------------------------------------

    # 이제 app 모듈에서 필요한 것들을 임포트할 수 있습니다.
    from app.core.config import settings
    from app.models.base import Base # Base 객체 (declarative_base로 생성된) 임포트

    # this is the Alembic Config object, which provides
    # access to the values within the .ini file in use.
    config = context.config

    # Interpret the config file for Python logging.
    # This line sets up loggers basically.
    if config.config_file_name is not None:
        fileConfig(config.config_file_name)

    # add your model's MetaData object here
    # for 'autogenerate' support
    target_metadata = Base.metadata # <-- Base.metadata로 변경합니다.

    # A database URL or connection string.
    # from alembic.ini, passed into the 'url' argument of connect()
    # config.set_main_option("sqlalchemy.url", "sqlite:///./test.db") # 예시 주석 처리

    # app.core.config.py에서 설정된 DATABASE_URL 사용
    config.set_main_option("sqlalchemy.url", settings.DATABASE_URL) # <-- 이 줄 추가

    def run_migrations_offline() -> None:
        """Run migrations in 'offline' mode."""
        url = config.get_main_option("sqlalchemy.url")
        context.configure(
            url=url,
            target_metadata=target_metadata,
            literal_binds=True,
            dialect_opts={"paramstyle": "named"},
        )

        with context.begin_transaction():
            context.run_migrations()


    def run_migrations_online() -> None:
        """Run migrations in 'online' mode."""
        connectable = engine_from_config(
            config.get_section(config.config_ini_section, {}), # 빈 딕셔너리 기본값 추가
            prefix="sqlalchemy.",
            poolclass=pool.NullPool,
        )

        with connectable.connect() as connection:
            context.configure(
                connection=connection, target_metadata=target_metadata
            )

            with context.begin_transaction():
                context.run_migrations()


    if context.is_offline_mode():
        run_migrations_offline()
    else:
        run_migrations_online()
    ```

4.  **로컬 PostgreSQL 데이터베이스 실행 (Docker Compose 사용):**
    프로젝트 루트에 `docker-compose.yml` 파일을 생성합니다.

    ```yaml
    # docker-compose.yml
    version: '3.8'

    services:
      db:
        image: postgres:15-alpine # 가볍고 안정적인 최신 PostgreSQL 이미지
        restart: always           # 컨테이너 종료 시 항상 재시작
        environment:
          POSTGRES_DB: your_fastapi_db # 실제 사용할 데이터베이스 이름으로 변경하세요!
          POSTGRES_USER: your_db_user   # 실제 사용할 사용자 이름으로 변경하세요!
          POSTGRES_PASSWORD: your_db_password # 실제 사용할 비밀번호로 변경하세요!
        ports:
          - "5432:5432"           # 로컬 5432 포트를 컨테이너 5432 포트에 매핑
        volumes:
          - db_data:/var/lib/postgresql/data # 데이터 영속성을 위한 볼륨 (컨테이너 삭제 시에도 데이터 유지)

      web:
        build: .                  # 현재 디렉토리의 Dockerfile을 사용하여 이미지 빌드
        command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload # 개발 시 --reload 추가
        volumes:
          - .:/app                # 로컬 코드를 컨테이너의 /app 디렉토리에 마운트 (코드 변경 시 즉시 반영)
        ports:
          - "8000:8000"           # 로컬 8000 포트를 컨테이너 8000 포트에 매핑
        env_file:
          - .env                  # .env 파일에서 환경 변수 로드
        depends_on:
          - db                    # DB 컨테이너가 먼저 시작되도록 설정 (필수!)

    volumes:
      db_data: # db_data라는 이름의 볼륨 정의
    ```

    **`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`를 꼭 실제 사용할 값으로 변경하세요.**

    프로젝트 루트에서 다음 명령어를 실행하여 PostgreSQL 컨테이너를 시작합니다:

    ```bash
    docker-compose up -d db
    ```

5.  `.env` **파일 생성 (로컬 개발용):**
    프로젝트 루트에 `.env` 파일을 생성하고 다음 내용을 입력합니다.
    **이 파일의 내용은 `docker-compose.yml`의 `db` 서비스에 설정한 환경 변수와 정확히 일치해야 합니다\!**
    절대 Git에 올리지 마세요\! 반드시 `.gitignore`에 추가해야 합니다.

    ```ini
    # .env
    DATABASE_URL=postgresql://your_db_user:your_db_password@localhost:5432/your_fastapi_db
    SECRET_KEY=super_secret_key_for_jwt_development # 개발용 시크릿 키 (실제 배포 시에는 강력한 키로 변경)
    ```

      * `your_db_user`, `your_db_password`, `your_fastapi_db`는 `docker-compose.yml`에서 설정한 값과 동일해야 합니다.

6.  **첫 마이그레이션 생성:**
    UV 가상 환경이 활성화된 상태에서 (아직 활성화하지 않았다면 `source ./.uv/bin/activate` 등으로 활성화) 프로젝트 루트에서 다음 Alembic 명령어를 실행합니다.

    ```bash
    alembic revision --autogenerate -m "create initial tables"
    ```

    이 명령어는 `src/migrations/versions/` 폴더 안에 새로운 마이그레이션 스크립트(`xxxx_create_initial_tables.py`와 같은 이름)를 생성합니다.

7.  **마이그레이션 적용:**
    생성된 마이그레이션 스크립트를 실제 로컬 PostgreSQL 데이터베이스에 적용합니다.

    ```bash
    alembic upgrade head
    ```

    이 명령어는 데이터베이스에 `users`, `items` 테이블 등을 생성합니다.

### 2.9. `app/schemas/` 및 Pydantic 스키마 정의

API 요청 및 응답 데이터를 위한 Pydantic 모델을 정의합니다.

1.  **`src/app/schemas/user.py`:**

    ```python
    # src/app/schemas/user.py
    from pydantic import BaseModel, EmailStr

    class UserBase(BaseModel):
        email: EmailStr

    class UserCreate(UserBase):
        password: str

    class UserResponse(UserBase):
        id: int
        # is_active: bool # 필요하면 추가

        class Config:
            from_attributes = True # Pydantic v2: orm_mode 대신 from_attributes 사용
            # orm_mode = True # Pydantic v1: ORM 객체를 직접 변환할 수 있도록 설정
    ```

2.  **`src/app/schemas/item.py`:**

    ```python
    # src/app/schemas/item.py
    from pydantic import BaseModel

    class ItemBase(BaseModel):
        title: str
        description: str | None = None # description은 선택 사항

    class ItemCreate(ItemBase):
        pass

    class ItemResponse(ItemBase):
        id: int
        # owner_id: int # 외래키가 있다면 추가

        class Config:
            from_attributes = True # Pydantic v2: orm_mode 대신 from_attributes 사용
            # orm_mode = True # Pydantic v1
    ```

### 2.10. `app/crud/` 및 CRUD 로직 정의

데이터베이스와의 상호작용(Create, Read, Update, Delete) 로직을 정의합니다. `base.py`를 통해 제네릭 CRUD를 구현할 수 있습니다.

1.  **`src/app/crud/base.py`:**

    ```python
    # src/app/crud/base.py
    from typing import Any, Dict, Generic, List, Optional, Type, TypeVar, Union

    from fastapi.encoders import jsonable_encoder
    from pydantic import BaseModel
    from sqlalchemy.orm import Session

    from app.models.base import Base # Base 모델 임포트

    ModelType = TypeVar("ModelType", bound=Base)
    CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
    UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)

    class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
        def __init__(self, model: Type[ModelType]):
            """
            CRUD object with default methods to Create, Read, Update, Delete (CRUD).
            **Parameters**
            * `model`: A SQLAlchemy model class
            """
            self.model = model

        def get(self, db: Session, id: Any) -> Optional[ModelType]:
            return db.query(self.model).filter(self.model.id == id).first()

        def get_multi(
            self, db: Session, *, skip: int = 0, limit: int = 100
        ) -> List[ModelType]:
            return db.query(self.model).offset(skip).limit(limit).all()

        def create(self, db: Session, *, obj_in: CreateSchemaType) -> ModelType:
            obj_in_data = jsonable_encoder(obj_in)
            db_obj = self.model(**obj_in_data)  # type: ignore
            db.add(db_obj)
            db.commit()
            db.refresh(db_obj)
            return db_obj

        def update(
            self,
            db: Session,
            *,
            db_obj: ModelType,
            obj_in: Union[UpdateSchemaType, Dict[str, Any]]
        ) -> ModelType:
            obj_data = jsonable_encoder(db_obj)
            if isinstance(obj_in, dict):
                update_data = obj_in
            else:
                update_data = obj_in.model_dump(exclude_unset=True) # pydantic v2
                # update_data = obj_in.dict(exclude_unset=True) # pydantic v1
            for field in obj_data:
                if field in update_data:
                    setattr(db_obj, field, update_data[field])
            db.add(db_obj)
            db.commit()
            db.refresh(db_obj)
            return db_obj

        def remove(self, db: Session, *, id: int) -> Optional[ModelType]:
            obj = db.query(self.model).get(id)
            if obj:
                db.delete(obj)
                db.commit()
            return obj
    ```

2.  **`src/app/crud/user.py`:**

    ```python
    # src/app/crud/user.py
    from sqlalchemy.orm import Session
    from app.crud.base import CRUDBase
    from app.models.user import User
    from app.schemas.user import UserCreate, UserResponse # UserResponse는 예시, 실제 User 클래스와 맵핑되는 스키마 사용
    from app.core.security import get_password_hash # 비밀번호 해싱을 위한 유틸리티 임포트

    class CRUDUser(CRUDBase[User, UserCreate, UserResponse]): # UserResponse는 스키마 타입에 따라 변경
        def create_with_password(self, db: Session, *, obj_in: UserCreate) -> User:
            hashed_password = get_password_hash(obj_in.password)
            db_obj = User(
                email=obj_in.email,
                hashed_password=hashed_password
            )
            db.add(db_obj)
            db.commit()
            db.refresh(db_obj)
            return db_obj

        def get_by_email(self, db: Session, *, email: str) -> User | None:
            return db.query(User).filter(User.email == email).first()

    user = CRUDUser(User)
    ```

3.  **`src/app/crud/item.py`:**

    ```python
    # src/app/crud/item.py
    from sqlalchemy.orm import Session
    from app.crud.base import CRUDBase
    from app.models.item import Item
    from app.schemas.item import ItemCreate, ItemResponse

    class CRUDItem(CRUDBase[Item, ItemCreate, ItemResponse]):
        pass # 추가적인 Item CRUD 로직은 여기에 정의

    item = CRUDItem(Item)
    ```

### 2.11. `app/core/security.py` (인증/인가 관련 유틸리티)

JWT 토큰 생성 및 비밀번호 해싱 등을 처리하는 유틸리티를 정의합니다.

```python
# src/app/core/security.py
from datetime import datetime, timedelta
from typing import Any, Union

from jose import jwt, JWTError
from passlib.context import CryptContext

from app.core.config import settings # settings 임포트

# 비밀번호 해싱 컨텍스트 설정 (bcrypt 사용)
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """일반 비밀번호와 해싱된 비밀번호를 비교합니다."""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """비밀번호를 해싱합니다."""
    return pwd_context.hash(password)

def create_access_token(
    data: dict, expires_delta: timedelta | None = None
) -> str:
    """액세스 토큰을 생성합니다."""
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)
    return encoded_jwt

# 토큰 디코딩 및 검증 함수도 여기에 추가할 수 있습니다.
# 예를 들어, from app.dependencies.common import get_current_user 등에서 사용
```

### 2.12. `app/dependencies/common.py` (API 종속성 주입)

API 종속성 주입 (DB 세션, 현재 사용자 등)을 위한 함수를 정의합니다.

```python
# src/app/dependencies/common.py
from typing import Generator
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from jose import JWTError, jwt

from app.core.config import settings
from app.core.database import SessionLocal # SessionLocal 임포트
from app.crud.user import user as crud_user # user CRUD 인스턴스 임포트
from app.models.user import User
from app.schemas.token import TokenData # TokenData 스키마 정의 필요 (아래 추가)

# OAuth2PasswordBearer를 사용하여 토큰을 추출할 엔드포인트 정의
# (예: /api/v1/auth/token 에서 토큰을 얻는다고 가정)
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/users/login") # 로그인 엔드포인트에 맞춰 변경

def get_db() -> Generator:
    """DB 세션을 반환하는 제너레이터 (FastAPI Depends에 사용)."""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

async def get_current_user(
    db: Session = Depends(get_db), token: str = Depends(oauth2_scheme)
) -> User:
    """현재 로그인한 사용자를 가져옵니다 (JWT 토큰 검증)."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username) # TokenData 스키마 사용
    except JWTError:
        raise credentials_exception
    user = crud_user.get_by_email(db, email=token_data.username) # 이메일로 사용자 조회
    if user is None:
        raise credentials_exception
    return user

# JWT Payload를 위한 스키마 (필요 시)
# src/app/schemas/token.py 파일에 다음 내용 추가
# ----------------------------------------------------
# src/app/schemas/token.py
from pydantic import BaseModel

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: str | None = None
# ----------------------------------------------------
```

  * `src/app/schemas/token.py` 파일을 생성하고 위의 `Token`과 `TokenData` 스키마를 정의해야 합니다.

### 2.13. `app/api/v1/endpoints/` 및 API 엔드포인트 정의

각 리소스별 `APIRouter`를 사용하여 API 엔드포인트를 정의합니다.

1.  **`src/app/api/v1/endpoints/users.py`:**

    ```python
    # src/app/api/v1/endpoints/users.py
    from typing import Any, List
    from fastapi import APIRouter, Depends, HTTPException, status
    from fastapi.security import OAuth2PasswordRequestForm
    from sqlalchemy.orm import Session

    from app.core.config import settings
    from app.core.security import create_access_token, verify_password, get_password_hash
    from app.crud.user import user as crud_user # user CRUD 인스턴스 임포트
    from app.schemas.user import UserCreate, UserResponse # 스키마 임포트
    from app.schemas.token import Token # Token 스키마 임포트
    from app.dependencies.common import get_db, get_current_user # 종속성 임포트
    from app.models.user import User # 모델 임포트 (예시: get_current_user 반환 타입)

    router = APIRouter()

    @router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
    def register_user(
        *,
        db: Session = Depends(get_db),
        user_in: UserCreate,
    ) -> Any:
        """
        새로운 사용자를 등록합니다.
        """
        user = crud_user.get_by_email(db, email=user_in.email)
        if user:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="The user with this email already exists in the system.",
            )
        new_user = crud_user.create_with_password(db, obj_in=user_in)
        return new_user

    @router.post("/login", response_model=Token)
    def login_for_access_token(
        db: Session = Depends(get_db), form_data: OAuth2PasswordRequestForm = Depends()
    ) -> Any:
        """
        사용자 로그인 및 액세스 토큰 발급.
        """
        user = crud_user.get_by_email(db, email=form_data.username)
        if not user or not verify_password(form_data.password, user.hashed_password):
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Incorrect username or password",
                headers={"WWW-Authenticate": "Bearer"},
            )
        access_token_expires = timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
        access_token = create_access_token(
            data={"sub": user.email}, expires_delta=access_token_expires
        )
        return {"access_token": access_token, "token_type": "bearer"}

    @router.get("/me", response_model=UserResponse)
    def read_users_me(
        current_user: User = Depends(get_current_user),
    ) -> Any:
        """
        현재 로그인한 사용자 정보를 가져옵니다.
        """
        return current_user

    @router.get("/", response_model=List[UserResponse])
    def read_users(
        db: Session = Depends(get_db),
        skip: int = 0,
        limit: int = 100,
        current_user: User = Depends(get_current_user) # 인증 필요
    ) -> Any:
        """
        모든 사용자 목록을 가져옵니다 (관리자용).
        """
        users = crud_user.get_multi(db, skip=skip, limit=limit)
        return users
    ```

2.  **`src/app/api/v1/endpoints/items.py`:**

    ```python
    # src/app/api/v1/endpoints/items.py
    from typing import Any, List
    from fastapi import APIRouter, Depends, HTTPException, status
    from sqlalchemy.orm import Session

    from app.crud.item import item as crud_item # item CRUD 인스턴스 임포트
    from app.schemas.item import ItemCreate, ItemResponse # 스키마 임포트
    from app.dependencies.common import get_db, get_current_user # 종속성 임포트
    from app.models.user import User # User 모델 임포트 (get_current_user 반환 타입)

    router = APIRouter()

    @router.post("/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
    def create_item(
        *,
        db: Session = Depends(get_db),
        item_in: ItemCreate,
        current_user: User = Depends(get_current_user), # 인증된 사용자만 생성 가능
    ) -> Any:
        """
        새로운 아이템을 생성합니다.
        """
        # 여기서는 owner_id를 사용하지 않는 간단한 예시
        # 만약 Item에 owner_id가 있다면 item_in에 추가하거나 여기서 할당
        # item_data = item_in.model_dump() # pydantic v2
        # item_data["owner_id"] = current_user.id
        item = crud_item.create(db, obj_in=item_in)
        return item

    @router.get("/", response_model=List[ItemResponse])
    def read_items(
        db: Session = Depends(get_db),
        skip: int = 0,
        limit: int = 100,
        current_user: User = Depends(get_current_user), # 인증 필요
    ) -> Any:
        """
        모든 아이템 목록을 가져옵니다.
        """
        items = crud_item.get_multi(db, skip=skip, limit=limit)
        return items

    @router.get("/{item_id}", response_model=ItemResponse)
    def read_item(
        *,
        db: Session = Depends(get_db),
        item_id: int,
        current_user: User = Depends(get_current_user) # 인증 필요
    ) -> Any:
        """
        특정 아이템을 ID로 가져옵니다.
        """
        item = crud_item.get(db, id=item_id)
        if not item:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Item not found")
        return item
    ```

### 2.14. `app/api/v1/routers.py`에서 라우터 통합

모든 `APIRouter`들을 한곳에 모아 `main.py`에서 `app.include_router`로 한 번에 등록할 수 있도록 합니다.

```python
# src/app/api/v1/routers.py
from fastapi import APIRouter
from app.api.v1.endpoints import users, items

api_router = APIRouter()

# 사용자 관련 API 라우터 포함
api_router.include_router(users.router, prefix="/users", tags=["users"])

# 아이템 관련 API 라우터 포함
api_router.include_router(items.router, prefix="/items", tags=["items"])

# 다른 라우터가 있다면 여기에 추가합니다.
```

### 2.15. `main.py` 작성

FastAPI 애플리케이션의 메인 진입점입니다.

```python
# main.py
from fastapi import FastAPI
from app.api.v1.routers import api_router

# FastAPI 애플리케이션 인스턴스 생성
app = FastAPI(
    title="Consolidation Accounting Backend API",
    description="A sample FastAPI project with Docker, SQLAlchemy, Alembic, and Pydantic-Settings.",
    version="1.0.0",
    docs_url="/docs",       # Swagger UI 경로
    redoc_url="/redoc",     # ReDoc UI 경로
    openapi_url="/openapi.json", # OpenAPI 스펙 JSON 경로
)

# /api/v1 접두사를 사용하여 API 라우터를 포함
app.include_router(api_router, prefix="/api/v1")

# 애플리케이션 시작/종료 이벤트 처리 (선택 사항)
@app.on_event("startup")
async def startup_event():
    print("Application startup: Database connection established, services initialized.")
    # 추가적인 초기화 로직 (예: 캐시 워밍업 등)

@app.on_event("shutdown")
async def shutdown_event():
    print("Application shutdown: Database connection closed, resources released.")
    # 추가적인 종료 로직

# 로컬 개발 환경에서 직접 실행 시 Uvicorn 서버 시작
if __name__ == "__main__":
    import uvicorn
    # --reload 옵션은 개발 중에 코드 변경 사항을 자동으로 감지하여 서버를 재시작합니다.
    uvicorn.run(app, host="0.0.0.0", port=8000, reload=True)
```

### 2.16. 로컬에서 Docker로 실행

DB 컨테이너와 앱 컨테이너를 함께 시작합니다.

1.  **Docker Compose로 모든 컨테이너 실행:**
    프로젝트 루트 디렉토리에서 다음 명령어를 사용합니다.

    ```bash
    docker-compose up --build
    # 또는 백그라운드에서 실행하려면:
    # docker-compose up -d --build
    ```

      * `--build`는 Dockerfile이 변경되었거나 이미지가 없는 경우 이미지를 새로 빌드합니다.

2.  **FastAPI Swagger UI 확인:**
    이제 웹 브라우저에서 `http://localhost:8000/docs`로 접속하여 FastAPI Swagger UI를 확인할 수 있습니다.

-----

## 3\. AWS EC2 배포 절차

이제 준비된 Docker 이미지를 AWS EC2에 배포하는 절차입니다.

### 3.1. AWS 사전 준비

1.  **AWS 계정 생성 및 로그인:**
    아직 AWS 계정이 없다면, [AWS 웹사이트](https://aws.amazon.com/)에서 계정을 생성하고 로그인합니다.

2.  **AWS RDS (Relational Database Service) for PostgreSQL 인스턴스 생성:**

      * **DB 엔진:** `PostgreSQL`을 선택합니다.
      * **버전 및 템플릿:** 적절한 버전과 개발/테스트 또는 프로덕션 템플릿을 선택합니다.
      * **DB 인스턴스 식별자, 마스터 사용자 이름, 마스터 암호**를 설정합니다. 이 정보는 `DATABASE_URL`에 사용될 것이므로 **반드시 기록해 두세요.** (예: `db-instance-1`, `rds_user`, `rds_password`)
      * **VPC 보안 그룹 생성:**
          * RDS 인스턴스를 위한 새로운 보안 그룹을 생성합니다. (예: `rds-security-group`)
          * **인바운드 규칙**에 **EC2 인스턴스의 보안 그룹 ID**를 소스로 추가하고 포트는 `5432`로 설정합니다. 이렇게 하면 EC2 인스턴스만 RDS에 접근할 수 있게 되어 보안을 강화합니다.
      * **퍼블릭 액세스:** 보안상 `'아니요'`로 설정하는 것이 좋습니다. 이 경우 EC2 인스턴스가 RDS와 **같은 VPC 내**에 있어야 합니다.
      * 생성된 RDS 인스턴스의 **엔드포인트**를 기록해 둡니다. 이 엔드포인트는 `DATABASE_URL`에 사용될 호스트 주소가 됩니다. (예: `mydbinstance.abcdefghij.ap-northeast-2.rds.amazonaws.com`)

3.  **AWS ECR (Elastic Container Registry) 리포지토리 생성:**
    Docker Hub 대신 AWS 내부 레지스트리인 ECR을 사용하는 것이 보안 및 성능 면에서 이점이 있습니다.

      * ECR 콘솔에서 `리포지토리 생성`을 클릭합니다.
      * 리포지토리 이름을 지정하고 생성합니다. (예: `fastapi-backend-repo`)
      * 생성 후 리포지토리 페이지에서 `푸시 명령 보기`를 클릭하여 나오는 **로그인/푸시 명령어를 기록해 둡니다.**

4.  **AWS Systems Manager Parameter Store 또는 Secrets Manager 설정:**
    데이터베이스 연결 정보, 시크릿 키 등 민감한 정보를 안전하게 저장하고 관리합니다. **Parameter Store `SecureString` 타입 사용을 권장합니다.**

      * **Parameter Store (권장):**

          * AWS Systems Manager 콘솔로 이동하여 `Parameter Store`를 선택합니다.
          * `파라미터 생성`을 클릭합니다.
          * **이름:** `/your-project-name/database-url`, `/your-project-name/secret-key` 등 의미 있는 이름으로 생성합니다. (예: `/prod/fastapi-backend/database-url`, `/prod/fastapi-backend/secret-key`)
          * **유형:** `SecureString`을 선택하고, KMS 키를 기본값으로 둡니다.
          * **값:** RDS 엔드포인트, 사용자 이름, 비밀번호를 포함하는 전체 `DATABASE_URL`과 `SECRET_KEY` 값을 입력합니다.
              * `DATABASE_URL` 예시: `postgresql://rds_user:rds_password@mydbinstance.abcdefghij.ap-northeast-2.rds.amazonaws.com:5432/your_fastapi_db`
              * `SECRET_KEY` 예시: `YOUR_HIGHLY_SECURE_PRODUCTION_SECRET_KEY`

      * **EC2 인스턴스에 IAM Role 부여:**

          * Parameter Store 또는 Secrets Manager에 접근할 수 있는 **IAM Role**을 생성합니다. (예: `FastAPIRole`)
          * 이 역할에 다음 권한을 부여합니다: `AmazonEC2ContainerRegistryReadOnly` (ECR 이미지 Pull용), `AmazonSSMReadOnlyAccess` (Parameter Store 읽기용, 또는 Secrets Manager Read용)
          * 이 역할은 이후 EC2 인스턴스 시작 시 인스턴스 프로필로 할당됩니다.

### 3.2. Docker 이미지 빌드 및 푸시

FastAPI 프로젝트의 최신 코드를 ECR로 푸시합니다.

1.  **FastAPI 프로젝트의 최신 코드 확인:** 로컬에서 작업한 코드가 모두 Git에 커밋되었는지 확인합니다.

2.  **Docker 이미지 빌드:** 프로젝트 루트 디렉토리에서 다음 명령어를 실행합니다.

    ```bash
    docker build -t consolidation-accounting-backend:latest .
    ```

      * `-t`: 이미지 이름과 태그를 지정합니다. `your-project-name`은 실제 프로젝트 이름으로 변경하세요.

3.  **ECR 레지스트리에 로그인 (터미널에서):**
    AWS 콘솔의 ECR 리포지토리 페이지에서 "푸시 명령 보기"를 클릭하여 나오는 로그인 명령어를 복사하여 실행합니다. 일반적으로 다음과 같은 형식입니다.

    ```bash
    aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.<your-region>.amazonaws.com
    ```

      * `<your-region>` (예: `ap-northeast-2`), `<your-account-id>`는 본인의 AWS 계정 정보로 변경해야 합니다. `aws-cli`가 설치되어 있어야 합니다.

4.  **이미지 태그 지정:** 빌드한 로컬 이미지를 ECR 리포지토리 URL로 태그합니다.

    ```bash
    docker tag consolidation-accounting-backend:latest <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/fastapi-backend-repo:latest
    ```

5.  **ECR로 이미지 푸시:**

    ```bash
    docker push <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/fastapi-backend-repo:latest
    ```

### 3.3. AWS EC2 인스턴스 설정

Docker 이미지를 실행할 EC2 인스턴스를 설정합니다.

1.  **EC2 인스턴스 시작:**

      * **AMI (Amazon Machine Image):** `Amazon Linux 2023` 또는 `Ubuntu LTS` 버전을 권장합니다.
      * **인스턴스 유형:** `t2.micro` (프리티어) 또는 적절한 인스턴스 유형을 선택합니다.
      * **키 페어:** SSH 접속을 위한 키 페어를 생성하거나 기존 키 페어를 사용합니다. `.pem` 파일을 안전하게 보관하세요.
      * **네트워크 설정:**
          * **VPC:** RDS와 동일한 VPC 또는 기본 VPC를 사용합니다.
          * **서브넷:** 공용 서브넷 (인터넷 접근 가능)을 선택합니다.
          * **퍼블릭 IP 자동 할당:** `'활성화'`로 설정하여 SSH 및 웹 접근이 가능하도록 합니다.
      * **보안 그룹 생성:**
          * 새로운 보안 그룹을 생성합니다. (예: `fastapi-ec2-sg`)
          * **SSH (Port 22):** 자신의 IP 주소에서만 접근을 허용합니다 (권장).
          * **HTTP (Port 80):** `0.0.0.0/0` (모든 IP)를 허용합니다. (Nginx를 리버스 프록시로 사용할 예정)
          * **HTTPS (Port 443):** `0.0.0.0/0` (모든 IP)를 허용합니다 (Nginx 사용 시).
          * **FastAPI 포트 (Port 8000):** 외부에서 8000번 포트로 직접 접근하는 것은 권장하지 않습니다. **이 포트는 Nginx가 실행되는 EC2 인스턴스의 보안 그룹 ID만 허용하도록 설정합니다.** 즉, 자기 자신의 보안 그룹(`fastapi-ec2-sg`의 보안 그룹 ID)에서 8000번 포트 인바운드를 허용합니다. (`Source`를 Custom으로 하고 `sg-xxxxxxxxxxxxxxxxx` 선택)
      * **스토리지:** 기본값으로 충분합니다.
      * **고급 세부 정보 (IAM 인스턴스 프로필):** 위에서 생성한 **IAM 역할 (`FastAPIRole`)을 인스턴스에 할당**합니다. 이는 EC2 인스턴스가 ECR에서 이미지를 당겨오고, Parameter Store/Secrets Manager에서 파라미터를 읽어올 수 있도록 권한을 부여합니다.
      * 인스턴스를 시작합니다.

2.  **EC2 인스턴스 접속:**
    SSH 클라이언트(예: PuTTY, Git Bash, macOS/Linux 터미널)를 사용하여 접속합니다. `<your-ec2-public-ip-or-dns>`는 EC2 인스턴스의 퍼블릭 IP 또는 퍼블릭 DNS를 의미합니다.

    ```bash
    ssh -i /path/to/your-key.pem ec2-user@<your-ec2-public-ip-or-dns>
    # 예시: ssh -i my-fastapi-key.pem ec2-user@ec2-13-125-XXX-XXX.ap-northeast-2.compute.amazonaws.com
    ```

3.  **Docker 설치 및 설정 (EC2 인스턴스 내부에서):**
    EC2 인스턴스에 접속한 후, Docker를 설치하고 사용자가 Docker 명령어를 `sudo` 없이 실행할 수 있도록 설정합니다.

      * **Amazon Linux 2023 (권장):**
        ```bash
        sudo yum update -y
        sudo yum install docker -y
        sudo service docker start
        sudo usermod -a -G docker ec2-user # ec2-user가 docker 명령어를 sudo 없이 실행하도록
        newgrp docker # 그룹 변경 즉시 적용 (세션을 끊고 다시 접속해도 됨)
        ```
      * **Ubuntu:**
        ```bash
        sudo apt update
        sudo apt install docker.io -y
        sudo systemctl start docker
        sudo systemctl enable docker
        sudo usermod -aG docker $USER # 현재 사용자가 docker 명령어를 sudo 없이 실행하도록
        newgrp docker # 그룹 변경 즉시 적용 (세션을 끊고 다시 접속해도 됨)
        ```

    Docker 설치 확인: `docker run hello-world`

4.  **Nginx 설치 및 설정 (리버스 프록시):**
    Nginx를 설치하고 (클라이언트의 80/443 포트 요청을 FastAPI 앱의 8000 포트로 전달) 설정 파일을 수정합니다.

      * **설치:**

          * **Amazon Linux 2023:** `sudo yum install nginx -y`
          * **Ubuntu:** `sudo apt install nginx -y`

      * **Nginx 설정 파일 수정:**

        ```bash
        sudo nano /etc/nginx/nginx.conf
        # 또는 Ubuntu에서는 sudo nano /etc/nginx/sites-available/default
        # 그리고 심볼릭 링크 sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
        ```

        `http` 블록 안에 `server` 블록을 추가하거나 수정합니다. 기존 `server` 블록이 있다면 적절히 수정합니다.

        ```nginx
        # /etc/nginx/nginx.conf (또는 /etc/nginx/conf.d/default.conf)
        # http {
        #     ...
        server {
            listen 80; # HTTP 요청을 80번 포트에서 수신
            server_name your_domain_name_or_ec2_public_ip; # 도메인 연결 시 도메인명, 아니면 EC2 퍼블릭 IP 사용 (예: 13.125.XXX.XXX)

            location / {
                proxy_pass http://127.0.0.1:8000; # FastAPI 컨테이너가 8000번 포트에서 실행 중이라고 가정
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                client_max_body_size 100M; # 파일 업로드 등 큰 요청 처리를 위한 설정 (선택 사항)
            }

            # /docs 경로를 위한 설정 (선택 사항, 필요시)
            location /docs {
                proxy_pass http://127.0.0.1:8000/docs;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }

            # /redoc 경로를 위한 설정 (선택 사항, 필요시)
            location /redoc {
                proxy_pass http://127.0.0.1:8000/redoc;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
        # }
        ```

          * `server_name`은 EC2 인스턴스의 퍼블릭 IP 주소 또는 연결된 도메인 이름을 입력합니다.
          * `proxy_pass`는 Docker 컨테이너 내부의 FastAPI 앱이 실행되는 주소와 포트입니다.

      * **Nginx 구문 검사 및 시작/자동 시작 설정:**

        ```bash
        sudo nginx -t # 설정 파일 문법 검사 (반드시!)
        sudo systemctl start nginx
        sudo systemctl enable nginx
        sudo systemctl status nginx # 상태 확인
        ```

      * **방화벽 설정 (Ubuntu의 경우):**

        ```bash
        sudo ufw allow 'Nginx HTTP'
        sudo ufw enable
        ```

### 3.4. Docker 컨테이너 실행

EC2 인스턴스에 SSH로 접속한 상태에서 다음 단계를 수행합니다.

1.  **ECR 레지스트리에 로그인 (EC2 인스턴스 내부에서):**

    ```bash
    aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.<your-region>.amazonaws.com
    ```

    이전에 로컬에서 사용했던 명령과 동일하지만, **EC2 인스턴스 내부에서 실행**해야 합니다.

2.  **환경 변수 설정 (Parameter Store 사용, 권장):**
    애플리케이션 시작 스크립트에서 AWS CLI/SDK (boto3 라이브러리 사용)를 사용하여 환경 변수를 동적으로 읽어오도록 설정하는 것이 가장 안전하고 권장되는 방법입니다.

      * **예시 스크립트 (`start_app.sh`):** EC2 인스턴스에 `start_app.sh` 파일을 생성합니다.

        ```bash
        #!/bin/bash

        # 시스템 관리자 파라미터 스토어에서 환경 변수 가져오기
        # YOUR_PROJECT_NAME을 실제 프로젝트 이름으로 변경하세요 (예: fastapi-backend)
        DATABASE_URL=$(aws ssm get-parameter --name "/prod/YOUR_PROJECT_NAME/database-url" --with-decryption --query Parameter.Value --output text)
        SECRET_KEY=$(aws ssm get-parameter --name "/prod/YOUR_PROJECT_NAME/secret-key" --with-decryption --query Parameter.Value --output text)

        # Docker 컨테이너 실행
        docker run -d \
          --name consolidation-accounting-backend \
          -p 8000:8000 \
          --restart always \
          -e DATABASE_URL="$DATABASE_URL" \
          -e SECRET_KEY="$SECRET_KEY" \
          <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/fastapi-backend-repo:latest

        # Docker 로그 확인 (선택 사항)
        docker logs -f consolidation-accounting-backend
        ```

      * `YOUR_PROJECT_NAME`을 실제 Parameter Store 파라미터 이름 접두사와 일치하도록 변경합니다.

      * 스크립트에 실행 권한 부여: `chmod +x start_app.sh`

      * 스크립트 실행: `./start_app.sh`

      * **Docker `run` 명령으로 직접 주입 시 (간단한 경우, 보안 취약):**
        이 방법은 개발/테스트 환경에서만 사용하고 프로덕션에서는 권장하지 않습니다.

        ```bash
        docker run -d --name consolidation-accounting-backend -p 8000:8000 --restart always \
          -e DATABASE_URL="postgresql://<rds_user>:<rds_password>@<rds-endpoint>:5432/<your_fastapi_db>" \
          -e SECRET_KEY="<your_jwt_secret_key>" \
          <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/fastapi-backend-repo:latest
        ```

3.  **Alembic 마이그레이션 적용 (중요\!):**
    데이터베이스 스키마 변경 시 마이그레이션을 적용해야 합니다. 애플리케이션 컨테이너를 실행하기 전에 마이그레이션을 별도의 명령으로 실행하는 것이 안전합니다.

    ```bash
    # 마이그레이션 전용 컨테이너를 일회성으로 실행합니다.
    # Parameter Store에서 환경 변수를 가져와서 주입합니다.
    DATABASE_URL=$(aws ssm get-parameter --name "/prod/YOUR_PROJECT_NAME/database-url" --with-decryption --query Parameter.Value --output text)

    docker run --rm \
      -e DATABASE_URL="$DATABASE_URL" \
      <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/fastapi-backend-repo:latest \
      alembic upgrade head
    ```

      * `--rm`: 컨테이너 종료 시 자동으로 삭제됩니다.
      * 마이그레이션이 완료되면 컨테이너는 자동으로 종료됩니다.

4.  **FastAPI Docker 컨테이너 실행:**
    `start_app.sh` 스크립트를 실행하거나, 직접 `docker run` 명령어를 실행합니다.

    ```bash
    # start_app.sh 스크립트 실행
    ./start_app.sh
    ```

    컨테이너가 백그라운드에서 실행되도록 설정되어 있으므로, 터미널을 닫아도 계속 실행됩니다.

### 3.5. 배포 확인

1.  **EC2 퍼블릭 IP 또는 도메인으로 접속:**
    웹 브라우저에서 `http://<EC2_Public_IP>/docs` 또는 `http://<EC2_Public_IP>/api/v1/docs` (Nginx 설정에 따라 접두사가 필요할 수 있음)로 접속하여 FastAPI Swagger UI가 정상적으로 표시되는지 확인합니다.

2.  **로그 확인:**

      * **Nginx 로그:** `sudo tail -f /var/log/nginx/access.log` 및 `sudo tail -f /var/log/nginx/error.log`를 확인하여 웹 서버에 문제가 없는지 점검합니다.
      * **Docker 컨테이너 로그:** `docker logs -f consolidation-accounting-backend`를 확인하여 FastAPI 애플리케이션에 오류가 없는지 점검합니다.

-----

## 4\. 다음 단계 (선택 사항 및 권장)

  * **HTTPS 설정:** AWS ACM(Certificate Manager)으로 SSL 인증서를 발급받고 Nginx 또는 AWS ALB에 적용하여 안전한 HTTPS 통신을 구현합니다.
  * **CI/CD 파이프라인 구축:** GitHub Actions, AWS CodePipeline/CodeBuild 등을 사용하여 코드 푸시부터 테스트, Docker 이미지 빌드, ECR 푸시, EC2 배포까지의 과정을 자동화하여 개발 및 배포 효율성을 극대화합니다.
  * **모니터링:** AWS CloudWatch 등을 사용하여 EC2 인스턴스와 애플리케이션의 로그, 메트릭을 수집하고 모니터링하여 문제 발생 시 신속하게 대응할 수 있도록 합니다.
  * **로드 밸런싱 및 오토 스케일링:** 트래픽이 많아질 경우 AWS ALB (Application Load Balancer)와 Auto Scaling Group을 사용하여 확장성을 확보합니다.
  * **보안 강화:** EC2 보안 그룹 및 네트워크 ACL 규칙을 최소한으로 설정하고, IAM 역할을 세분화하여 최소 권한 원칙을 준수합니다.
