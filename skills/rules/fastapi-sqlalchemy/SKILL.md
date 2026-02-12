---
name: fastapi-sqlalchemy
description: FastAPI + SQLAlchemy 2.0 async 구현 패턴. Clean Architecture에서 FastAPI와 SQLAlchemy를 사용할 때의 구체적 구현 규칙.
---

# FastAPI + SQLAlchemy 구현 규칙

## 기술 스택

- Python 3.11+
- FastAPI 0.128+
- SQLAlchemy 2.0+ (async)
- Pydantic v2
- uv (패키지 매니저)

## Classical Mapping (핵심)

도메인 엔티티를 ORM에서 분리하기 위해 Classical Mapping을 사용한다.

```python
# infrastructure/persistence/mapping.py
from sqlalchemy import Table, Column, String, Integer, MetaData
from sqlalchemy.orm import registry

mapper_registry = registry()
metadata = mapper_registry.metadata

# 테이블 정의
fortune_table = Table(
    "fortunes",
    metadata,
    Column("id", String, primary_key=True),
    Column("user_name", String(50), nullable=False),
    Column("birth_date", DateTime, nullable=False),
    Column("overall_fortune", String(500)),
)

# 도메인 엔티티에 매핑 (엔티티는 순수 Python 유지)
def start_mappers():
    mapper_registry.map_imperatively(Fortune, fortune_table)
```

### 관계(Relationship) 매핑

#### 1:N 관계

```python
from sqlalchemy import Table, Column, String, Integer, ForeignKey
from sqlalchemy.orm import relationship

order_table = Table(
    "orders", metadata,
    Column("id", String, primary_key=True),
    Column("customer_id", String, nullable=False),
)

order_item_table = Table(
    "order_items", metadata,
    Column("id", String, primary_key=True),
    Column("order_id", String, ForeignKey("orders.id"), nullable=False),
    Column("product_name", String, nullable=False),
    Column("quantity", Integer, nullable=False),
)

def start_mappers():
    mapper_registry.map_imperatively(Order, order_table, properties={
        "_items": relationship(OrderItem, lazy="joined"),  # 즉시 로딩
    })
    mapper_registry.map_imperatively(OrderItem, order_item_table)
```

#### N:M 관계

```python
# 연결 테이블(association table)
order_tag_table = Table(
    "order_tags", metadata,
    Column("order_id", String, ForeignKey("orders.id"), primary_key=True),
    Column("tag_id", String, ForeignKey("tags.id"), primary_key=True),
)

def start_mappers():
    mapper_registry.map_imperatively(Order, order_table, properties={
        "_tags": relationship(Tag, secondary=order_tag_table, lazy="joined"),
    })
```

#### async lazy loading 주의사항
- async 세션에서 `lazy="select"` (기본값)를 사용하면 **implicit I/O 오류**가 발생한다
- 반드시 `lazy="joined"`, `lazy="subquery"`, 또는 명시적 `selectinload()`를 사용한다
- 컬렉션이 크면 `lazy="selectin"`이 `lazy="joined"`보다 효율적이다

### 매핑 초기화

```python
# main.py (앱 시작 시 한 번만 실행)
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    start_mappers()      # ORM 매핑 초기화
    await create_tables() # 테이블 생성 (개발용)
    yield
    await dispose_engine()

app = FastAPI(lifespan=lifespan)
```

### 중복 매핑 방지

`start_mappers()`가 여러 번 호출되면 `ArgumentError`가 발생한다. 가드 패턴으로 방지한다.

```python
# infrastructure/persistence/mapping.py
_mappers_initialized = False

def start_mappers():
    global _mappers_initialized
    if _mappers_initialized:
        return
    mapper_registry.map_imperatively(Fortune, fortune_table)
    # ... 추가 매핑
    _mappers_initialized = True
```

- 테스트에서 `lifespan` 없이 앱을 생성할 때 중복 호출이 발생하기 쉽다
- `conftest.py`에서도 `start_mappers()`를 호출하는 경우 반드시 이 가드를 적용한다

## 데이터베이스 설정

```python
# infrastructure/persistence/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=False,             # 프로덕션에서는 False
    pool_size=10,
    max_overflow=20,
)

async_session_factory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # async에서는 반드시 False
)

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

## Repository 구현

```python
# infrastructure/persistence/repository/fortune_repository_impl.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

class SqlAlchemyFortuneRepository(FortuneRepository):
    """domain의 FortuneRepository ABC를 구현"""

    def __init__(self, session: AsyncSession):
        self._session = session

    async def find_by_id(self, fortune_id: str) -> Fortune | None:
        result = await self._session.execute(
            select(Fortune).where(Fortune.id == fortune_id)
        )
        return result.scalar_one_or_none()

    async def save(self, fortune: Fortune) -> None:
        self._session.add(fortune)
        await self._session.flush()  # commit은 세션 관리자가 담당
```

## 의존성 주입 (DI)

```python
# config/dependencies.py
from fastapi import Depends

async def get_fortune_repository(
    session: AsyncSession = Depends(get_session),
) -> FortuneRepository:
    return SqlAlchemyFortuneRepository(session)

async def get_create_fortune_use_case(
    repo: FortuneRepository = Depends(get_fortune_repository),
) -> CreateFortuneUseCase:
    return CreateFortuneUseCase(fortune_repo=repo)
```

## Presentation 레이어

### Router

```python
# presentation/{aggregate}/router.py
from fastapi import APIRouter, Depends, status

router = APIRouter(prefix="/fortunes", tags=["fortunes"])

@router.post("", status_code=status.HTTP_201_CREATED, response_model=FortuneResponseSchema)
async def create_fortune(
    request: CreateFortuneRequestSchema,
    use_case: CreateFortuneUseCase = Depends(get_create_fortune_use_case),
):
    command = CreateFortuneCommand(
        user_name=request.user_name,
        birth_date=request.birth_date,
    )
    result = await use_case.execute(command)
    return FortuneResponseSchema.from_dto(result)
```

### Pydantic Schema (Request/Response)

```python
# presentation/{aggregate}/schema.py
from pydantic import BaseModel, Field

class CreateFortuneRequestSchema(BaseModel):
    """API 요청 스키마 (presentation 레이어)"""
    user_name: str = Field(..., min_length=1, max_length=50)
    birth_date: datetime = Field(...)

class FortuneResponseSchema(BaseModel):
    """API 응답 스키마"""
    id: str
    user_name: str
    overall_fortune: str

    @classmethod
    def from_dto(cls, dto: FortuneResponse) -> "FortuneResponseSchema":
        return cls(id=dto.id, user_name=dto.user_name, overall_fortune=dto.overall_fortune)
```

### DTO vs Schema 구분

| 구분 | 위치 | 역할 | 의존성 |
|------|------|------|--------|
| **Schema** (Pydantic) | presentation | HTTP 요청/응답 직렬화, 입력 검증 | Pydantic |
| **DTO** (dataclass) | application | 레이어 간 데이터 전달 | 없음 (순수 Python) |

```python
# application/{aggregate}/dto/fortune_dto.py
from dataclasses import dataclass

@dataclass(frozen=True)
class CreateFortuneCommand:
    user_name: str
    birth_date: datetime

@dataclass(frozen=True)
class FortuneResponse:
    id: str
    user_name: str
    overall_fortune: str

    @classmethod
    def from_entity(cls, entity: Fortune) -> "FortuneResponse":
        return cls(id=entity.id, user_name=entity.user_name, overall_fortune=entity.overall_fortune)
```

## 필수 규칙

### 반드시 지킬 것
- 모든 I/O 작업은 async 사용
- `expire_on_commit=False` 설정 (async 필수)
- Repository에서 `flush()` 사용, `commit()`은 세션 관리자가 담당
- Pydantic Schema는 presentation에만, DTO는 application에만
- `Field()`로 입력 검증 (min_length, gt, regex 등)
- 적절한 HTTP 상태 코드 (201 Created, 204 No Content 등)
- Depends()로 의존성 주입 (직접 인스턴스 생성 금지)

### 절대 금지
- async 라우트에서 동기 블로킹 호출 (`time.sleep()`, 동기 DB 호출)
- Router에 비즈니스 로직 작성 (UseCase를 통해서만)
- 시크릿 하드코딩 (환경 변수 또는 설정 파일 사용)
- CORS에 `allow_origins=["*"]` (프로덕션에서 특정 origin만)
- Pydantic 검증 생략 (모든 외부 입력은 Schema로 검증)
- Entity를 API 응답으로 직접 반환 (DTO → Schema 변환 필수)

## 알려진 함정

### 1. BackgroundTasks와 Response 충돌

```python
# 주입된 BackgroundTasks와 Response의 background가 충돌
# 한 가지 메커니즘만 사용할 것

# ✅ BackgroundTasks만 사용
@router.post("")
async def endpoint(tasks: BackgroundTasks):
    tasks.add_task(send_notification)
    tasks.add_task(log_event)
    return {"status": "done"}
```

### 2. from __future__ import annotations 주의

```python
# OpenAPI 스키마가 깨질 수 있음
# FastAPI 라우터 파일에서는 사용하지 않는 것을 권장
```

### 3. field_validator의 ValueError → 500 오류

```python
# validator에서 ValueError 대신 내장 제약 사용
# ❌ raise ValueError("양수여야 함")  → 500 반환
# ✅ Field(gt=0)                    → 422 반환
```

### 4. Optional Form Fields Literal 회귀

```python
# FastAPI 0.114.0+에서 optional Literal form 필드가 422 반환
# Literal 대신 str | None 사용
# ❌ attribute: Annotated[Optional[Literal["a", "b"]], Form()]
# ✅ attribute: Annotated[str | None, Form()] = None
```

## Alembic 마이그레이션

### Classical Mapping과 Alembic 연동

Classical Mapping 사용 시 Alembic의 `env.py`에서 매퍼를 먼저 초기화해야 `target_metadata`를 인식한다.

```python
# alembic/env.py
from common.infrastructure.persistence.mapping import start_mappers, metadata

# 매퍼 초기화 (메타데이터에 테이블이 등록되어야 autogenerate 동작)
start_mappers()
target_metadata = metadata
```

### 주요 명령어

```bash
# 마이그레이션 초기화
uv run alembic init alembic

# 마이그레이션 생성 (autogenerate)
uv run alembic revision --autogenerate -m "add_orders_table"

# 마이그레이션 적용
uv run alembic upgrade head

# 롤백
uv run alembic downgrade -1
```

### 주의사항
- `start_mappers()`가 `env.py`에서 호출되지 않으면 autogenerate가 빈 마이그레이션을 생성한다
- async 엔진 사용 시 `env.py`의 `run_migrations_online()`을 async로 변경해야 한다
- 프로덕션에서는 `create_all()` 대신 **반드시 Alembic 마이그레이션**을 사용한다

## 테스트 패턴

```python
# tests/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient

@pytest.fixture
async def client():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac

# tests/test_fortune_router.py
@pytest.mark.asyncio
async def test_create_fortune(client: AsyncClient):
    response = await client.post(
        "/fortunes",
        json={"user_name": "홍길동", "birth_date": "1990-01-01T00:00:00"},
    )
    assert response.status_code == 201
    assert response.json()["user_name"] == "홍길동"
```

## 실행 명령어

```bash
# 개발
uv run fastapi dev packages/user-api/src/user_api/main.py

# 프로덕션
uv run uvicorn user_api.main:app --host 0.0.0.0 --port 8000

# 테스트
uv run pytest packages/user-api/tests/ -xvs

# 린트
uv run ruff check . && uv run ruff format .
```
