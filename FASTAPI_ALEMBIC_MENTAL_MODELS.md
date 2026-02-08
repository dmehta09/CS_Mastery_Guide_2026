# FastAPI & Alembic Mental Models 2026

## Part 1: FastAPI Mental Models

---

## Mental Model 1: FastAPI Request Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FASTAPI REQUEST LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  HTTP Request arrives                                                      │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. MIDDLEWARE STACK (Before)                                       │   │
│  │     - CORS middleware                                               │   │
│  │     - Authentication middleware                                     │   │
│  │     - Logging middleware                                            │   │
│  │     - Custom middleware                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  2. ROUTING                                                         │   │
│  │     Match path + HTTP method to route handler                       │   │
│  │     /users/{user_id} + GET → get_user()                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  3. DEPENDENCY INJECTION                                            │   │
│  │     Resolve all Depends() in order                                  │   │
│  │     - Database sessions                                             │   │
│  │     - Current user                                                  │   │
│  │     - Settings                                                      │   │
│  │     - Service classes                                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  4. REQUEST VALIDATION (Pydantic)                                   │   │
│  │     - Path parameters                                               │   │
│  │     - Query parameters                                              │   │
│  │     - Request body                                                  │   │
│  │     - Headers                                                       │   │
│  │     → 422 Validation Error if fails                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  5. PATH OPERATION FUNCTION (Your Code)                             │   │
│  │     async def get_user(user_id: int, db: Session):                 │   │
│  │         return db.query(User).get(user_id)                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  6. RESPONSE VALIDATION (Pydantic)                                  │   │
│  │     Serialize return value to response_model                        │   │
│  │     Filter out fields not in schema                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  7. MIDDLEWARE STACK (After)                                        │   │
│  │     Response flows back through middleware                          │   │
│  │     - Add headers                                                   │   │
│  │     - Log response                                                  │   │
│  │     - Timing                                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                    │
│        ▼                                                                    │
│  HTTP Response sent                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exception Handling Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EXCEPTION HANDLING                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Exception raised anywhere in the stack                                    │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Is it HTTPException?                                               │   │
│  │       │                                                             │   │
│  │  YES  │   NO                                                        │   │
│  │  ▼    │                                                             │   │
│  │ Return│   Is there a custom exception handler?                      │   │
│  │ HTTP  │        │                                                    │   │
│  │ error │   YES  │   NO                                               │   │
│  │       │   ▼    │                                                    │   │
│  │       │ Custom │   Return 500 Internal Server Error                 │   │
│  │       │ handler│   (with traceback in debug mode)                   │   │
│  │       │        │                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CUSTOM EXCEPTION HANDLERS:                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  class UserNotFoundError(Exception):                               │   │
│  │      def __init__(self, user_id: int):                             │   │
│  │          self.user_id = user_id                                    │   │
│  │                                                                     │   │
│  │  @app.exception_handler(UserNotFoundError)                         │   │
│  │  async def user_not_found_handler(request, exc):                   │   │
│  │      return JSONResponse(                                          │   │
│  │          status_code=404,                                          │   │
│  │          content={"detail": f"User {exc.user_id} not found"}       │   │
│  │      )                                                             │   │
│  │                                                                     │   │
│  │  # Now in your code:                                               │   │
│  │  raise UserNotFoundError(user_id=123)  # Returns nice 404          │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 2: Dependency Injection System

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FASTAPI DEPENDENCY INJECTION                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  THE MENTAL MODEL: Dependencies are "providers" that run BEFORE your       │
│  route handler and provide values to it.                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  @app.get("/users/{user_id}")                                      │   │
│  │  async def get_user(                                               │   │
│  │      user_id: int,                          # From path            │   │
│  │      db: Session = Depends(get_db),         # Dependency           │   │
│  │      current_user: User = Depends(get_current_user),  # Dependency │   │
│  │      settings: Settings = Depends(get_settings),      # Dependency │   │
│  │  ):                                                                │   │
│  │      ...                                                           │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEPENDENCY RESOLUTION ORDER:                                              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │                    get_user() ◄── Your route handler               │   │
│  │                    /   │   \                                        │   │
│  │                   /    │    \                                       │   │
│  │                  ▼     ▼     ▼                                      │   │
│  │            get_db  get_current_user  get_settings                  │   │
│  │              │           │                                          │   │
│  │              │           ▼                                          │   │
│  │              │     get_token_from_header                           │   │
│  │              │           │                                          │   │
│  │              └─────►     ▼                                          │   │
│  │                    get_db (shared!)                                │   │
│  │                                                                     │   │
│  │  Dependencies are resolved bottom-up, cached per-request           │   │
│  │  (same dependency = same instance within one request)              │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DEPENDENCY TYPES:                                                         │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. SIMPLE FUNCTION                                                │   │
│  │  ─────────────────────                                             │   │
│  │  def get_db():                                                     │   │
│  │      db = SessionLocal()                                           │   │
│  │      try:                                                          │   │
│  │          yield db          # Yield = cleanup after request         │   │
│  │      finally:                                                      │   │
│  │          db.close()        # This runs AFTER response is sent      │   │
│  │                                                                     │   │
│  │  2. ASYNC FUNCTION                                                 │   │
│  │  ────────────────────                                              │   │
│  │  async def get_async_client():                                     │   │
│  │      async with httpx.AsyncClient() as client:                     │   │
│  │          yield client                                              │   │
│  │                                                                     │   │
│  │  3. CLASS-BASED (Callable)                                         │   │
│  │  ─────────────────────────                                         │   │
│  │  class Pagination:                                                 │   │
│  │      def __init__(self, skip: int = 0, limit: int = 10):          │   │
│  │          self.skip = skip                                          │   │
│  │          self.limit = limit                                        │   │
│  │                                                                     │   │
│  │  # Usage: pagination: Pagination = Depends(Pagination)             │   │
│  │                                                                     │   │
│  │  4. PARAMETERIZED DEPENDENCY                                       │   │
│  │  ───────────────────────────                                       │   │
│  │  def require_permission(permission: str):                          │   │
│  │      def checker(user: User = Depends(get_current_user)):          │   │
│  │          if permission not in user.permissions:                    │   │
│  │              raise HTTPException(403, "Forbidden")                 │   │
│  │          return user                                               │   │
│  │      return checker                                                │   │
│  │                                                                     │   │
│  │  # Usage: Depends(require_permission("admin"))                     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Dependency Injection Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMMON DI PATTERNS                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  PATTERN 1: Database Session                                        │   │
│  │                                                                     │   │
│  │  # database.py                                                     │   │
│  │  def get_db():                                                     │   │
│  │      db = SessionLocal()                                           │   │
│  │      try:                                                          │   │
│  │          yield db                                                  │   │
│  │      finally:                                                      │   │
│  │          db.close()                                                │   │
│  │                                                                     │   │
│  │  # Async version with SQLAlchemy 2.0                               │   │
│  │  async def get_async_db():                                         │   │
│  │      async with async_session() as session:                        │   │
│  │          try:                                                      │   │
│  │              yield session                                         │   │
│  │              await session.commit()                                │   │
│  │          except Exception:                                         │   │
│  │              await session.rollback()                              │   │
│  │              raise                                                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  PATTERN 2: Current User with JWT                                   │   │
│  │                                                                     │   │
│  │  oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")            │   │
│  │                                                                     │   │
│  │  async def get_current_user(                                       │   │
│  │      token: str = Depends(oauth2_scheme),                          │   │
│  │      db: Session = Depends(get_db)                                 │   │
│  │  ) -> User:                                                        │   │
│  │      credentials_exception = HTTPException(                        │   │
│  │          status_code=401,                                          │   │
│  │          detail="Could not validate credentials",                  │   │
│  │          headers={"WWW-Authenticate": "Bearer"},                   │   │
│  │      )                                                             │   │
│  │      try:                                                          │   │
│  │          payload = jwt.decode(token, SECRET_KEY, algorithms=[ALG]) │   │
│  │          user_id: int = payload.get("sub")                         │   │
│  │          if user_id is None:                                       │   │
│  │              raise credentials_exception                           │   │
│  │      except JWTError:                                              │   │
│  │          raise credentials_exception                               │   │
│  │                                                                     │   │
│  │      user = db.query(User).filter(User.id == user_id).first()      │   │
│  │      if user is None:                                              │   │
│  │          raise credentials_exception                               │   │
│  │      return user                                                   │   │
│  │                                                                     │   │
│  │  # Optional user (for public endpoints)                            │   │
│  │  async def get_current_user_optional(                              │   │
│  │      token: str | None = Depends(oauth2_scheme_optional)           │   │
│  │  ) -> User | None:                                                 │   │
│  │      if token is None:                                             │   │
│  │          return None                                               │   │
│  │      return await get_current_user(token)                          │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  PATTERN 3: Service Layer Injection                                 │   │
│  │                                                                     │   │
│  │  class UserService:                                                │   │
│  │      def __init__(self, db: Session):                              │   │
│  │          self.db = db                                              │   │
│  │                                                                     │   │
│  │      def get_user(self, user_id: int) -> User | None:              │   │
│  │          return self.db.query(User).filter(User.id == user_id).first()
│  │                                                                     │   │
│  │      def create_user(self, user_data: UserCreate) -> User:         │   │
│  │          user = User(**user_data.model_dump())                     │   │
│  │          self.db.add(user)                                         │   │
│  │          self.db.commit()                                          │   │
│  │          return user                                               │   │
│  │                                                                     │   │
│  │  def get_user_service(db: Session = Depends(get_db)) -> UserService:
│  │      return UserService(db)                                        │   │
│  │                                                                     │   │
│  │  # Usage in route                                                  │   │
│  │  @app.get("/users/{user_id}")                                      │   │
│  │  def get_user(                                                     │   │
│  │      user_id: int,                                                 │   │
│  │      user_service: UserService = Depends(get_user_service)         │   │
│  │  ):                                                                │   │
│  │      return user_service.get_user(user_id)                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 3: Pydantic Models & Validation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PYDANTIC IN FASTAPI                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  THE MENTAL MODEL: Pydantic models are DATA CONTRACTS that define          │
│  shape, types, and validation rules for data flowing in and out.           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │                    ┌─────────────┐                                 │   │
│  │   Request Body ──▶ │  UserCreate │ ──▶ Validated data             │   │
│  │   (raw JSON)       │  (Pydantic) │     (Python object)            │   │
│  │                    └─────────────┘                                 │   │
│  │                           │                                        │   │
│  │                    Validation failed?                              │   │
│  │                           │                                        │   │
│  │                    422 Unprocessable Entity                        │   │
│  │                    with detailed errors                            │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MODEL HIERARCHY PATTERN:                                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Base model - shared fields                                      │   │
│  │  class UserBase(BaseModel):                                        │   │
│  │      email: EmailStr                                               │   │
│  │      name: str                                                     │   │
│  │      is_active: bool = True                                        │   │
│  │                                                                     │   │
│  │  # Create model - for POST requests                                │   │
│  │  class UserCreate(UserBase):                                       │   │
│  │      password: str  # Only needed on creation                      │   │
│  │                                                                     │   │
│  │  # Update model - all fields optional                              │   │
│  │  class UserUpdate(BaseModel):                                      │   │
│  │      email: EmailStr | None = None                                 │   │
│  │      name: str | None = None                                       │   │
│  │      password: str | None = None                                   │   │
│  │      is_active: bool | None = None                                 │   │
│  │                                                                     │   │
│  │  # Response model - what we return (no password!)                  │   │
│  │  class UserResponse(UserBase):                                     │   │
│  │      id: int                                                       │   │
│  │      created_at: datetime                                          │   │
│  │                                                                     │   │
│  │      model_config = ConfigDict(from_attributes=True)               │   │
│  │      # Allows: UserResponse.model_validate(db_user)                │   │
│  │                                                                     │   │
│  │  # List response with pagination                                   │   │
│  │  class UserListResponse(BaseModel):                                │   │
│  │      items: list[UserResponse]                                     │   │
│  │      total: int                                                    │   │
│  │      page: int                                                     │   │
│  │      per_page: int                                                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  VALIDATION TECHNIQUES:                                                    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  from pydantic import BaseModel, Field, field_validator, model_validator
│  │                                                                     │   │
│  │  class Product(BaseModel):                                         │   │
│  │      # Field constraints                                           │   │
│  │      name: str = Field(..., min_length=1, max_length=100)         │   │
│  │      price: float = Field(..., gt=0, description="Price in USD")  │   │
│  │      quantity: int = Field(default=0, ge=0)                       │   │
│  │      tags: list[str] = Field(default_factory=list, max_length=10) │   │
│  │                                                                     │   │
│  │      # Field-level validator                                       │   │
│  │      @field_validator('name')                                      │   │
│  │      @classmethod                                                  │   │
│  │      def name_must_not_contain_special(cls, v: str) -> str:       │   │
│  │          if not v.replace(' ', '').isalnum():                     │   │
│  │              raise ValueError('Name must be alphanumeric')         │   │
│  │          return v.strip()                                          │   │
│  │                                                                     │   │
│  │      # Model-level validator (access multiple fields)              │   │
│  │      @model_validator(mode='after')                                │   │
│  │      def check_price_quantity(self) -> 'Product':                  │   │
│  │          if self.price > 1000 and self.quantity > 100:            │   │
│  │              raise ValueError('High value inventory needs approval')
│  │          return self                                               │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RESPONSE MODEL USAGE:                                                     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  @app.post("/users", response_model=UserResponse)                  │   │
│  │  def create_user(user: UserCreate, db: Session = Depends(get_db)): │   │
│  │      db_user = User(                                               │   │
│  │          email=user.email,                                         │   │
│  │          name=user.name,                                           │   │
│  │          hashed_password=hash_password(user.password)              │   │
│  │      )                                                             │   │
│  │      db.add(db_user)                                               │   │
│  │      db.commit()                                                   │   │
│  │      return db_user  # FastAPI serializes via UserResponse         │   │
│  │      # password is NOT in response because UserResponse doesn't have it
│  │                                                                     │   │
│  │  # Exclude unset fields (for PATCH)                                │   │
│  │  @app.patch("/users/{user_id}", response_model=UserResponse)       │   │
│  │  def update_user(user_id: int, user: UserUpdate):                  │   │
│  │      update_data = user.model_dump(exclude_unset=True)             │   │
│  │      # Only fields that were actually sent                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 4: Async/Await in FastAPI

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ASYNC IN FASTAPI                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SYNC VS ASYNC DECISION:                                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  Is your code doing I/O (database, HTTP calls, file reads)?        │   │
│  │       │                                                             │   │
│  │       ├── NO (CPU-bound) ────▶ Use sync def                        │   │
│  │       │                        FastAPI runs in threadpool          │   │
│  │       │                                                             │   │
│  │       └── YES (I/O-bound) ──▶ Is the library async-native?         │   │
│  │                                    │                                │   │
│  │                              YES   │   NO                           │   │
│  │                              ▼     │   ▼                            │   │
│  │                         async def  │  sync def (or run_in_executor)│   │
│  │                                    │                                │   │
│  │  EXAMPLES:                                                         │   │
│  │  ─────────                                                         │   │
│  │  async def:  httpx, asyncpg, aiofiles, SQLAlchemy 2.0 async        │   │
│  │  sync def:   requests, psycopg2, old SQLAlchemy, file I/O          │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  HOW FASTAPI HANDLES IT:                                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Async endpoint - runs on main event loop                        │   │
│  │  @app.get("/async")                                                │   │
│  │  async def async_endpoint():                                       │   │
│  │      await some_async_operation()  # Non-blocking                  │   │
│  │      return {"result": "done"}                                     │   │
│  │                                                                     │   │
│  │  # Sync endpoint - runs in external threadpool                     │   │
│  │  @app.get("/sync")                                                 │   │
│  │  def sync_endpoint():                                              │   │
│  │      time.sleep(1)  # Blocking, but won't block other requests     │   │
│  │      return {"result": "done"}                                     │   │
│  │                                                                     │   │
│  │  ┌───────────────────────────────────────────────────────────────┐ │   │
│  │  │                    Event Loop                                 │ │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │ │   │
│  │  │  │ async   │  │ async   │  │ async   │   Concurrent!         │ │   │
│  │  │  │ handler │  │ handler │  │ handler │                       │ │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘                       │ │   │
│  │  │       │            │            │                             │ │   │
│  │  │       └────────────┴────────────┘                             │ │   │
│  │  │                    │                                          │ │   │
│  │  │              ┌─────────────┐                                  │ │   │
│  │  │              │ Threadpool  │  For sync handlers               │ │   │
│  │  │              │ ┌───┐ ┌───┐ │                                  │ │   │
│  │  │              │ │ T │ │ T │ │                                  │ │   │
│  │  │              │ └───┘ └───┘ │                                  │ │   │
│  │  │              └─────────────┘                                  │ │   │
│  │  └───────────────────────────────────────────────────────────────┘ │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MIXING SYNC AND ASYNC:                                                    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Running sync code in async context                              │   │
│  │  import asyncio                                                    │   │
│  │  from functools import partial                                     │   │
│  │                                                                     │   │
│  │  async def async_endpoint():                                       │   │
│  │      loop = asyncio.get_event_loop()                               │   │
│  │      # Run blocking code in threadpool                             │   │
│  │      result = await loop.run_in_executor(                          │   │
│  │          None,  # Default threadpool                               │   │
│  │          partial(blocking_function, arg1, arg2)                    │   │
│  │      )                                                             │   │
│  │      return result                                                 │   │
│  │                                                                     │   │
│  │  # Better: Use dedicated library                                   │   │
│  │  from starlette.concurrency import run_in_threadpool               │   │
│  │                                                                     │   │
│  │  async def async_endpoint():                                       │   │
│  │      result = await run_in_threadpool(blocking_function, arg1)     │   │
│  │      return result                                                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 5: FastAPI Project Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RECOMMENDED PROJECT STRUCTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  my_app/                                                           │   │
│  │  ├── alembic/                    # Database migrations             │   │
│  │  │   ├── versions/                                                 │   │
│  │  │   ├── env.py                                                    │   │
│  │  │   └── script.py.mako                                            │   │
│  │  │                                                                 │   │
│  │  ├── app/                                                          │   │
│  │  │   ├── __init__.py                                               │   │
│  │  │   ├── main.py                 # FastAPI app creation            │   │
│  │  │   ├── config.py               # Settings (pydantic-settings)    │   │
│  │  │   ├── database.py             # DB connection, session          │   │
│  │  │   │                                                             │   │
│  │  │   ├── models/                 # SQLAlchemy models               │   │
│  │  │   │   ├── __init__.py                                           │   │
│  │  │   │   ├── user.py                                               │   │
│  │  │   │   └── product.py                                            │   │
│  │  │   │                                                             │   │
│  │  │   ├── schemas/                # Pydantic schemas                │   │
│  │  │   │   ├── __init__.py                                           │   │
│  │  │   │   ├── user.py                                               │   │
│  │  │   │   └── product.py                                            │   │
│  │  │   │                                                             │   │
│  │  │   ├── api/                    # Route handlers                  │   │
│  │  │   │   ├── __init__.py                                           │   │
│  │  │   │   ├── deps.py             # Shared dependencies             │   │
│  │  │   │   └── v1/                                                   │   │
│  │  │   │       ├── __init__.py                                       │   │
│  │  │   │       ├── router.py       # Combines all routers            │   │
│  │  │   │       ├── users.py                                          │   │
│  │  │   │       └── products.py                                       │   │
│  │  │   │                                                             │   │
│  │  │   ├── services/               # Business logic                  │   │
│  │  │   │   ├── __init__.py                                           │   │
│  │  │   │   ├── user_service.py                                       │   │
│  │  │   │   └── email_service.py                                      │   │
│  │  │   │                                                             │   │
│  │  │   ├── repositories/           # Data access (optional layer)    │   │
│  │  │   │   ├── __init__.py                                           │   │
│  │  │   │   └── user_repository.py                                    │   │
│  │  │   │                                                             │   │
│  │  │   └── core/                   # Core utilities                  │   │
│  │  │       ├── __init__.py                                           │   │
│  │  │       ├── security.py         # JWT, password hashing           │   │
│  │  │       └── exceptions.py       # Custom exceptions               │   │
│  │  │                                                                 │   │
│  │  ├── tests/                                                        │   │
│  │  │   ├── conftest.py             # Pytest fixtures                 │   │
│  │  │   ├── test_users.py                                             │   │
│  │  │   └── test_products.py                                          │   │
│  │  │                                                                 │   │
│  │  ├── alembic.ini                                                   │   │
│  │  ├── pyproject.toml                                                │   │
│  │  ├── Dockerfile                                                    │   │
│  │  └── docker-compose.yml                                            │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KEY FILES:                                                                │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # main.py                                                         │   │
│  │  from fastapi import FastAPI                                       │   │
│  │  from app.api.v1.router import api_router                          │   │
│  │  from app.config import settings                                   │   │
│  │                                                                     │   │
│  │  app = FastAPI(                                                    │   │
│  │      title=settings.PROJECT_NAME,                                  │   │
│  │      openapi_url=f"{settings.API_V1_PREFIX}/openapi.json"         │   │
│  │  )                                                                 │   │
│  │                                                                     │   │
│  │  app.include_router(api_router, prefix=settings.API_V1_PREFIX)     │   │
│  │                                                                     │   │
│  │  # config.py                                                       │   │
│  │  from pydantic_settings import BaseSettings                        │   │
│  │                                                                     │   │
│  │  class Settings(BaseSettings):                                     │   │
│  │      PROJECT_NAME: str = "My API"                                  │   │
│  │      API_V1_PREFIX: str = "/api/v1"                                │   │
│  │      DATABASE_URL: str                                             │   │
│  │      SECRET_KEY: str                                               │   │
│  │                                                                     │   │
│  │      model_config = ConfigDict(env_file=".env")                    │   │
│  │                                                                     │   │
│  │  settings = Settings()                                             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 6: Authentication Flow (JWT + OAuth2)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    JWT AUTHENTICATION FLOW                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. LOGIN FLOW                                                     │   │
│  │  ──────────────                                                    │   │
│  │                                                                     │   │
│  │  Client                           Server                           │   │
│  │    │                                │                              │   │
│  │    │  POST /token                   │                              │   │
│  │    │  {username, password}          │                              │   │
│  │    │ ─────────────────────────────▶ │                              │   │
│  │    │                                │  Verify credentials          │   │
│  │    │                                │  Generate JWT tokens         │   │
│  │    │   {access_token, token_type}   │                              │   │
│  │    │ ◀───────────────────────────── │                              │   │
│  │    │                                │                              │   │
│  │  Store token                        │                              │   │
│  │  (memory/localStorage)              │                              │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  2. AUTHENTICATED REQUEST FLOW                                     │   │
│  │  ─────────────────────────────────                                 │   │
│  │                                                                     │   │
│  │  Client                           Server                           │   │
│  │    │                                │                              │   │
│  │    │  GET /users/me                 │                              │   │
│  │    │  Authorization: Bearer <token> │                              │   │
│  │    │ ─────────────────────────────▶ │                              │   │
│  │    │                                │                              │   │
│  │    │                        ┌───────┴───────┐                      │   │
│  │    │                        │ Decode JWT    │                      │   │
│  │    │                        │ Verify sig    │                      │   │
│  │    │                        │ Check expiry  │                      │   │
│  │    │                        │ Load user     │                      │   │
│  │    │                        └───────┬───────┘                      │   │
│  │    │                                │                              │   │
│  │    │   {user data}                  │                              │   │
│  │    │ ◀───────────────────────────── │                              │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  IMPLEMENTATION:                                                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # core/security.py                                                │   │
│  │  from datetime import datetime, timedelta                          │   │
│  │  from jose import JWTError, jwt                                    │   │
│  │  from passlib.context import CryptContext                          │   │
│  │                                                                     │   │
│  │  pwd_context = CryptContext(schemes=["bcrypt"])                    │   │
│  │                                                                     │   │
│  │  def verify_password(plain: str, hashed: str) -> bool:             │   │
│  │      return pwd_context.verify(plain, hashed)                      │   │
│  │                                                                     │   │
│  │  def get_password_hash(password: str) -> str:                      │   │
│  │      return pwd_context.hash(password)                             │   │
│  │                                                                     │   │
│  │  def create_access_token(                                          │   │
│  │      data: dict,                                                   │   │
│  │      expires_delta: timedelta = timedelta(minutes=15)              │   │
│  │  ) -> str:                                                         │   │
│  │      to_encode = data.copy()                                       │   │
│  │      expire = datetime.utcnow() + expires_delta                    │   │
│  │      to_encode.update({"exp": expire})                             │   │
│  │      return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM) │   │
│  │                                                                     │   │
│  │  # api/v1/auth.py                                                  │   │
│  │  @router.post("/token", response_model=Token)                      │   │
│  │  async def login(                                                  │   │
│  │      form_data: OAuth2PasswordRequestForm = Depends(),             │   │
│  │      db: Session = Depends(get_db)                                 │   │
│  │  ):                                                                │   │
│  │      user = authenticate_user(db, form_data.username, form_data.password)
│  │      if not user:                                                  │   │
│  │          raise HTTPException(                                      │   │
│  │              status_code=401,                                      │   │
│  │              detail="Incorrect username or password",              │   │
│  │              headers={"WWW-Authenticate": "Bearer"},               │   │
│  │          )                                                         │   │
│  │      access_token = create_access_token(data={"sub": str(user.id)})│   │
│  │      return {"access_token": access_token, "token_type": "bearer"} │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 7: FastAPI Testing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TESTING FASTAPI APPLICATIONS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  TEST CLIENT SETUP:                                                │   │
│  │                                                                     │   │
│  │  # conftest.py                                                     │   │
│  │  import pytest                                                     │   │
│  │  from fastapi.testclient import TestClient                         │   │
│  │  from sqlalchemy import create_engine                              │   │
│  │  from sqlalchemy.orm import sessionmaker                           │   │
│  │                                                                     │   │
│  │  # Test database                                                   │   │
│  │  SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"                   │   │
│  │  engine = create_engine(SQLALCHEMY_DATABASE_URL)                   │   │
│  │  TestingSessionLocal = sessionmaker(bind=engine)                   │   │
│  │                                                                     │   │
│  │  @pytest.fixture(scope="function")                                 │   │
│  │  def db():                                                         │   │
│  │      Base.metadata.create_all(bind=engine)                         │   │
│  │      db = TestingSessionLocal()                                    │   │
│  │      try:                                                          │   │
│  │          yield db                                                  │   │
│  │      finally:                                                      │   │
│  │          db.close()                                                │   │
│  │          Base.metadata.drop_all(bind=engine)                       │   │
│  │                                                                     │   │
│  │  @pytest.fixture(scope="function")                                 │   │
│  │  def client(db):                                                   │   │
│  │      def override_get_db():                                        │   │
│  │          try:                                                      │   │
│  │              yield db                                              │   │
│  │          finally:                                                  │   │
│  │              pass                                                  │   │
│  │                                                                     │   │
│  │      app.dependency_overrides[get_db] = override_get_db            │   │
│  │      with TestClient(app) as c:                                    │   │
│  │          yield c                                                   │   │
│  │      app.dependency_overrides.clear()                              │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  TEST EXAMPLES:                                                    │   │
│  │                                                                     │   │
│  │  # test_users.py                                                   │   │
│  │  def test_create_user(client):                                     │   │
│  │      response = client.post(                                       │   │
│  │          "/api/v1/users",                                          │   │
│  │          json={"email": "test@example.com", "password": "secret"}  │   │
│  │      )                                                             │   │
│  │      assert response.status_code == 201                            │   │
│  │      data = response.json()                                        │   │
│  │      assert data["email"] == "test@example.com"                    │   │
│  │      assert "id" in data                                           │   │
│  │      assert "password" not in data  # Not in response!             │   │
│  │                                                                     │   │
│  │  def test_create_user_duplicate_email(client, db):                 │   │
│  │      # Setup: Create existing user                                 │   │
│  │      existing_user = User(email="test@example.com", hashed_password="x")
│  │      db.add(existing_user)                                         │   │
│  │      db.commit()                                                   │   │
│  │                                                                     │   │
│  │      # Test: Try to create duplicate                               │   │
│  │      response = client.post(                                       │   │
│  │          "/api/v1/users",                                          │   │
│  │          json={"email": "test@example.com", "password": "secret"}  │   │
│  │      )                                                             │   │
│  │      assert response.status_code == 400                            │   │
│  │      assert "already registered" in response.json()["detail"]      │   │
│  │                                                                     │   │
│  │  def test_get_user_unauthorized(client):                           │   │
│  │      response = client.get("/api/v1/users/me")                     │   │
│  │      assert response.status_code == 401                            │   │
│  │                                                                     │   │
│  │  def test_get_user_authorized(client, auth_headers):               │   │
│  │      response = client.get("/api/v1/users/me", headers=auth_headers)
│  │      assert response.status_code == 200                            │   │
│  │                                                                     │   │
│  │  # Fixture for authenticated requests                              │   │
│  │  @pytest.fixture                                                   │   │
│  │  def auth_headers(client, db):                                     │   │
│  │      # Create test user                                            │   │
│  │      user = User(email="auth@test.com", hashed_password=hash("pw"))│   │
│  │      db.add(user)                                                  │   │
│  │      db.commit()                                                   │   │
│  │                                                                     │   │
│  │      # Login                                                       │   │
│  │      response = client.post("/token", data={                       │   │
│  │          "username": "auth@test.com",                              │   │
│  │          "password": "pw"                                          │   │
│  │      })                                                            │   │
│  │      token = response.json()["access_token"]                       │   │
│  │      return {"Authorization": f"Bearer {token}"}                   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ASYNC TESTING:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # For async endpoints, use httpx + pytest-asyncio                 │   │
│  │  import pytest                                                     │   │
│  │  from httpx import AsyncClient, ASGITransport                      │   │
│  │                                                                     │   │
│  │  @pytest.fixture                                                   │   │
│  │  async def async_client():                                         │   │
│  │      async with AsyncClient(                                       │   │
│  │          transport=ASGITransport(app=app),                         │   │
│  │          base_url="http://test"                                    │   │
│  │      ) as client:                                                  │   │
│  │          yield client                                              │   │
│  │                                                                     │   │
│  │  @pytest.mark.asyncio                                              │   │
│  │  async def test_async_endpoint(async_client):                      │   │
│  │      response = await async_client.get("/async-endpoint")          │   │
│  │      assert response.status_code == 200                            │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Alembic Mental Models

---

## Mental Model 8: Alembic Migration Concepts

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ALEMBIC MIGRATION MENTAL MODEL                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  THE CORE CONCEPT:                                                         │
│  Alembic = Version control for your database schema                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  Code (SQLAlchemy Models)      Database (Actual Tables)            │   │
│  │  ────────────────────────      ────────────────────────            │   │
│  │  class User(Base):             CREATE TABLE users (                │   │
│  │      id = Column(Integer)          id INTEGER PRIMARY KEY,         │   │
│  │      email = Column(String)        email VARCHAR(255)              │   │
│  │                                );                                   │   │
│  │                                                                     │   │
│  │            │                              ▲                         │   │
│  │            │                              │                         │   │
│  │            └──────── ALEMBIC ─────────────┘                         │   │
│  │                   (Migration files)                                 │   │
│  │                                                                     │   │
│  │  Alembic bridges the gap between your Python models                │   │
│  │  and the actual database schema                                    │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  REVISION CHAIN:                                                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  (base) ──▶ rev_001 ──▶ rev_002 ──▶ rev_003 ──▶ (head)            │   │
│  │              │           │           │                              │   │
│  │              │           │           └─ Add index to users          │   │
│  │              │           └─ Add posts table                         │   │
│  │              └─ Create users table                                  │   │
│  │                                                                     │   │
│  │  Each revision knows its:                                          │   │
│  │  - revision ID (unique hash)                                       │   │
│  │  - down_revision (parent)                                          │   │
│  │  - upgrade() function (apply change)                               │   │
│  │  - downgrade() function (revert change)                            │   │
│  │                                                                     │   │
│  │  The database stores current revision in 'alembic_version' table   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMON COMMANDS:                                                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Initialize alembic in project                                   │   │
│  │  alembic init alembic                                              │   │
│  │                                                                     │   │
│  │  # Generate migration from model changes (autogenerate)            │   │
│  │  alembic revision --autogenerate -m "Add users table"              │   │
│  │                                                                     │   │
│  │  # Create empty migration (manual)                                 │   │
│  │  alembic revision -m "Add custom function"                         │   │
│  │                                                                     │   │
│  │  # Apply all pending migrations                                    │   │
│  │  alembic upgrade head                                              │   │
│  │                                                                     │   │
│  │  # Apply specific migration                                        │   │
│  │  alembic upgrade abc123                                            │   │
│  │                                                                     │   │
│  │  # Rollback one migration                                          │   │
│  │  alembic downgrade -1                                              │   │
│  │                                                                     │   │
│  │  # Rollback to specific revision                                   │   │
│  │  alembic downgrade abc123                                          │   │
│  │                                                                     │   │
│  │  # Rollback all migrations                                         │   │
│  │  alembic downgrade base                                            │   │
│  │                                                                     │   │
│  │  # Show current revision                                           │   │
│  │  alembic current                                                   │   │
│  │                                                                     │   │
│  │  # Show migration history                                          │   │
│  │  alembic history                                                   │   │
│  │                                                                     │   │
│  │  # Show pending migrations                                         │   │
│  │  alembic history --indicate-current                                │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 9: Alembic Configuration & Setup

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ALEMBIC CONFIGURATION                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROJECT STRUCTURE:                                                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  my_project/                                                       │   │
│  │  ├── alembic/                                                      │   │
│  │  │   ├── versions/           # Migration files live here          │   │
│  │  │   │   ├── 001_create_users.py                                   │   │
│  │  │   │   ├── 002_add_posts.py                                      │   │
│  │  │   │   └── 003_add_index.py                                      │   │
│  │  │   ├── env.py              # Migration environment config        │   │
│  │  │   └── script.py.mako      # Template for new migrations         │   │
│  │  ├── app/                                                          │   │
│  │  │   ├── models/             # Your SQLAlchemy models              │   │
│  │  │   │   ├── __init__.py     # Import all models here!            │   │
│  │  │   │   ├── base.py         # Base = declarative_base()          │   │
│  │  │   │   └── user.py                                               │   │
│  │  │   └── database.py                                               │   │
│  │  └── alembic.ini             # Main config file                    │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KEY CONFIGURATION FILES:                                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # alembic.ini                                                     │   │
│  │  [alembic]                                                         │   │
│  │  script_location = alembic                                         │   │
│  │  prepend_sys_path = .                                              │   │
│  │  # Don't set sqlalchemy.url here - use env.py instead!            │   │
│  │                                                                     │   │
│  │  # alembic/env.py (the important parts)                            │   │
│  │  from logging.config import fileConfig                             │   │
│  │  from sqlalchemy import engine_from_config, pool                   │   │
│  │  from alembic import context                                       │   │
│  │                                                                     │   │
│  │  # THIS IS CRUCIAL: Import your models' Base                       │   │
│  │  from app.models import Base  # Import all models via __init__     │   │
│  │  from app.config import settings                                   │   │
│  │                                                                     │   │
│  │  # Set target metadata for autogenerate                            │   │
│  │  target_metadata = Base.metadata                                   │   │
│  │                                                                     │   │
│  │  def get_url():                                                    │   │
│  │      return settings.DATABASE_URL                                  │   │
│  │                                                                     │   │
│  │  def run_migrations_offline():                                     │   │
│  │      """Run migrations in 'offline' mode - generates SQL"""        │   │
│  │      url = get_url()                                               │   │
│  │      context.configure(                                            │   │
│  │          url=url,                                                  │   │
│  │          target_metadata=target_metadata,                          │   │
│  │          literal_binds=True,                                       │   │
│  │          dialect_opts={"paramstyle": "named"},                     │   │
│  │      )                                                             │   │
│  │      with context.begin_transaction():                             │   │
│  │          context.run_migrations()                                  │   │
│  │                                                                     │   │
│  │  def run_migrations_online():                                      │   │
│  │      """Run migrations in 'online' mode - connects to DB"""        │   │
│  │      configuration = config.get_section(config.config_ini_section) │   │
│  │      configuration["sqlalchemy.url"] = get_url()                   │   │
│  │      connectable = engine_from_config(                             │   │
│  │          configuration,                                            │   │
│  │          prefix="sqlalchemy.",                                     │   │
│  │          poolclass=pool.NullPool,                                  │   │
│  │      )                                                             │   │
│  │      with connectable.connect() as connection:                     │   │
│  │          context.configure(                                        │   │
│  │              connection=connection,                                │   │
│  │              target_metadata=target_metadata,                      │   │
│  │          )                                                         │   │
│  │          with context.begin_transaction():                         │   │
│  │              context.run_migrations()                              │   │
│  │                                                                     │   │
│  │  if context.is_offline_mode():                                     │   │
│  │      run_migrations_offline()                                      │   │
│  │  else:                                                             │   │
│  │      run_migrations_online()                                       │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CRITICAL: Import ALL models in app/models/__init__.py                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # app/models/__init__.py                                          │   │
│  │  from app.models.base import Base                                  │   │
│  │  from app.models.user import User     # Must import!               │   │
│  │  from app.models.post import Post     # Must import!               │   │
│  │  from app.models.comment import Comment  # Must import!            │   │
│  │                                                                     │   │
│  │  # If you don't import a model, autogenerate won't see it!        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 10: Migration File Anatomy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MIGRATION FILE STRUCTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # alembic/versions/001_abc123_create_users_table.py               │   │
│  │                                                                     │   │
│  │  """Create users table                                             │   │
│  │                                                                     │   │
│  │  Revision ID: abc123                                               │   │
│  │  Revises:                        # Empty = first migration         │   │
│  │  Create Date: 2024-01-15 10:30:00                                  │   │
│  │                                                                     │   │
│  │  """                                                               │   │
│  │  from alembic import op                                            │   │
│  │  import sqlalchemy as sa                                           │   │
│  │                                                                     │   │
│  │  # revision identifiers, used by Alembic                           │   │
│  │  revision = 'abc123'                                               │   │
│  │  down_revision = None            # Previous revision (or None)     │   │
│  │  branch_labels = None                                              │   │
│  │  depends_on = None                                                 │   │
│  │                                                                     │   │
│  │  def upgrade():                                                    │   │
│  │      """Apply the migration"""                                     │   │
│  │      op.create_table(                                              │   │
│  │          'users',                                                  │   │
│  │          sa.Column('id', sa.Integer(), primary_key=True),          │   │
│  │          sa.Column('email', sa.String(255), nullable=False),       │   │
│  │          sa.Column('hashed_password', sa.String(255)),             │   │
│  │          sa.Column('is_active', sa.Boolean(), default=True),       │   │
│  │          sa.Column('created_at', sa.DateTime(),                    │   │
│  │                    server_default=sa.func.now()),                  │   │
│  │      )                                                             │   │
│  │      op.create_index('ix_users_email', 'users', ['email'],         │   │
│  │                      unique=True)                                  │   │
│  │                                                                     │   │
│  │  def downgrade():                                                  │   │
│  │      """Revert the migration (opposite of upgrade)"""              │   │
│  │      op.drop_index('ix_users_email', 'users')                      │   │
│  │      op.drop_table('users')                                        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMON OPERATIONS:                                                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # TABLE OPERATIONS                                                │   │
│  │  op.create_table('tablename', *columns)                            │   │
│  │  op.drop_table('tablename')                                        │   │
│  │  op.rename_table('old_name', 'new_name')                           │   │
│  │                                                                     │   │
│  │  # COLUMN OPERATIONS                                               │   │
│  │  op.add_column('table', sa.Column('name', sa.String(50)))          │   │
│  │  op.drop_column('table', 'column_name')                            │   │
│  │  op.alter_column('table', 'column',                                │   │
│  │                  existing_type=sa.String(50),                      │   │
│  │                  type_=sa.String(100))                             │   │
│  │  op.alter_column('table', 'column', nullable=False)                │   │
│  │                                                                     │   │
│  │  # INDEX OPERATIONS                                                │   │
│  │  op.create_index('ix_name', 'table', ['column'])                   │   │
│  │  op.create_index('ix_name', 'table', ['col1', 'col2'], unique=True)│   │
│  │  op.drop_index('ix_name', 'table')                                 │   │
│  │                                                                     │   │
│  │  # FOREIGN KEY OPERATIONS                                          │   │
│  │  op.create_foreign_key('fk_name', 'source_table', 'target_table',  │   │
│  │                        ['source_col'], ['target_col'])             │   │
│  │  op.drop_constraint('fk_name', 'table', type_='foreignkey')        │   │
│  │                                                                     │   │
│  │  # EXECUTE RAW SQL (when ops aren't enough)                        │   │
│  │  op.execute("UPDATE users SET status = 'active' WHERE status IS NULL")
│  │                                                                     │   │
│  │  # DATA MIGRATION (use connection for complex cases)               │   │
│  │  def upgrade():                                                    │   │
│  │      conn = op.get_bind()                                          │   │
│  │      result = conn.execute(sa.text("SELECT id, name FROM users"))  │   │
│  │      for row in result:                                            │   │
│  │          # Process each row                                        │   │
│  │          conn.execute(                                             │   │
│  │              sa.text("UPDATE users SET slug = :slug WHERE id = :id"),
│  │              {"slug": slugify(row.name), "id": row.id}             │   │
│  │          )                                                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 11: Autogenerate & Its Limitations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AUTOGENERATE: WHAT IT CAN & CAN'T DO                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  HOW AUTOGENERATE WORKS:                                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  alembic revision --autogenerate -m "description"                  │   │
│  │                                                                     │   │
│  │  1. Reads your SQLAlchemy models (target_metadata)                 │   │
│  │  2. Reads current database schema                                  │   │
│  │  3. Compares them                                                  │   │
│  │  4. Generates migration for differences                            │   │
│  │                                                                     │   │
│  │  SQLAlchemy Models                  Database Schema                │   │
│  │  ──────────────────                 ───────────────                │   │
│  │  class User:                        users table                    │   │
│  │    id: int                          - id: INT                      │   │
│  │    email: str     ◄──── DIFF ────▶  - email: VARCHAR              │   │
│  │    name: str      ◄── NEW COLUMN    (no name column)              │   │
│  │                                                                     │   │
│  │  Result: Migration adds 'name' column to 'users' table             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ✓ WHAT AUTOGENERATE DETECTS:                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ✓ Table additions and removals                                    │   │
│  │  ✓ Column additions and removals                                   │   │
│  │  ✓ Column type changes (usually)                                   │   │
│  │  ✓ Nullable changes                                                │   │
│  │  ✓ Index additions and removals                                    │   │
│  │  ✓ Foreign key additions and removals                              │   │
│  │  ✓ Unique constraint changes                                       │   │
│  │  ✓ Primary key changes                                             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ✗ WHAT AUTOGENERATE DOESN'T DETECT:                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ✗ Table renames (sees as drop + create)                           │   │
│  │  ✗ Column renames (sees as drop + add)                             │   │
│  │  ✗ Constraint renames                                              │   │
│  │  ✗ Data migrations (moving data between columns)                   │   │
│  │  ✗ Some server defaults                                            │   │
│  │  ✗ Enum value changes                                              │   │
│  │  ✗ Custom types                                                    │   │
│  │  ✗ Triggers, stored procedures, views                              │   │
│  │                                                                     │   │
│  │  FOR THESE: Create manual migration and write upgrade/downgrade    │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MANUAL MIGRATION EXAMPLE (Column Rename):                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Don't use autogenerate for renames!                             │   │
│  │  # alembic revision -m "Rename name to full_name"                  │   │
│  │                                                                     │   │
│  │  def upgrade():                                                    │   │
│  │      # Rename column (preserves data)                              │   │
│  │      op.alter_column('users', 'name',                              │   │
│  │                      new_column_name='full_name')                  │   │
│  │                                                                     │   │
│  │  def downgrade():                                                  │   │
│  │      op.alter_column('users', 'full_name',                         │   │
│  │                      new_column_name='name')                       │   │
│  │                                                                     │   │
│  │  # Compare to what autogenerate would do (WRONG - loses data):     │   │
│  │  # def upgrade():                                                  │   │
│  │  #     op.drop_column('users', 'name')          # Data lost!      │   │
│  │  #     op.add_column('users', sa.Column('full_name', ...))        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ALWAYS REVIEW AUTOGENERATED MIGRATIONS!                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. Run: alembic revision --autogenerate -m "Description"          │   │
│  │  2. Open the generated file                                        │   │
│  │  3. Review upgrade() and downgrade()                               │   │
│  │  4. Check for:                                                     │   │
│  │     - Unintended drops (data loss!)                                │   │
│  │     - Missing operations                                           │   │
│  │     - Wrong order of operations                                    │   │
│  │  5. Test: alembic upgrade head                                     │   │
│  │  6. Verify: Check database schema                                  │   │
│  │  7. Test downgrade: alembic downgrade -1, then upgrade again       │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 12: Migration Best Practices

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ALEMBIC BEST PRACTICES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. ONE LOGICAL CHANGE PER MIGRATION                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ✗ BAD: "Update schema" (too vague, multiple changes)              │   │
│  │                                                                     │   │
│  │  ✓ GOOD:                                                           │   │
│  │    - "Add users table"                                             │   │
│  │    - "Add email index to users"                                    │   │
│  │    - "Add posts table with user FK"                                │   │
│  │                                                                     │   │
│  │  WHY: Easier to rollback, review, and understand                   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. ALWAYS IMPLEMENT DOWNGRADE                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  def upgrade():                                                    │   │
│  │      op.add_column('users', sa.Column('avatar_url', sa.String))    │   │
│  │                                                                     │   │
│  │  def downgrade():                                                  │   │
│  │      op.drop_column('users', 'avatar_url')                         │   │
│  │                                                                     │   │
│  │  WHY: Need to rollback failed deployments!                         │   │
│  │                                                                     │   │
│  │  EXCEPTION: Truly irreversible migrations (document with comment)  │   │
│  │  def downgrade():                                                  │   │
│  │      # Cannot downgrade: data was transformed/deleted              │   │
│  │      raise NotImplementedError("Cannot reverse data deletion")     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. SAFE DEPLOYMENT PATTERNS                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ADDING A COLUMN (Safe - no downtime)                              │   │
│  │  ────────────────────────────────────                              │   │
│  │  1. Add column as nullable (or with default)                       │   │
│  │  2. Deploy new code that writes to column                          │   │
│  │  3. Backfill existing rows                                         │   │
│  │  4. Add NOT NULL constraint (if needed)                            │   │
│  │                                                                     │   │
│  │  REMOVING A COLUMN (Safe - phased approach)                        │   │
│  │  ──────────────────────────────────────────                        │   │
│  │  1. Deploy code that stops reading the column                      │   │
│  │  2. Deploy code that stops writing to the column                   │   │
│  │  3. Remove column in migration                                     │   │
│  │                                                                     │   │
│  │  RENAMING A COLUMN (Avoid if possible!)                            │   │
│  │  ──────────────────────────────────────                            │   │
│  │  If you must:                                                      │   │
│  │  1. Add new column                                                 │   │
│  │  2. Copy data                                                      │   │
│  │  3. Update code to use new column                                  │   │
│  │  4. Drop old column                                                │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  4. TESTING MIGRATIONS                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Test script for CI/CD                                           │   │
│  │  #!/bin/bash                                                       │   │
│  │  set -e                                                            │   │
│  │                                                                     │   │
│  │  # Start from clean slate                                          │   │
│  │  alembic downgrade base                                            │   │
│  │                                                                     │   │
│  │  # Apply all migrations                                            │   │
│  │  alembic upgrade head                                              │   │
│  │                                                                     │   │
│  │  # Test rollback of last migration                                 │   │
│  │  alembic downgrade -1                                              │   │
│  │  alembic upgrade head                                              │   │
│  │                                                                     │   │
│  │  echo "Migrations OK!"                                             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  5. TEAM WORKFLOW                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                                                             │   │   │
│  │  │  Developer A                Developer B                     │   │   │
│  │  │      │                          │                           │   │   │
│  │  │  Creates migration          Creates migration               │   │   │
│  │  │  rev_abc123                 rev_def456                      │   │   │
│  │  │      │                          │                           │   │   │
│  │  │      └──────────┬───────────────┘                           │   │   │
│  │  │                 │                                           │   │   │
│  │  │            Both merge to main                               │   │   │
│  │  │                 │                                           │   │   │
│  │  │          CONFLICT! Both have                                │   │   │
│  │  │          down_revision = prev                               │   │   │
│  │  │                 │                                           │   │   │
│  │  │          Solution: One must                                 │   │   │
│  │  │          update down_revision                               │   │   │
│  │  │                                                             │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  MERGE CONFLICT RESOLUTION:                                        │   │
│  │  1. Pull latest main                                               │   │
│  │  2. Note the current head revision                                 │   │
│  │  3. Update your migration's down_revision to that head             │   │
│  │  4. Test: alembic upgrade head                                     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 13: Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FASTAPI & ALEMBIC QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FASTAPI ESSENTIALS:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Basic route                                                     │   │
│  │  @app.get("/items/{item_id}")                                      │   │
│  │  async def get_item(item_id: int, q: str = None):                  │   │
│  │      return {"item_id": item_id, "q": q}                           │   │
│  │                                                                     │   │
│  │  # With Pydantic validation                                        │   │
│  │  @app.post("/items", response_model=ItemResponse)                  │   │
│  │  async def create_item(item: ItemCreate):                          │   │
│  │      return item                                                   │   │
│  │                                                                     │   │
│  │  # With dependency injection                                       │   │
│  │  @app.get("/users/me")                                             │   │
│  │  async def get_me(user: User = Depends(get_current_user)):         │   │
│  │      return user                                                   │   │
│  │                                                                     │   │
│  │  # HTTP errors                                                     │   │
│  │  raise HTTPException(status_code=404, detail="Not found")          │   │
│  │                                                                     │   │
│  │  # Background tasks                                                │   │
│  │  @app.post("/send-email")                                          │   │
│  │  async def send_email(bg: BackgroundTasks):                        │   │
│  │      bg.add_task(send_email_task, email)                           │   │
│  │      return {"status": "queued"}                                   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ALEMBIC COMMANDS:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  alembic init alembic              # Initialize                    │   │
│  │  alembic revision --autogenerate -m "msg"  # Create migration      │   │
│  │  alembic revision -m "msg"         # Empty migration               │   │
│  │  alembic upgrade head              # Apply all                     │   │
│  │  alembic upgrade +1                # Apply one                     │   │
│  │  alembic downgrade -1              # Rollback one                  │   │
│  │  alembic downgrade base            # Rollback all                  │   │
│  │  alembic current                   # Show current                  │   │
│  │  alembic history                   # Show history                  │   │
│  │  alembic heads                     # Show latest                   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PYDANTIC V2 PATTERNS:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  from pydantic import BaseModel, Field, ConfigDict                 │   │
│  │                                                                     │   │
│  │  class UserBase(BaseModel):                                        │   │
│  │      email: str                                                    │   │
│  │      name: str = Field(..., min_length=1, max_length=100)         │   │
│  │                                                                     │   │
│  │  class UserCreate(UserBase):                                       │   │
│  │      password: str                                                 │   │
│  │                                                                     │   │
│  │  class UserResponse(UserBase):                                     │   │
│  │      id: int                                                       │   │
│  │      model_config = ConfigDict(from_attributes=True)               │   │
│  │                                                                     │   │
│  │  # Usage                                                           │   │
│  │  user_dict = user.model_dump()                                     │   │
│  │  user_dict = user.model_dump(exclude_unset=True)  # PATCH          │   │
│  │  user = UserResponse.model_validate(db_user)      # From ORM       │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DATABASE SESSION PATTERN:                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  # Sync (SQLAlchemy 1.x style)                                     │   │
│  │  def get_db():                                                     │   │
│  │      db = SessionLocal()                                           │   │
│  │      try:                                                          │   │
│  │          yield db                                                  │   │
│  │      finally:                                                      │   │
│  │          db.close()                                                │   │
│  │                                                                     │   │
│  │  # Async (SQLAlchemy 2.x)                                          │   │
│  │  async def get_db():                                               │   │
│  │      async with async_session() as session:                        │   │
│  │          yield session                                             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMON ALEMBIC OPERATIONS:                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  op.create_table('name', Column(...), Column(...))                 │   │
│  │  op.drop_table('name')                                             │   │
│  │  op.add_column('table', Column('name', Type))                      │   │
│  │  op.drop_column('table', 'column')                                 │   │
│  │  op.alter_column('table', 'col', type_=NewType)                    │   │
│  │  op.create_index('ix_name', 'table', ['column'])                   │   │
│  │  op.create_foreign_key('fk', 'src', 'tgt', ['col'], ['col'])       │   │
│  │  op.execute("RAW SQL")                                             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 14: Learning Path

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FASTAPI & ALEMBIC LEARNING PATH                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PHASE 1: FASTAPI BASICS (Week 1)                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Create basic CRUD endpoints                                      │   │
│  │ □ Understand path parameters and query parameters                  │   │
│  │ □ Use Pydantic models for request/response                         │   │
│  │ □ Handle errors with HTTPException                                 │   │
│  │ □ Explore automatic OpenAPI docs (/docs)                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 2: DEPENDENCY INJECTION (Week 2)                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Create database session dependency                               │   │
│  │ □ Build authentication dependency                                  │   │
│  │ □ Understand yield dependencies (cleanup)                          │   │
│  │ □ Create parameterized dependencies                                │   │
│  │ □ Build service layer with DI                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 3: DATABASE WITH SQLALCHEMY (Week 3)                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Set up SQLAlchemy with FastAPI                                   │   │
│  │ □ Create models with relationships                                 │   │
│  │ □ Implement CRUD operations                                        │   │
│  │ □ Handle database transactions                                     │   │
│  │ □ Initialize Alembic in project                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 4: ALEMBIC MIGRATIONS (Week 4)                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Configure alembic env.py properly                                │   │
│  │ □ Generate autogenerate migrations                                 │   │
│  │ □ Write manual migrations                                          │   │
│  │ □ Practice upgrade/downgrade                                       │   │
│  │ □ Handle migration conflicts                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 5: AUTHENTICATION & SECURITY (Week 5)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Implement JWT authentication                                     │   │
│  │ □ Create login endpoint with OAuth2                                │   │
│  │ □ Protect routes with dependencies                                 │   │
│  │ □ Implement role-based access control                              │   │
│  │ □ Handle password hashing (bcrypt)                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 6: TESTING (Week 6)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Set up pytest with TestClient                                    │   │
│  │ □ Create test database fixtures                                    │   │
│  │ □ Override dependencies in tests                                   │   │
│  │ □ Test authentication flows                                        │   │
│  │ □ Test migrations (upgrade/downgrade)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 7: PRODUCTION READINESS (Week 7-8)                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Async database operations (SQLAlchemy 2.0)                       │   │
│  │ □ Background tasks                                                 │   │
│  │ □ Middleware (CORS, logging, timing)                               │   │
│  │ □ Configuration with pydantic-settings                             │   │
│  │ □ Docker containerization                                          │   │
│  │ □ CI/CD pipeline with migration testing                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BUILD PROJECTS:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. Todo API                                                       │   │
│  │     - CRUD operations, user auth, SQLite                           │   │
│  │                                                                     │   │
│  │  2. Blog API                                                       │   │
│  │     - Users, posts, comments, tags                                 │   │
│  │     - Relationships, pagination, search                            │   │
│  │                                                                     │   │
│  │  3. E-commerce API                                                 │   │
│  │     - Products, orders, payments                                   │   │
│  │     - Complex migrations, data migrations                          │   │
│  │                                                                     │   │
│  │  4. Real-time Chat                                                 │   │
│  │     - WebSockets, async, Redis                                     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

This mental models guide covers:

**FastAPI:**
1. **Request Lifecycle** - Complete flow from request to response
2. **Dependency Injection** - Patterns, types, and common use cases
3. **Pydantic Models** - Validation, schemas, response models
4. **Async/Await** - When to use sync vs async
5. **Project Structure** - Recommended folder organization
6. **Authentication** - JWT + OAuth2 flow
7. **Testing** - TestClient, fixtures, dependency overrides

**Alembic:**
8. **Migration Concepts** - Revision chain, commands
9. **Configuration** - env.py setup, target_metadata
10. **Migration Files** - Structure, common operations
11. **Autogenerate** - What it can and can't detect
12. **Best Practices** - Safe deployment, team workflow
13. **Quick Reference** - Essential commands and patterns
14. **Learning Path** - 8-week progression with projects
