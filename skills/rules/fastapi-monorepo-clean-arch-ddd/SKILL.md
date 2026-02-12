---
name: fastapi-monorepo-clean-arch-ddd
description: FastAPI + uv workspace 모노레포 기반 Clean Architecture + DDD. 실제 서비스 가능한 중소규모 프로젝트 전용 구조.
---

# FastAPI Clean Architecture + DDD (실제 서비스용)

## 대상 프로젝트

- **규모**: 중소규모 모노리스 (Aggregate 30개 이하)
- **팀**: 혼자 또는 작은 팀 (1~5명)
- **목적**: MVP가 아닌 실제 서비스 운영 (2년 이상)

❌ **부적합**: 마이크로서비스, 대규모 조직, 빠른 프로토타입

## 핵심 원칙

```
✅ 레이어 분리 엄격 (유지보수성 우선)
✅ 파이썬 특성 반영 (과도한 추상화 제거)
✅ 실용적 적용 (복잡도에 비례한 구조)
```

---

## 레이어 정의

```
presentation → application → domain ← infrastructure
(Router)      (UseCase/Service)  (Entity)  (ORM/외부API)
```

| 레이어             | 책임                             | 포함 요소                                          |
| ------------------ | -------------------------------- | -------------------------------------------------- |
| **domain**         | 비즈니스 핵심 규칙 (순수 Python) | Entity, Value Object, Repository ABC, Domain Event |
| **application**    | 유스케이스 조율, 비즈니스 로직   | UseCase (Facade), Service (CQRS), DTO              |
| **infrastructure** | 외부 시스템 연동                 | ORM Model, Mapper, Repository 구현체, 외부 API     |
| **presentation**   | 요청/응답 처리                   | Router, Pydantic Schema                            |

---

## 의존성 규칙 (자바 Spring 패턴)

### 의존성 방향 매트릭스

| 호출자 ↓ \ 대상 → | Router | UseCase | Service | Repository |
| ----------------- | :----: | :-----: | :-----: | :--------: |
| **Router**        |   -    |   ✅    |   ❌    |     ❌     |
| **UseCase**       |   ❌   |   ❌    |   ✅    |     ❌     |
| **Service**       |   ❌   |   ❌    |   ❌    |     ✅     |
| **Repository**    |   ❌   |   ❌    |   ❌    |     -      |

> **같은 레이어 간 호출 금지**, 역방향 호출 금지

### 레이어별 핵심 규칙

- **Router**: UseCase만 호출 (1:1 위임)
- **UseCase**: Service 조합, 트랜잭션 경계, 크로스 도메인 조합
- **Service**: 자기 도메인 Repository만 (타 도메인 Service 호출 금지)
- **Repository**: 데이터 접근, Entity ↔ Model 변환

### 절대 금지

- Router에서 Service 직접 호출
- **Service에서 다른 Service 호출** (크로스 도메인은 UseCase에서)
- domain에서 infrastructure import
- application에서 ORM Model 직접 사용

---

## 모노레포 구조

```
packages/
  domain/                             # 순수 도메인
    src/domain/
      shared/                         # 공통 (Base Entity, DomainError 등)
      {aggregate}/                    # User, Project, Order 등
        entity/{aggregate}.py
        repository/{aggregate}_repository.py  # ABC
        event/{aggregate}_events.py

  infrastructure/                     # domain 구현
    src/infrastructure/
      persistence/
        model/{aggregate}_model.py    # SQLAlchemy Model
        mapper/{aggregate}_mapper.py  # Entity ↔ Model 변환
        repository/{aggregate}_repository_impl.py
        database.py
      cache/redis_client.py
      external/email_service.py

  user-api/                           # 사용자 API
    src/user_api/
      application/
        {aggregate}/
          service/
            {aggregate}_command_service.py  # ⭐ CQRS 쓰기
            {aggregate}_query_service.py    # ⭐ CQRS 읽기
          use_case/{use_case_name}.py       # ⭐ Facade
          dto/{aggregate}_dto.py
      presentation/{aggregate}/
        router.py
        schema.py
      config/dependencies.py
      main.py

  admin-api/                          # 관리자 API (동일 구조)
```

### pyproject.toml 설정

```toml
# 루트
[tool.uv.workspace]
members = ["packages/*"]

# user-api
[project]
dependencies = ["domain", "infrastructure", "fastapi"]

[tool.uv.sources]
domain = { workspace = true }
infrastructure = { workspace = true }
```

---

## 핵심 패턴

### 1. Aggregate

> **"한 번의 트랜잭션에서 일관성을 보장해야 하는 객체들의 묶음"**

#### 규칙

1. **외부는 Root만 접근**
2. **Aggregate 간은 ID로만 참조** (`customer_id: str`, NOT `customer: Customer`)
3. **하나의 트랜잭션 = 하나의 Aggregate**

```python
class Order:
    def __init__(self, customer_id: str):  # ✅ ID만
        self._customer_id = customer_id
```

### 2. CQRS (Command/Query Separation)

```
CommandService: 쓰기 (생성, 수정, 삭제)
QueryService: 읽기 (조회, 검색)
```

#### 적용 기준

| 상황                 | Service 레이어                             |
| -------------------- | ------------------------------------------ |
| 크로스 도메인 조합   | ✅ 필수                                    |
| 복잡한 비즈니스 로직 | ✅ 필수                                    |
| 단순 CRUD            | ⚠️ 선택적 (UseCase → Repository 직접 가능) |

### 3. 크로스 도메인 조합

> **도메인 간 조합은 반드시 UseCase에서만**

#### 왜 Service 간 호출을 금지?

1. **순환 의존 방지**: ServiceA → ServiceB → ServiceA 차단
2. **트랜잭션 경계 명확화**: UseCase가 유일한 오케스트레이터
3. **테스트 용이성**: Service는 Repository mock만으로 테스트
4. **변경 영향 최소화**: 결합점이 UseCase에만 존재

---

## 코드 예시 (핵심만)

### Domain Entity (순수 Python)

```python
# domain/user/entity/user.py
class User:
    """순수 Python - 외부 의존성 없음"""
    def __init__(self, id: str, email: str, is_active: bool = False):
        self._id = id
        self._email = email
        self._is_active = is_active

    def activate(self) -> None:
        """비즈니스 로직은 Entity에"""
        if self._is_active:
            raise DomainError("이미 활성화된 사용자")
        self._is_active = True
```

### Repository ABC (domain)

```python
# domain/user/repository/user_repository.py
from abc import ABC, abstractmethod

class UserRepository(ABC):
    @abstractmethod
    async def find_by_id(self, user_id: str) -> User | None: ...

    @abstractmethod
    async def save(self, user: User) -> None: ...
```

### SQLAlchemy (infrastructure)

```python
# infrastructure/persistence/model/user_model.py
from sqlalchemy.orm import Mapped, mapped_column

class UserModel(Base):
    __tablename__ = "users"
    id: Mapped[str] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    is_active: Mapped[bool] = mapped_column(default=False)

# infrastructure/persistence/mapper/user_mapper.py
class UserMapper:
    @staticmethod
    def to_entity(model: UserModel) -> User:
        return User(id=model.id, email=model.email, is_active=model.is_active)

    @staticmethod
    def to_model(entity: User) -> UserModel:
        return UserModel(id=entity.id, email=entity.email, is_active=entity.is_active)

# infrastructure/persistence/repository/user_repository_impl.py
class UserRepositoryImpl(UserRepository):
    async def save(self, user: User) -> None:
        existing = await self._session.execute(
            select(UserModel).where(UserModel.id == user.id)
        )
        model = existing.scalar_one_or_none()

        if model:
            # 업데이트
            model.email = user.email
            model.is_active = user.is_active
        else:
            # 새로 생성
            self._session.add(UserMapper.to_model(user))
```

### Service (CQRS)

```python
# user-api/application/user/service/user_command_service.py
class UserCommandService:
    """쓰기 작업 (자기 도메인만)"""
    async def create_user(self, email: str) -> User:
        existing = await self._repo.find_by_email(email)
        if existing:
            raise DomainError("중복 이메일")

        user = User(id=str(uuid.uuid4()), email=email)
        await self._repo.save(user)
        return user

# user-api/application/user/service/user_query_service.py
class UserQueryService:
    """읽기 작업"""
    async def find_by_ids(self, user_ids: list[str]) -> dict[str, User]:
        """벌크 조회 (크로스 도메인 조합용)"""
        # ...
```

### UseCase (Facade)

```python
# user-api/application/user/use_case/register_user.py
class RegisterUserUseCase:
    """UseCase = Facade (Service 조합 + 트랜잭션)"""
    def __init__(
        self,
        user_cmd: UserCommandService,
        email_service: EmailService,
        session: AsyncSession,
    ):
        self._user_cmd = user_cmd
        self._email_service = email_service
        self._session = session

    async def execute(self, email: str, password: str) -> UserResponse:
        # 트랜잭션 경계
        async with self._session.begin():
            user = await self._user_cmd.create_user(email, hash_password(password))

        # 외부 서비스 (트랜잭션 외부)
        await self._email_service.send_verification(user.email)
        return UserResponse.from_entity(user)

# user-api/application/project/use_case/create_project.py
class CreateProjectUseCase:
    """크로스 도메인 조합"""
    def __init__(
        self,
        project_cmd: ProjectCommandService,
        workspace_query: WorkspaceQueryService,  # 타 도메인
        session: AsyncSession,
    ):
        ...

    async def execute(self, workspace_id: str, name: str) -> ProjectResponse:
        # 1. 크로스 도메인 검증
        await self._workspace_query.validate_member(workspace_id, user_id)

        # 2. 트랜잭션
        async with self._session.begin():
            project = await self._project_cmd.create(workspace_id, name)

        return ProjectResponse.from_entity(project)
```

### Router

```python
# user-api/presentation/user/router.py
@router.post("/register", response_model=UserResponse)
async def register_user(
    request: RegisterUserRequest,
    use_case: RegisterUserUseCase = Depends(),
):
    """Router는 UseCase만 호출"""
    return await use_case.execute(request.email, request.password)
```

---

## 에러 처리

### 레이어별 에러 타입

| 레이어             | 에러 타입                           | 예시                                  |
| ------------------ | ----------------------------------- | ------------------------------------- |
| **domain**         | `DomainError` 계층                  | `InvalidOrderError`                   |
| **application**    | `ApplicationError`                  | `OrderNotFoundError`                  |
| **infrastructure** | infra 예외 → 도메인 예외로 변환     | `SQLAlchemyError` → `RepositoryError` |
| **presentation**   | `RequestValidationError` (Pydantic) | 422                                   |

### 규칙

- domain 에러에 HTTP 상태 코드 **절대 포함하지 않음**
- infrastructure 예외는 **반드시 도메인 예외로 변환**
- exception_handler에서 HTTP 응답으로 매핑

```python
# domain/shared/error/domain_error.py
class DomainError(Exception):
    def __init__(self, message: str):
        self.message = message

# user-api/presentation/exception_handlers.py
@app.exception_handler(DomainError)
async def handle_domain_error(request: Request, exc: DomainError):
    return JSONResponse(status_code=400, content={"detail": exc.message})
```

---

## Anti-Patterns

| 안티패턴                                | 올바른 방법                                  |
| --------------------------------------- | -------------------------------------------- |
| Router → Service 직접 호출              | Router → UseCase → Service                   |
| Service → 다른 Service 호출             | 크로스 도메인은 UseCase에서                  |
| Entity에 ORM 데코레이터                 | Entity는 순수 Python, ORM은 infrastructure에 |
| Aggregate 간 객체 참조                  | ID로만 참조 (`customer_id: str`)             |
| UseCase에 비즈니스 로직                 | 비즈니스 로직은 Entity/Service에             |
| 하나의 트랜잭션에서 여러 Aggregate 수정 | Aggregate당 하나의 트랜잭션, 나머지는 Event  |
| DTO 없이 Entity 직접 반환               | Response DTO로 변환                          |
| DomainError에 HTTP 상태 코드            | 도메인은 HTTP를 모름. presentation에서 매핑  |

---

## 테스트 책임

| 레이어         | 테스트 범위                         | Mock 대상                |
| -------------- | ----------------------------------- | ------------------------ |
| **Router**     | HTTP 요청/응답, Pydantic 검증       | UseCase                  |
| **UseCase**    | 비즈니스 플로우, 크로스 도메인 조율 | Service                  |
| **Service**    | 단일 도메인 비즈니스 로직           | Repository               |
| **Repository** | 쿼리 정확성                         | 실제 DB (testcontainers) |

> Service 테스트에서 다른 Service를 mock할 필요가 없다면, 레이어 규칙이 잘 지켜지고 있는 것

---

## 기타 패턴

### Domain Event

```python
# domain/user/event/user_events.py
@dataclass(frozen=True)
class UserCreated:
    user_id: str
    occurred_at: datetime

# UseCase에서 발행
await self._event_bus.publish(UserCreated(user_id=user.id, occurred_at=datetime.now()))
```

### ID 생성

```python
# 방법 1: uuid 직접 (권장)
user = User(id=str(uuid.uuid4()), ...)

# 방법 2: ULID (시간순 정렬 필요 시)
user = User(id=str(ulid.new()), ...)
```

---

## 정리

### 핵심 원칙

1. **레이어 분리 엄격**: Router → UseCase → Service → Repository
2. **의존성 방향 준수**: 자바 Spring 매트릭스와 동일
3. **Aggregate 중심**: 트랜잭션 = Aggregate 단위
4. **크로스 도메인은 UseCase**: Service는 자기 도메인만
5. **Entity 순수성**: domain은 외부 의존성 없음

### 실용적 적용

- **복잡한 로직**: UseCase → Service → Repository
- **단순한 로직**: UseCase → Repository (Service 생략 가능)
- **최소 기준**: Router는 무조건 UseCase만 호출

### 장점

- ✅ 유지보수성 (레이어 분리 명확)
- ✅ 테스트 용이성 (독립 테스트)
- ✅ 확장성 (새 도메인 추가 쉬움)
- ✅ 안정성 (의존성 방향 위반 방지)
