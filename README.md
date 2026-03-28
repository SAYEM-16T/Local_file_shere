আমি তোমার আগের final build prompt, merged PostgreSQL/app-server summary, schema note, S3 note, আর PostgreSQL setup note follow করে এই implementation scaffold দিচ্ছি। Stack choice হিসেবে আমি FastAPI-এর APIRouter + JWT pattern, Vite React frontend, Boto3 presigned S3 access, আর Alembic migrations follow করছি—এগুলো official docs-এর recommended building blocks-এর সাথে align করে।       

1. Complete project overview

এই projectটা একটি modular monolith Library Management System:

Backend: FastAPI

Frontend: React + Vite + TypeScript

Database: PostgreSQL

File storage: AWS S3

Auth: JWT Bearer access token

Architecture style: modular monolith, future microservice-friendly


Business rules enforced

বই hard delete হবে না, default behavior হবে archive / inactive

normal user:

register/login করতে পারবে

বই দেখতে, search/filter করতে, detail দেখতে, download করতে পারবে

PDF book upload করতে পারবে

নিজের profile/password update করতে পারবে


admin:

বই update/archive/restore করতে পারবে

category manage করতে পারবে

users manage করতে পারবে

dashboard metrics দেখতে পারবে


email verification, forgot password, reset password, initial admin forced password change flow included

S3 private bucket strategy ধরে metadata PostgreSQL-এ থাকবে, file S3-তে থাকবে


Assumptions I am locking

email = primary login identifier

username = unique profile identity

real SMTP/SES-এর বদলে development-safe console email backend use করা হয়েছে, যাতে verification/reset URL log হয় এবং response-এ development mode-এ পাওয়া যায়

existing PostgreSQL/S3 infra ready, so code directly env-based integration use করবে



---

2. Full project tree structure

library-management-system/
├── .gitignore
├── backend/
│   ├── .env.example
│   ├── alembic.ini
│   ├── requirements.txt
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── api/
│   │   │   ├── deps.py
│   │   │   └── v1/
│   │   │       ├── api.py
│   │   │       └── endpoints/
│   │   │           ├── admin.py
│   │   │           ├── auth.py
│   │   │           ├── books.py
│   │   │           ├── categories.py
│   │   │           └── users.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   ├── database.py
│   │   │   └── security.py
│   │   ├── db/
│   │   │   ├── base.py
│   │   │   └── seed_admin.py
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── base.py
│   │   │   ├── book.py
│   │   │   ├── category.py
│   │   │   ├── download.py
│   │   │   ├── enums.py
│   │   │   ├── token.py
│   │   │   └── user.py
│   │   ├── repositories/
│   │   │   ├── book_repository.py
│   │   │   ├── category_repository.py
│   │   │   ├── token_repository.py
│   │   │   └── user_repository.py
│   │   ├── schemas/
│   │   │   ├── auth.py
│   │   │   ├── book.py
│   │   │   ├── category.py
│   │   │   ├── common.py
│   │   │   └── user.py
│   │   └── services/
│   │       ├── auth_service.py
│   │       ├── book_service.py
│   │       ├── category_service.py
│   │       ├── notification_service.py
│   │       ├── storage_service.py
│   │       └── user_service.py
│   └── migrations/
│       ├── env.py
│       └── versions/
│           └── 0001_initial_schema.py
└── frontend/
    ├── .env.example
    ├── index.html
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts
    └── src/
        ├── App.tsx
        ├── main.tsx
        ├── vite-env.d.ts
        ├── api/
        │   ├── admin.ts
        │   ├── auth.ts
        │   ├── books.ts
        │   ├── categories.ts
        │   ├── client.ts
        │   └── users.ts
        ├── components/
        │   ├── books/
        │   │   ├── BookCard.tsx
        │   │   └── BookFilters.tsx
        │   ├── common/
        │   │   ├── EmptyState.tsx
        │   │   ├── LoadingState.tsx
        │   │   ├── PageHeader.tsx
        │   │   ├── ProtectedRoute.tsx
        │   │   └── RoleRoute.tsx
        │   └── layout/
        │       └── AppLayout.tsx
        ├── context/
        │   └── AuthContext.tsx
        ├── hooks/
        │   └── useAuth.ts
        ├── pages/
        │   ├── AddBookPage.tsx
        │   ├── AdminBooksPage.tsx
        │   ├── AdminDashboardPage.tsx
        │   ├── ArchivedBooksPage.tsx
        │   ├── BookDetailPage.tsx
        │   ├── BooksPage.tsx
        │   ├── कै? no
        │   ├── CategoryManagementPage.tsx
        │   ├── ChangePasswordPage.tsx
        │   ├── DashboardPage.tsx
        │   ├── ForgotPasswordPage.tsx
        │   ├── LoginPage.tsx
        │   ├── NotFoundPage.tsx
        │   ├── ProfilePage.tsx
        │   ├── RegisterPage.tsx
        │   ├── ResetPasswordPage.tsx
        │   ├── UserManagementPage.tsx
        │   └── VerifyEmailPage.tsx
        ├── styles/
        │   └── global.css
        └── types/
            ├── auth.ts
            ├── book.ts
            ├── category.ts
            ├── common.ts
            └── user.ts


---

3. Backend folder structure and all backend files with full code

File: backend/requirements.txt

fastapi==0.115.0
uvicorn[standard]==0.30.6
sqlalchemy==2.0.35
psycopg[binary]==3.2.1
alembic==1.13.2
pydantic-settings==2.6.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
email-validator==2.2.0
python-multipart==0.0.9
boto3==1.35.21

File: backend/app/init.py

__all__ = ["main"]

File: backend/app/main.py

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.v1.api import api_router
from app.core.config import settings
from app.core.database import SessionLocal
from app.db.seed_admin import seed_initial_admin

logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    db = SessionLocal()
    try:
        seed_initial_admin(db)
    except Exception as exc:  # pragma: no cover
        logger.warning("Initial admin bootstrap skipped: %s", exc)
    finally:
        db.close()
    yield


app = FastAPI(
    title=settings.APP_NAME,
    version="1.0.0",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.BACKEND_CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(api_router, prefix=settings.API_V1_STR)


@app.get("/health", tags=["health"])
def health_check():
    return {"status": "ok", "environment": settings.APP_ENV}

File: backend/app/core/config.py

from functools import lru_cache

from pydantic import EmailStr, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True,
        extra="ignore",
    )

    APP_NAME: str = "Library Management System"
    APP_ENV: str = "development"
    API_V1_STR: str = "/api/v1"
    FRONTEND_URL: str = "http://localhost:5173"
    BACKEND_CORS_ORIGINS: list[str] | str = ["http://localhost:5173"]

    DATABASE_URL: str

    JWT_SECRET_KEY: str
    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60

    AWS_REGION: str
    S3_BUCKET_NAME: str
    S3_PREFIX: str = "books/"
    S3_PRESIGNED_URL_EXPIRE_SECONDS: int = 900
    MAX_UPLOAD_SIZE_MB: int = 25

    INITIAL_ADMIN_EMAIL: EmailStr = "admin@example.com"
    INITIAL_ADMIN_USERNAME: str = "admin"
    INITIAL_ADMIN_PASSWORD: str = "ChangeMe123!"
    INITIAL_ADMIN_FULL_NAME: str = "System Administrator"

    EMAIL_SENDER_NAME: str = "Library Management System"
    EMAIL_SENDER_ADDRESS: EmailStr = "noreply@example.com"

    @field_validator("BACKEND_CORS_ORIGINS", mode="before")
    @classmethod
    def parse_cors_origins(cls, value: str | list[str]) -> list[str]:
        if isinstance(value, str):
            return [item.strip() for item in value.split(",") if item.strip()]
        return value

    @field_validator("S3_PREFIX")
    @classmethod
    def normalize_s3_prefix(cls, value: str) -> str:
        value = value.strip().strip("/")
        if not value:
            return ""
        return f"{value}/"

    @property
    def is_development(self) -> bool:
        return self.APP_ENV.lower() in {"development", "dev", "local"}


@lru_cache
def get_settings() -> Settings:
    return Settings()


settings = get_settings()

File: backend/app/core/database.py

from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from app.core.config import settings

engine = create_engine(
    settings.DATABASE_URL,
    pool_pre_ping=True,
)

SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,
    autocommit=False,
    expire_on_commit=False,
)


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

File: backend/app/core/security.py

import hashlib
import secrets
from datetime import datetime, timedelta, timezone
from typing import Any

from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from passlib.context import CryptContext

from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl=f"{settings.API_V1_STR}/auth/login")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)


def create_access_token(subject: str, role: str, expires_delta: timedelta | None = None) -> str:
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    payload: dict[str, Any] = {"sub": subject, "role": role, "exp": expire}
    return jwt.encode(payload, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)


def decode_access_token(token: str) -> dict[str, Any]:
    try:
        return jwt.decode(token, settings.JWT_SECRET_KEY, algorithms=[settings.JWT_ALGORITHM])
    except JWTError as exc:  # pragma: no cover
        raise ValueError("Invalid authentication token.") from exc


def generate_random_token() -> str:
    return secrets.token_urlsafe(32)


def hash_token(token: str) -> str:
    return hashlib.sha256(token.encode("utf-8")).hexdigest()

File: backend/app/api/deps.py

from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session

from app.core.database import get_db
from app.core.security import decode_access_token, oauth2_scheme
from app.models.enums import AccountStatus, UserRole
from app.models.user import User
from app.repositories.user_repository import UserRepository


def get_current_user(
    db: Session = Depends(get_db),
    token: str = Depends(oauth2_scheme),
) -> User:
    try:
        payload = decode_access_token(token)
        user_id = int(payload.get("sub"))
    except Exception as exc:  # pragma: no cover
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials.",
        ) from exc

    user = UserRepository(db).get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found.")

    if not user.is_active or user.account_status in {AccountStatus.INACTIVE, AccountStatus.BLOCKED}:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Your account is not allowed to access this resource.",
        )

    return user


def get_current_admin(current_user: User = Depends(get_current_user)) -> User:
    if current_user.role != UserRole.ADMIN:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required.",
        )
    return current_user

File: backend/app/api/v1/api.py

from fastapi import APIRouter

from app.api.v1.endpoints import admin, auth, books, categories, users

api_router = APIRouter()
api_router.include_router(auth.router, prefix="/auth", tags=["auth"])
api_router.include_router(users.router, prefix="/users", tags=["users"])
api_router.include_router(categories.router, prefix="/categories", tags=["categories"])
api_router.include_router(books.router, prefix="/books", tags=["books"])
api_router.include_router(admin.router, prefix="/admin", tags=["admin"])

File: backend/app/api/v1/endpoints/auth.py

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from app.api.deps import get_current_user
from app.core.database import get_db
from app.models.user import User
from app.schemas.auth import (
    ActionResponse,
    AuthTokenResponse,
    ChangePasswordRequest,
    ForgotPasswordRequest,
    LoginRequest,
    RegisterRequest,
    RegisterResponse,
    ResendVerificationRequest,
    ResetPasswordRequest,
    VerifyEmailRequest,
)
from app.services.auth_service import AuthService

router = APIRouter()


@router.post("/register", response_model=RegisterResponse, status_code=status.HTTP_201_CREATED)
def register(payload: RegisterRequest, db: Session = Depends(get_db)):
    user, verification_url = AuthService(db).register(payload)
    return RegisterResponse(
        message="Registration successful. Please verify your email before logging in.",
        user=user,
        verification_url=verification_url,
    )


@router.post("/login", response_model=AuthTokenResponse)
def login(payload: LoginRequest, db: Session = Depends(get_db)):
    return AuthService(db).login(payload)


@router.post("/verify-email", response_model=ActionResponse)
def verify_email(payload: VerifyEmailRequest, db: Session = Depends(get_db)):
    message = AuthService(db).verify_email(payload.token)
    return ActionResponse(message=message)


@router.post("/verify-email/resend", response_model=ActionResponse)
def resend_verification(payload: ResendVerificationRequest, db: Session = Depends(get_db)):
    message, action_url = AuthService(db).resend_verification(payload.email)
    return ActionResponse(message=message, action_url=action_url)


@router.post("/forgot-password", response_model=ActionResponse)
def forgot_password(payload: ForgotPasswordRequest, db: Session = Depends(get_db)):
    message, action_url = AuthService(db).forgot_password(payload.email)
    return ActionResponse(message=message, action_url=action_url)


@router.post("/reset-password", response_model=ActionResponse)
def reset_password(payload: ResetPasswordRequest, db: Session = Depends(get_db)):
    message = AuthService(db).reset_password(payload.token, payload.new_password)
    return ActionResponse(message=message)


@router.post("/change-password", response_model=ActionResponse)
def change_password(
    payload: ChangePasswordRequest,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    message = AuthService(db).change_password(
        current_user=current_user,
        current_password=payload.current_password,
        new_password=payload.new_password,
    )
    return ActionResponse(message=message)

File: backend/app/api/v1/endpoints/users.py

from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from app.api.deps import get_current_user
from app.core.database import get_db
from app.models.user import User
from app.schemas.user import UserPublic, UserSelfUpdate
from app.services.user_service import UserService

router = APIRouter()


@router.get("/me", response_model=UserPublic)
def get_me(current_user: User = Depends(get_current_user)):
    return current_user


@router.patch("/me", response_model=UserPublic)
def update_me(
    payload: UserSelfUpdate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    return UserService(db).update_self(current_user=current_user, payload=payload)

File: backend/app/api/v1/endpoints/categories.py

from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.orm import Session

from app.api.deps import get_current_admin, get_current_user
from app.core.database import get_db
from app.models.user import User
from app.schemas.category import CategoryCreate, CategoryListResponse, CategoryPublic, CategoryUpdate
from app.services.category_service import CategoryService

router = APIRouter()


@router.get("", response_model=CategoryListResponse)
def list_categories(
    include_inactive: bool = Query(False),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    service = CategoryService(db)
    if include_inactive and current_user.role.value != "admin":
        include_inactive = False
    items = service.list_categories(include_inactive=include_inactive)
    return CategoryListResponse(items=items)


@router.post("", response_model=CategoryPublic, status_code=status.HTTP_201_CREATED)
def create_category(
    payload: CategoryCreate,
    db: Session = Depends(get_db),
    _: User = Depends(get_current_admin),
):
    return CategoryService(db).create_category(payload)


@router.patch("/{category_id}", response_model=CategoryPublic)
def update_category(
    category_id: int,
    payload: CategoryUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(get_current_admin),
):
    return CategoryService(db).update_category(category_id=category_id, payload=payload)

File: backend/app/api/v1/endpoints/books.py

from fastapi import APIRouter, Depends, File, Form, Query, Request, UploadFile, status
from sqlalchemy.orm import Session

from app.api.deps import get_current_admin, get_current_user
from app.core.database import get_db
from app.models.enums import BookStatus
from app.models.user import User
from app.schemas.book import BookDownloadResponse, BookListResponse, BookPublic, BookUpdate
from app.services.book_service import BookService

router = APIRouter()


@router.get("", response_model=BookListResponse)
def list_books(
    search: str | None = Query(None),
    author: str | None = Query(None),
    category_id: int | None = Query(None),
    status_filter: BookStatus | None = Query(None, alias="status"),
    include_archived: bool = Query(False),
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    return BookService(db).list_books(
        current_user=current_user,
        search=search,
        author=author,
        category_id=category_id,
        status_filter=status_filter,
        include_archived=include_archived,
        page=page,
        page_size=page_size,
    )


@router.get("/{book_id}", response_model=BookPublic)
def get_book(
    book_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    return BookService(db).get_book(book_id=book_id, current_user=current_user)


@router.post("", response_model=BookPublic, status_code=status.HTTP_201_CREATED)
async def create_book(
    title: str = Form(...),
    author: str = Form(...),
    description: str | None = Form(None),
    category_id: int | None = Form(None),
    file: UploadFile = File(...),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    file_bytes = await file.read()
    return BookService(db).create_book(
        current_user=current_user,
        title=title,
        author=author,
        description=description,
        category_id=category_id,
        file_bytes=file_bytes,
        original_filename=file.filename or "upload.pdf",
        content_type=file.content_type or "application/pdf",
    )


@router.patch("/{book_id}", response_model=BookPublic)
def update_book(
    book_id: int,
    payload: BookUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(get_current_admin),
):
    return BookService(db).update_book(book_id=book_id, payload=payload)


@router.post("/{book_id}/archive", response_model=BookPublic)
def archive_book(
    book_id: int,
    db: Session = Depends(get_db),
    current_admin: User = Depends(get_current_admin),
):
    return BookService(db).archive_book(book_id=book_id, admin_user=current_admin)


@router.post("/{book_id}/restore", response_model=BookPublic)
def restore_book(
    book_id: int,
    db: Session = Depends(get_db),
    current_admin: User = Depends(get_current_admin),
):
    return BookService(db).restore_book(book_id=book_id, admin_user=current_admin)


@router.post("/{book_id}/download", response_model=BookDownloadResponse)
def generate_download_link(
    book_id: int,
    request: Request,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    client_ip = request.client.host if request.client else None
    user_agent = request.headers.get("user-agent")
    return BookService(db).generate_download_link(
        book_id=book_id,
        current_user=current_user,
        client_ip=client_ip,
        user_agent=user_agent,
    )

File: backend/app/api/v1/endpoints/admin.py

from fastapi import APIRouter, Depends, Query
from sqlalchemy.orm import Session

from app.api.deps import get_current_admin
from app.core.database import get_db
from app.models.user import User
from app.schemas.user import AdminUserUpdate, DashboardSummary, UserListResponse, UserPublic
from app.services.book_service import BookService
from app.services.user_service import UserService

router = APIRouter()


@router.get("/dashboard", response_model=DashboardSummary)
def dashboard(
    db: Session = Depends(get_db),
    _: User = Depends(get_current_admin),
):
    return BookService(db).dashboard_summary()


@router.get("/users", response_model=UserListResponse)
def list_users(
    search: str | None = Query(None),
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100),
    db: Session = Depends(get_db),
    _: User = Depends(get_current_admin),
):
    return UserService(db).list_users(search=search, page=page, page_size=page_size)


@router.patch("/users/{user_id}", response_model=UserPublic)
def update_user(
    user_id: int,
    payload: AdminUserUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(get_current_admin),
):
    return UserService(db).admin_update_user(user_id=user_id, payload=payload)

File: backend/app/db/base.py

from app.models.base import Base
from app.models.book import Book
from app.models.category import Category
from app.models.download import Download
from app.models.token import EmailVerificationToken, PasswordResetToken
from app.models.user import User

__all__ = [
    "Base",
    "User",
    "Category",
    "Book",
    "EmailVerificationToken",
    "PasswordResetToken",
    "Download",
]

File: backend/app/db/seed_admin.py

import logging

from sqlalchemy.orm import Session

from app.core.config import settings
from app.core.security import hash_password
from app.models.enums import AccountStatus, UserRole
from app.models.user import User

logger = logging.getLogger(__name__)


def seed_initial_admin(db: Session) -> None:
    existing = (
        db.query(User)
        .filter(
            (User.email == settings.INITIAL_ADMIN_EMAIL)
            | (User.username == settings.INITIAL_ADMIN_USERNAME)
        )
        .first()
    )
    if existing:
        logger.info("Initial admin already exists, skipping bootstrap.")
        return

    admin = User(
        full_name=settings.INITIAL_ADMIN_FULL_NAME,
        username=settings.INITIAL_ADMIN_USERNAME,
        email=settings.INITIAL_ADMIN_EMAIL,
        hashed_password=hash_password(settings.INITIAL_ADMIN_PASSWORD),
        role=UserRole.ADMIN,
        account_status=AccountStatus.ACTIVE,
        is_active=True,
        is_verified=True,
        must_change_password_on_first_login=True,
    )
    db.add(admin)
    db.commit()
    logger.info("Initial admin created successfully.")

File: backend/app/models/base.py

from datetime import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )

File: backend/app/models/enums.py

from enum import Enum


def enum_values(enum_cls: type[Enum]) -> list[str]:
    return [item.value for item in enum_cls]


class UserRole(str, Enum):
    ADMIN = "admin"
    NORMAL_USER = "normal_user"


class AccountStatus(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    BLOCKED = "blocked"


class BookStatus(str, Enum):
    ACTIVE = "active"
    ARCHIVED = "archived"
    INACTIVE = "inactive"

File: backend/app/models/user.py

from datetime import datetime
from typing import TYPE_CHECKING

from sqlalchemy import BigInteger, Boolean, DateTime, Identity, String, Text
from sqlalchemy import Enum as SQLEnum
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin
from app.models.enums import AccountStatus, UserRole, enum_values

if TYPE_CHECKING:
    from app.models.book import Book
    from app.models.download import Download
    from app.models.token import EmailVerificationToken, PasswordResetToken


class User(Base, TimestampMixin):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    full_name: Mapped[str] = mapped_column(String(150), nullable=False)
    username: Mapped[str] = mapped_column(String(50), nullable=False, unique=True, index=True)
    email: Mapped[str] = mapped_column(String(255), nullable=False, unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(Text, nullable=False)
    role: Mapped[UserRole] = mapped_column(
        SQLEnum(UserRole, name="user_role", values_callable=enum_values),
        default=UserRole.NORMAL_USER,
        nullable=False,
    )
    account_status: Mapped[AccountStatus] = mapped_column(
        SQLEnum(AccountStatus, name="account_status", values_callable=enum_values),
        default=AccountStatus.ACTIVE,
        nullable=False,
    )
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    is_verified: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    must_change_password_on_first_login: Mapped[bool] = mapped_column(
        Boolean,
        default=False,
        nullable=False,
    )
    last_login: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    uploaded_books: Mapped[list["Book"]] = relationship(
        "Book",
        foreign_keys="Book.uploaded_by",
        back_populates="uploader",
    )
    archived_books: Mapped[list["Book"]] = relationship(
        "Book",
        foreign_keys="Book.archived_by",
        back_populates="archiver",
    )
    email_verification_tokens: Mapped[list["EmailVerificationToken"]] = relationship(
        "EmailVerificationToken",
        back_populates="user",
        cascade="all, delete-orphan",
    )
    password_reset_tokens: Mapped[list["PasswordResetToken"]] = relationship(
        "PasswordResetToken",
        back_populates="user",
        cascade="all, delete-orphan",
    )
    download_records: Mapped[list["Download"]] = relationship(
        "Download",
        back_populates="user",
    )

File: backend/app/models/category.py

from typing import TYPE_CHECKING

from sqlalchemy import BigInteger, Boolean, Identity, String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin

if TYPE_CHECKING:
    from app.models.book import Book


class Category(Base, TimestampMixin):
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False, unique=True, index=True)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)

    books: Mapped[list["Book"]] = relationship("Book", back_populates="category")

File: backend/app/models/book.py

from datetime import datetime
from typing import TYPE_CHECKING

from sqlalchemy import BigInteger, ForeignKey, Identity, String, Text
from sqlalchemy import Enum as SQLEnum
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin
from app.models.enums import BookStatus, enum_values

if TYPE_CHECKING:
    from app.models.category import Category
    from app.models.download import Download
    from app.models.user import User


class Book(Base, TimestampMixin):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    title: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    author: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    category_id: Mapped[int | None] = mapped_column(
        BigInteger,
        ForeignKey("categories.id", ondelete="SET NULL"),
        nullable=True,
        index=True,
    )
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    file_name: Mapped[str] = mapped_column(String(255), nullable=False)
    storage_key: Mapped[str] = mapped_column(String(500), nullable=False, unique=True)
    mime_type: Mapped[str] = mapped_column(String(100), nullable=False)
    file_size: Mapped[int] = mapped_column(BigInteger, nullable=False)
    uploaded_by: Mapped[int] = mapped_column(
        BigInteger,
        ForeignKey("users.id", ondelete="RESTRICT"),
        nullable=False,
        index=True,
    )
    status: Mapped[BookStatus] = mapped_column(
        SQLEnum(BookStatus, name="book_status", values_callable=enum_values),
        default=BookStatus.ACTIVE,
        nullable=False,
        index=True,
    )
    archived_at: Mapped[datetime | None] = mapped_column(nullable=True)
    archived_by: Mapped[int | None] = mapped_column(
        BigInteger,
        ForeignKey("users.id", ondelete="SET NULL"),
        nullable=True,
    )
    download_count: Mapped[int] = mapped_column(BigInteger, default=0, nullable=False)

    category: Mapped["Category | None"] = relationship("Category", back_populates="books")
    uploader: Mapped["User"] = relationship(
        "User",
        foreign_keys=[uploaded_by],
        back_populates="uploaded_books",
    )
    archiver: Mapped["User | None"] = relationship(
        "User",
        foreign_keys=[archived_by],
        back_populates="archived_books",
    )
    downloads: Mapped[list["Download"]] = relationship(
        "Download",
        back_populates="book",
        cascade="all, delete-orphan",
    )

File: backend/app/models/token.py

from datetime import datetime
from typing import TYPE_CHECKING

from sqlalchemy import BigInteger, ForeignKey, Identity, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base

if TYPE_CHECKING:
    from app.models.user import User


class EmailVerificationToken(Base):
    __tablename__ = "email_verification_tokens"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    user_id: Mapped[int] = mapped_column(
        BigInteger,
        ForeignKey("users.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    token_hash: Mapped[str] = mapped_column(Text, nullable=False, unique=True, index=True)
    expires_at: Mapped[datetime] = mapped_column(nullable=False)
    used_at: Mapped[datetime | None] = mapped_column(nullable=True)
    created_at: Mapped[datetime] = mapped_column(nullable=False)

    user: Mapped["User"] = relationship("User", back_populates="email_verification_tokens")


class PasswordResetToken(Base):
    __tablename__ = "password_reset_tokens"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    user_id: Mapped[int] = mapped_column(
        BigInteger,
        ForeignKey("users.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    token_hash: Mapped[str] = mapped_column(Text, nullable=False, unique=True, index=True)
    expires_at: Mapped[datetime] = mapped_column(nullable=False)
    used_at: Mapped[datetime | None] = mapped_column(nullable=True)
    created_at: Mapped[datetime] = mapped_column(nullable=False)

    user: Mapped["User"] = relationship("User", back_populates="password_reset_tokens")

File: backend/app/models/download.py

from datetime import datetime
from typing import TYPE_CHECKING

from sqlalchemy import BigInteger, ForeignKey, Identity, Text
from sqlalchemy.dialects.postgresql import INET
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base

if TYPE_CHECKING:
    from app.models.book import Book
    from app.models.user import User


class Download(Base):
    __tablename__ = "downloads"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    book_id: Mapped[int] = mapped_column(
        BigInteger,
        ForeignKey("books.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    user_id: Mapped[int | None] = mapped_column(
        BigInteger,
        ForeignKey("users.id", ondelete="SET NULL"),
        nullable=True,
        index=True,
    )
    downloaded_at: Mapped[datetime] = mapped_column(nullable=False)
    ip_address: Mapped[str | None] = mapped_column(INET, nullable=True)
    user_agent: Mapped[str | None] = mapped_column(Text, nullable=True)

    book: Mapped["Book"] = relationship("Book", back_populates="downloads")
    user: Mapped["User | None"] = relationship("User", back_populates="download_records")

File: backend/app/models/init.py

from app.models.book import Book
from app.models.category import Category
from app.models.download import Download
from app.models.token import EmailVerificationToken, PasswordResetToken
from app.models.user import User

__all__ = [
    "User",
    "Category",
    "Book",
    "EmailVerificationToken",
    "PasswordResetToken",
    "Download",
]

File: backend/app/repositories/user_repository.py

from sqlalchemy import func, or_, select
from sqlalchemy.orm import Session

from app.models.user import User


class UserRepository:
    def __init__(self, db: Session):
        self.db = db

    def get_by_id(self, user_id: int) -> User | None:
        return self.db.get(User, user_id)

    def get_by_email(self, email: str) -> User | None:
        return self.db.execute(select(User).where(User.email == email)).scalar_one_or_none()

    def get_by_username(self, username: str) -> User | None:
        return self.db.execute(select(User).where(User.username == username)).scalar_one_or_none()

    def add(self, user: User) -> User:
        self.db.add(user)
        return user

    def list_users(self, search: str | None, page: int, page_size: int) -> tuple[list[User], int]:
        stmt = select(User).order_by(User.created_at.desc())

        if search:
            pattern = f"%{search.strip()}%"
            stmt = stmt.where(
                or_(
                    User.full_name.ilike(pattern),
                    User.username.ilike(pattern),
                    User.email.ilike(pattern),
                )
            )

        total_stmt = select(func.count()).select_from(stmt.order_by(None).subquery())
        total = self.db.execute(total_stmt).scalar_one()

        items = self.db.execute(
            stmt.offset((page - 1) * page_size).limit(page_size)
        ).scalars().all()
        return items, total

    def count_all(self) -> int:
        return self.db.execute(select(func.count(User.id))).scalar_one()

File: backend/app/repositories/category_repository.py

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.category import Category


class CategoryRepository:
    def __init__(self, db: Session):
        self.db = db

    def get_by_id(self, category_id: int) -> Category | None:
        return self.db.get(Category, category_id)

    def get_by_name(self, name: str) -> Category | None:
        return self.db.execute(select(Category).where(Category.name == name)).scalar_one_or_none()

    def list_categories(self, include_inactive: bool = False) -> list[Category]:
        stmt = select(Category).order_by(Category.name.asc())
        if not include_inactive:
            stmt = stmt.where(Category.is_active.is_(True))
        return self.db.execute(stmt).scalars().all()

    def add(self, category: Category) -> Category:
        self.db.add(category)
        return category

File: backend/app/repositories/book_repository.py

from sqlalchemy import func, or_, select
from sqlalchemy.orm import Session, joinedload

from app.models.book import Book
from app.models.enums import BookStatus


class BookRepository:
    def __init__(self, db: Session):
        self.db = db

    def get_by_id(self, book_id: int) -> Book | None:
        stmt = (
            select(Book)
            .options(
                joinedload(Book.category),
                joinedload(Book.uploader),
                joinedload(Book.archiver),
            )
            .where(Book.id == book_id)
        )
        return self.db.execute(stmt).scalar_one_or_none()

    def list_books(
        self,
        *,
        search: str | None,
        author: str | None,
        category_id: int | None,
        status_filter: BookStatus | None,
        include_archived: bool,
        page: int,
        page_size: int,
    ) -> tuple[list[Book], int]:
        stmt = (
            select(Book)
            .options(
                joinedload(Book.category),
                joinedload(Book.uploader),
                joinedload(Book.archiver),
            )
            .order_by(Book.created_at.desc())
        )

        if search:
            pattern = f"%{search.strip()}%"
            stmt = stmt.where(or_(Book.title.ilike(pattern), Book.author.ilike(pattern)))

        if author:
            stmt = stmt.where(Book.author.ilike(f"%{author.strip()}%"))

        if category_id is not None:
            stmt = stmt.where(Book.category_id == category_id)

        if status_filter is not None:
            stmt = stmt.where(Book.status == status_filter)
        elif not include_archived:
            stmt = stmt.where(Book.status == BookStatus.ACTIVE)

        total_stmt = select(func.count()).select_from(stmt.order_by(None).subquery())
        total = self.db.execute(total_stmt).scalar_one()

        items = self.db.execute(
            stmt.offset((page - 1) * page_size).limit(page_size)
        ).scalars().unique().all()
        return items, total

    def add(self, book: Book) -> Book:
        self.db.add(book)
        return book

    def count_all(self) -> int:
        return self.db.execute(select(func.count(Book.id))).scalar_one()

    def count_by_status(self, status_filter: BookStatus) -> int:
        return self.db.execute(
            select(func.count(Book.id)).where(Book.status == status_filter)
        ).scalar_one()

    def total_download_count(self) -> int:
        return self.db.execute(select(func.coalesce(func.sum(Book.download_count), 0))).scalar_one()

File: backend/app/repositories/token_repository.py

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.token import EmailVerificationToken, PasswordResetToken


class TokenRepository:
    def __init__(self, db: Session):
        self.db = db

    def add_email_token(self, token: EmailVerificationToken) -> EmailVerificationToken:
        self.db.add(token)
        return token

    def add_reset_token(self, token: PasswordResetToken) -> PasswordResetToken:
        self.db.add(token)
        return token

    def get_email_token_by_hash(self, token_hash: str) -> EmailVerificationToken | None:
        return self.db.execute(
            select(EmailVerificationToken).where(EmailVerificationToken.token_hash == token_hash)
        ).scalar_one_or_none()

    def get_reset_token_by_hash(self, token_hash: str) -> PasswordResetToken | None:
        return self.db.execute(
            select(PasswordResetToken).where(PasswordResetToken.token_hash == token_hash)
        ).scalar_one_or_none()

File: backend/app/schemas/common.py

from pydantic import BaseModel, ConfigDict


class MessageResponse(BaseModel):
    message: str


class PaginationMeta(BaseModel):
    page: int
    page_size: int
    total: int
    total_pages: int


class ORMBaseModel(BaseModel):
    model_config = ConfigDict(from_attributes=True)

File: backend/app/schemas/user.py

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field

from app.models.enums import AccountStatus, UserRole
from app.schemas.common import PaginationMeta


class UserBrief(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    full_name: str
    username: str
    role: UserRole


class UserPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    full_name: str
    username: str
    email: str
    role: UserRole
    account_status: AccountStatus
    is_active: bool
    is_verified: bool
    must_change_password_on_first_login: bool
    created_at: datetime
    updated_at: datetime
    last_login: datetime | None = None


class UserSelfUpdate(BaseModel):
    full_name: str | None = Field(default=None, min_length=2, max_length=150)
    username: str | None = Field(default=None, min_length=3, max_length=50)


class AdminUserUpdate(BaseModel):
    role: UserRole | None = None
    account_status: AccountStatus | None = None
    is_active: bool | None = None
    is_verified: bool | None = None
    must_change_password_on_first_login: bool | None = None


class UserListResponse(BaseModel):
    items: list[UserPublic]
    meta: PaginationMeta


class DashboardSummary(BaseModel):
    total_users: int
    total_books: int
    active_books: int
    archived_books: int
    total_downloads: int

File: backend/app/schemas/auth.py

from pydantic import BaseModel, EmailStr, Field

from app.schemas.user import UserPublic


class RegisterRequest(BaseModel):
    full_name: str = Field(min_length=2, max_length=150)
    username: str = Field(min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)


class RegisterResponse(BaseModel):
    message: str
    user: UserPublic
    verification_url: str | None = None


class LoginRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)


class AuthTokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    user: UserPublic


class VerifyEmailRequest(BaseModel):
    token: str


class ResendVerificationRequest(BaseModel):
    email: EmailStr


class ForgotPasswordRequest(BaseModel):
    email: EmailStr


class ResetPasswordRequest(BaseModel):
    token: str
    new_password: str = Field(min_length=8, max_length=128)


class ChangePasswordRequest(BaseModel):
    current_password: str = Field(min_length=8, max_length=128)
    new_password: str = Field(min_length=8, max_length=128)


class ActionResponse(BaseModel):
    message: str
    action_url: str | None = None

File: backend/app/schemas/category.py

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class CategoryCreate(BaseModel):
    name: str = Field(min_length=2, max_length=100)
    description: str | None = Field(default=None, max_length=500)


class CategoryUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=2, max_length=100)
    description: str | None = Field(default=None, max_length=500)
    is_active: bool | None = None


class CategoryPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str
    description: str | None = None
    is_active: bool
    created_at: datetime
    updated_at: datetime


class CategoryListResponse(BaseModel):
    items: list[CategoryPublic]

File: backend/app/schemas/book.py

from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field

from app.models.enums import BookStatus
from app.schemas.category import CategoryPublic
from app.schemas.common import PaginationMeta
from app.schemas.user import UserBrief


class BookPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    author: str
    description: str | None = None
    file_name: str
    storage_key: str
    mime_type: str
    file_size: int
    status: BookStatus
    download_count: int
    created_at: datetime
    updated_at: datetime
    archived_at: datetime | None = None
    category: CategoryPublic | None = None
    uploader: UserBrief
    archiver: UserBrief | None = None


class BookUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=2, max_length=255)
    author: str | None = Field(default=None, min_length=2, max_length=255)
    description: str | None = Field(default=None, max_length=5000)
    category_id: int | None = None


class BookListResponse(BaseModel):
    items: list[BookPublic]
    meta: PaginationMeta


class BookDownloadResponse(BaseModel):
    download_url: str
    expires_in_seconds: int

File: backend/app/services/notification_service.py

import logging

from app.core.config import settings

logger = logging.getLogger(__name__)


class NotificationService:
    def build_verification_url(self, raw_token: str) -> str:
        return f"{settings.FRONTEND_URL}/verify-email?token={raw_token}"

    def build_reset_url(self, raw_token: str) -> str:
        return f"{settings.FRONTEND_URL}/reset-password?token={raw_token}"

    def send_verification_email(self, recipient: str, recipient_name: str, raw_token: str) -> str | None:
        url = self.build_verification_url(raw_token)
        logger.info(
            "DEV EMAIL | verification | to=%s | name=%s | url=%s",
            recipient,
            recipient_name,
            url,
        )
        return url if settings.is_development else None

    def send_password_reset_email(self, recipient: str, recipient_name: str, raw_token: str) -> str | None:
        url = self.build_reset_url(raw_token)
        logger.info(
            "DEV EMAIL | password-reset | to=%s | name=%s | url=%s",
            recipient,
            recipient_name,
            url,
        )
        return url if settings.is_development else None

File: backend/app/services/storage_service.py

from pathlib import Path
from uuid import uuid4

import boto3
from botocore.client import BaseClient

from app.core.config import settings


class StorageService:
    def __init__(self) -> None:
        self.client: BaseClient = boto3.client("s3", region_name=settings.AWS_REGION)

    def build_storage_key(self, original_filename: str) -> str:
        extension = Path(original_filename).suffix.lower() or ".pdf"
        return f"{settings.S3_PREFIX}{uuid4()}{extension}"

    def upload_pdf(
        self,
        *,
        file_bytes: bytes,
        original_filename: str,
        content_type: str,
    ) -> dict[str, str | int]:
        storage_key = self.build_storage_key(original_filename)
        self.client.put_object(
            Bucket=settings.S3_BUCKET_NAME,
            Key=storage_key,
            Body=file_bytes,
            ContentType=content_type,
            ServerSideEncryption="AES256",
        )
        return {
            "storage_key": storage_key,
            "file_name": original_filename,
            "mime_type": content_type,
            "file_size": len(file_bytes),
        }

    def generate_download_url(self, *, storage_key: str, file_name: str) -> str:
        return self.client.generate_presigned_url(
            ClientMethod="get_object",
            Params={
                "Bucket": settings.S3_BUCKET_NAME,
                "Key": storage_key,
                "ResponseContentDisposition": f'attachment; filename="{file_name}"',
            },
            ExpiresIn=settings.S3_PRESIGNED_URL_EXPIRE_SECONDS,
        )

File: backend/app/services/auth_service.py

from datetime import datetime, timedelta, timezone

from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.core.security import (
    create_access_token,
    generate_random_token,
    hash_password,
    hash_token,
    verify_password,
)
from app.models.enums import AccountStatus, UserRole
from app.models.token import EmailVerificationToken, PasswordResetToken
from app.models.user import User
from app.repositories.token_repository import TokenRepository
from app.repositories.user_repository import UserRepository
from app.schemas.auth import LoginRequest, RegisterRequest
from app.schemas.user import UserPublic
from app.services.notification_service import NotificationService


class AuthService:
    def __init__(self, db: Session):
        self.db = db
        self.users = UserRepository(db)
        self.tokens = TokenRepository(db)
        self.notifications = NotificationService()

    def register(self, payload: RegisterRequest) -> tuple[User, str | None]:
        if self.users.get_by_email(payload.email):
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Email is already registered.")

        if self.users.get_by_username(payload.username):
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Username is already taken.")

        user = User(
            full_name=payload.full_name,
            username=payload.username,
            email=payload.email,
            hashed_password=hash_password(payload.password),
            role=UserRole.NORMAL_USER,
            account_status=AccountStatus.ACTIVE,
            is_active=True,
            is_verified=False,
            must_change_password_on_first_login=False,
        )
        self.users.add(user)
        self.db.flush()

        raw_token = generate_random_token()
        token = EmailVerificationToken(
            user_id=user.id,
            token_hash=hash_token(raw_token),
            expires_at=datetime.now(timezone.utc) + timedelta(hours=24),
            used_at=None,
            created_at=datetime.now(timezone.utc),
        )
        self.tokens.add_email_token(token)
        self.db.commit()
        self.db.refresh(user)

        verification_url = self.notifications.send_verification_email(
            recipient=user.email,
            recipient_name=user.full_name,
            raw_token=raw_token,
        )
        return user, verification_url

    def login(self, payload: LoginRequest):
        user = self.users.get_by_email(payload.email)
        if not user or not verify_password(payload.password, user.hashed_password):
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid email or password.",
            )

        if not user.is_verified:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Please verify your email before logging in.",
            )

        if not user.is_active or user.account_status in {AccountStatus.INACTIVE, AccountStatus.BLOCKED}:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Your account is not allowed to log in.",
            )

        user.last_login = datetime.now(timezone.utc)
        self.db.commit()
        self.db.refresh(user)

        access_token = create_access_token(subject=str(user.id), role=user.role.value)
        return {
            "access_token": access_token,
            "token_type": "bearer",
            "user": UserPublic.model_validate(user),
        }

    def verify_email(self, raw_token: str) -> str:
        token = self.tokens.get_email_token_by_hash(hash_token(raw_token))
        if not token:
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Invalid verification token.")

        if token.used_at is not None:
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="This token has already been used.")

        if token.expires_at < datetime.now(timezone.utc):
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="This token has expired.")

        token.used_at = datetime.now(timezone.utc)
        token.user.is_verified = True
        self.db.commit()
        return "Email verified successfully. You can now log in."

    def resend_verification(self, email: str) -> tuple[str, str | None]:
        user = self.users.get_by_email(email)
        if not user:
            return "If the email exists, a verification link has been sent.", None

        if user.is_verified:
            return "This email is already verified.", None

        raw_token = generate_random_token()
        token = EmailVerificationToken(
            user_id=user.id,
            token_hash=hash_token(raw_token),
            expires_at=datetime.now(timezone.utc) + timedelta(hours=24),
            used_at=None,
            created_at=datetime.now(timezone.utc),
        )
        self.tokens.add_email_token(token)
        self.db.commit()

        verification_url = self.notifications.send_verification_email(
            recipient=user.email,
            recipient_name=user.full_name,
            raw_token=raw_token,
        )
        return "Verification link generated successfully.", verification_url

    def forgot_password(self, email: str) -> tuple[str, str | None]:
        user = self.users.get_by_email(email)
        if not user:
            return "If the email exists, a password reset link has been sent.", None

        raw_token = generate_random_token()
        token = PasswordResetToken(
            user_id=user.id,
            token_hash=hash_token(raw_token),
            expires_at=datetime.now(timezone.utc) + timedelta(hours=1),
            used_at=None,
            created_at=datetime.now(timezone.utc),
        )
        self.tokens.add_reset_token(token)
        self.db.commit()

        reset_url = self.notifications.send_password_reset_email(
            recipient=user.email,
            recipient_name=user.full_name,
            raw_token=raw_token,
        )
        return "Password reset link generated successfully.", reset_url

    def reset_password(self, raw_token: str, new_password: str) -> str:
        token = self.tokens.get_reset_token_by_hash(hash_token(raw_token))
        if not token:
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Invalid reset token.")

        if token.used_at is not None:
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="This token has already been used.")

        if token.expires_at < datetime.now(timezone.utc):
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="This token has expired.")

        token.user.hashed_password = hash_password(new_password)
        token.user.must_change_password_on_first_login = False
        token.used_at = datetime.now(timezone.utc)
        self.db.commit()
        return "Password reset successful. You can now log in with your new password."

    def change_password(self, current_user: User, current_password: str, new_password: str) -> str:
        if not verify_password(current_password, current_user.hashed_password):
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Current password is incorrect.",
            )

        current_user.hashed_password = hash_password(new_password)
        current_user.must_change_password_on_first_login = False
        self.db.commit()
        return "Password changed successfully."

File: backend/app/services/user_service.py

from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.repositories.user_repository import UserRepository
from app.schemas.common import PaginationMeta
from app.schemas.user import AdminUserUpdate, UserListResponse, UserSelfUpdate
from app.models.user import User


class UserService:
    def __init__(self, db: Session):
        self.db = db
        self.users = UserRepository(db)

    def update_self(self, *, current_user: User, payload: UserSelfUpdate) -> User:
        data = payload.model_dump(exclude_unset=True)

        if "username" in data and data["username"] != current_user.username:
            existing = self.users.get_by_username(data["username"])
            if existing:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Username is already taken.",
                )
            current_user.username = data["username"]

        if "full_name" in data:
            current_user.full_name = data["full_name"]

        self.db.commit()
        self.db.refresh(current_user)
        return current_user

    def list_users(self, *, search: str | None, page: int, page_size: int) -> UserListResponse:
        items, total = self.users.list_users(search=search, page=page, page_size=page_size)
        total_pages = (total + page_size - 1) // page_size if total else 1
        return UserListResponse(
            items=items,
            meta=PaginationMeta(
                page=page,
                page_size=page_size,
                total=total,
                total_pages=total_pages,
            ),
        )

    def admin_update_user(self, *, user_id: int, payload: AdminUserUpdate) -> User:
        user = self.users.get_by_id(user_id)
        if not user:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="User not found.")

        data = payload.model_dump(exclude_unset=True)
        for key, value in data.items():
            setattr(user, key, value)

        self.db.commit()
        self.db.refresh(user)
        return user

File: backend/app/services/category_service.py

from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.models.category import Category
from app.repositories.category_repository import CategoryRepository
from app.schemas.category import CategoryCreate, CategoryUpdate


class CategoryService:
    def __init__(self, db: Session):
        self.db = db
        self.categories = CategoryRepository(db)

    def list_categories(self, *, include_inactive: bool) -> list[Category]:
        return self.categories.list_categories(include_inactive=include_inactive)

    def create_category(self, payload: CategoryCreate) -> Category:
        if self.categories.get_by_name(payload.name):
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Category name already exists.",
            )

        category = Category(name=payload.name, description=payload.description, is_active=True)
        self.categories.add(category)
        self.db.commit()
        self.db.refresh(category)
        return category

    def update_category(self, *, category_id: int, payload: CategoryUpdate) -> Category:
        category = self.categories.get_by_id(category_id)
        if not category:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Category not found.")

        data = payload.model_dump(exclude_unset=True)
        if "name" in data and data["name"] != category.name:
            existing = self.categories.get_by_name(data["name"])
            if existing:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail="Category name already exists.",
                )

        for key, value in data.items():
            setattr(category, key, value)

        self.db.commit()
        self.db.refresh(category)
        return category

File: backend/app/services/book_service.py

from datetime import datetime, timezone

from fastapi import HTTPException, status
from sqlalchemy.orm import Session

from app.core.config import settings
from app.models.book import Book
from app.models.download import Download
from app.models.enums import BookStatus, UserRole
from app.models.user import User
from app.repositories.book_repository import BookRepository
from app.repositories.category_repository import CategoryRepository
from app.repositories.user_repository import UserRepository
from app.schemas.book import BookDownloadResponse, BookListResponse, BookUpdate
from app.schemas.common import PaginationMeta
from app.schemas.user import DashboardSummary
from app.services.storage_service import StorageService


class BookService:
    def __init__(self, db: Session):
        self.db = db
        self.books = BookRepository(db)
        self.categories = CategoryRepository(db)
        self.users = UserRepository(db)
        self.storage = StorageService()

    def list_books(
        self,
        *,
        current_user: User,
        search: str | None,
        author: str | None,
        category_id: int | None,
        status_filter: BookStatus | None,
        include_archived: bool,
        page: int,
        page_size: int,
    ) -> BookListResponse:
        include_all = current_user.role == UserRole.ADMIN and include_archived
        if current_user.role != UserRole.ADMIN:
            status_filter = None
            include_all = False

        items, total = self.books.list_books(
            search=search,
            author=author,
            category_id=category_id,
            status_filter=status_filter,
            include_archived=include_all,
            page=page,
            page_size=page_size,
        )
        total_pages = (total + page_size - 1) // page_size if total else 1
        return BookListResponse(
            items=items,
            meta=PaginationMeta(
                page=page,
                page_size=page_size,
                total=total,
                total_pages=total_pages,
            ),
        )

    def get_book(self, *, book_id: int, current_user: User) -> Book:
        book = self.books.get_by_id(book_id)
        if not book:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Book not found.")

        if current_user.role != UserRole.ADMIN and book.status != BookStatus.ACTIVE:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Book not found.")

        return book

    def create_book(
        self,
        *,
        current_user: User,
        title: str,
        author: str,
        description: str | None,
        category_id: int | None,
        file_bytes: bytes,
        original_filename: str,
        content_type: str,
    ) -> Book:
        if not original_filename.lower().endswith(".pdf"):
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Only PDF files are allowed.")

        allowed_content_types = {"application/pdf", "application/x-pdf", "binary/octet-stream"}
        if content_type not in allowed_content_types and not original_filename.lower().endswith(".pdf"):
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Invalid file type.")

        if not file_bytes:
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Uploaded file is empty.")

        max_size_bytes = settings.MAX_UPLOAD_SIZE_MB * 1024 * 1024
        if len(file_bytes) > max_size_bytes:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"File size exceeds {settings.MAX_UPLOAD_SIZE_MB} MB.",
            )

        if category_id is not None and not self.categories.get_by_id(category_id):
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Category not found.")

        upload_result = self.storage.upload_pdf(
            file_bytes=file_bytes,
            original_filename=original_filename,
            content_type="application/pdf",
        )

        book = Book(
            title=title.strip(),
            author=author.strip(),
            description=description.strip() if description else None,
            category_id=category_id,
            file_name=str(upload_result["file_name"]),
            storage_key=str(upload_result["storage_key"]),
            mime_type=str(upload_result["mime_type"]),
            file_size=int(upload_result["file_size"]),
            uploaded_by=current_user.id,
            status=BookStatus.ACTIVE,
            download_count=0,
        )
        self.books.add(book)
        self.db.commit()
        self.db.refresh(book)
        return self.get_book(book_id=book.id, current_user=current_user)

    def update_book(self, *, book_id: int, payload: BookUpdate) -> Book:
        book = self.books.get_by_id(book_id)
        if not book:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Book not found.")

        data = payload.model_dump(exclude_unset=True)

        if "category_id" in payload.model_fields_set:
            category_id = data.get("category_id")
            if category_id is not None and not self.categories.get_by_id(category_id):
                raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Category not found.")
            book.category_id = category_id

        for field in {"title", "author", "description"}:
            if field in data:
                setattr(book, field, data[field])

        self.db.commit()
        self.db.refresh(book)
        return self.books.get_by_id(book.id)

    def archive_book(self, *, book_id: int, admin_user: User) -> Book:
        book = self.books.get_by_id(book_id)
        if not book:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Book not found.")

        book.status = BookStatus.ARCHIVED
        book.archived_at = datetime.now(timezone.utc)
        book.archived_by = admin_user.id
        self.db.commit()
        self.db.refresh(book)
        return self.books.get_by_id(book.id)

    def restore_book(self, *, book_id: int, admin_user: User) -> Book:
        book = self.books.get_by_id(book_id)
        if not book:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Book not found.")

        book.status = BookStatus.ACTIVE
        book.archived_at = None
        book.archived_by = None
        self.db.commit()
        self.db.refresh(book)
        return self.books.get_by_id(book.id)

    def generate_download_link(
        self,
        *,
        book_id: int,
        current_user: User,
        client_ip: str | None,
        user_agent: str | None,
    ) -> BookDownloadResponse:
        book = self.get_book(book_id=book_id, current_user=current_user)

        download_record = Download(
            book_id=book.id,
            user_id=current_user.id,
            downloaded_at=datetime.now(timezone.utc),
            ip_address=client_ip,
            user_agent=user_agent,
        )
        self.db.add(download_record)

        book.download_count += 1
        self.db.commit()

        download_url = self.storage.generate_download_url(
            storage_key=book.storage_key,
            file_name=book.file_name,
        )
        return BookDownloadResponse(
            download_url=download_url,
            expires_in_seconds=settings.S3_PRESIGNED_URL_EXPIRE_SECONDS,
        )

    def dashboard_summary(self) -> DashboardSummary:
        return DashboardSummary(
            total_users=self.users.count_all(),
            total_books=self.books.count_all(),
            active_books=self.books.count_by_status(BookStatus.ACTIVE),
            archived_books=self.books.count_by_status(BookStatus.ARCHIVED),
            total_downloads=self.books.total_download_count(),
        )

File: backend/alembic.ini

[alembic]
script_location = migrations
prepend_sys_path = .
sqlalchemy.url = postgresql+psycopg://library_app_user:change_me@localhost:5432/library_db

[post_write_hooks]

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers = console
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s

File: backend/migrations/env.py

from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

from app.core.config import settings
from app.db.base import Base

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        compare_type=True,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata, compare_type=True)

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()

File: backend/migrations/versions/0001_initial_schema.py

"""initial schema

Revision ID: 0001_initial_schema
Revises:
Create Date: 2026-03-28 00:00:00
"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql


revision = "0001_initial_schema"
down_revision = None
branch_labels = None
depends_on = None


user_role_enum = sa.Enum("admin", "normal_user", name="user_role")
account_status_enum = sa.Enum("active", "inactive", "blocked", name="account_status")
book_status_enum = sa.Enum("active", "archived", "inactive", name="book_status")


def upgrade() -> None:
    bind = op.get_bind()
    user_role_enum.create(bind, checkfirst=True)
    account_status_enum.create(bind, checkfirst=True)
    book_status_enum.create(bind, checkfirst=True)

    op.create_table(
        "users",
        sa.Column("id", sa.BigInteger(), sa.Identity(always=False), primary_key=True),
        sa.Column("full_name", sa.String(length=150), nullable=False),
        sa.Column("username", sa.String(length=50), nullable=False),
        sa.Column("email", sa.String(length=255), nullable=False),
        sa.Column("hashed_password", sa.Text(), nullable=False),
        sa.Column("role", user_role_enum, nullable=False, server_default="normal_user"),
        sa.Column("account_status", account_status_enum, nullable=False, server_default="active"),
        sa.Column("is_active", sa.Boolean(), nullable=False, server_default=sa.text("true")),
        sa.Column("is_verified", sa.Boolean(), nullable=False, server_default=sa.text("false")),
        sa.Column(
            "must_change_password_on_first_login",
            sa.Boolean(),
            nullable=False,
            server_default=sa.text("false"),
        ),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.text("CURRENT_TIMESTAMP")),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.text("CURRENT_TIMESTAMP")),
        sa.Column("last_login", sa.DateTime(timezone=True), nullable=True),
    )
    op.create_index("ix_users_username", "users", ["username"], unique=True)
    op.create_index("ix_users_email", "users", ["email"], unique=True)
    op.create_index("ix_users_role", "users", ["role"], unique=False)

    op.create_table(
        "categories",
        sa.Column("id", sa.BigInteger(), sa.Identity(always=False), primary_key=True),
        sa.Column("name", sa.String(length=100), nullable=False),
        sa.Column("description", sa.Text(), nullable=True),
        sa.Column("is_active", sa.Boolean(), nullable=False, server_default=sa.text("true")),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.text("CURRENT_TIMESTAMP")),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.text("CURRENT_TIMESTAMP")),
    )
    op.create_index("ix_categories_name", "categories", ["name"], unique=True)

    op.create_table(
        "books",
        sa.Column("id", sa.BigInteger(), sa.Identity(always=False), primary_key=True),
        sa.Column("title", sa.String(length=255), nullable=False),
        sa.Column("author", sa.String(length=255), nullable=False),
        sa.Column("category_id", sa.BigInteger(), nullable=True),
        sa.Column("description", sa.Text(), nullable=True),
        sa.Column("file_name", sa.String(length=255), nullable=False),
        sa.Column("storage_key", sa.String(length=500), nullable=False),
        sa.Column("mime_type", sa.String(length=100), nullable=False),
        sa.Column("file_size", sa.BigInteger(), nullable=False),
        sa.Column("uploaded_by", sa.BigInteger(), nullable=False),
        sa.Column("status", book_status_enum, nullable=False, server_default="active"),
        sa.Column("archived_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("archived_by", sa.BigInteger(), nullable=True),
        sa.Column("download_count", sa.BigInteger(), nullable=False, server_default="0"),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.text("CURRENT_TIMESTAMP")),
        sa.Column("updated_at", sa.DateTime(timezone=True), nullable=False, server_default=sa.text("CURRENT_TIMESTAMP")),
        sa.ForeignKeyConstraint(["category_id"], ["categories.id"], ondelete="SET NULL"),
        sa.ForeignKeyConstraint(["uploaded_by"], ["users.id"], ondelete="RESTRICT"),
        sa.ForeignKeyConstraint(["archived_by"], ["users.id"], ondelete="SET NULL"),
    )
    op.create_index("ix_books_title", "books", ["title"], unique=False)
    op.create_index("ix_books_author", "books", ["author"], unique=False)
    op.create_index("ix_books_status", "books", ["status"], unique=False)
    op.create_index("ix_books_category_id", "books", ["category_id"], unique=False)
    op.create_index("ix_books_uploaded_by", "books", ["uploaded_by"], unique=False)
    op.create_index("ix_books_storage_key", "books", ["storage_key"], unique=True)

    op.create_table(
        "email_verification_tokens",
        sa.Column("id", sa.BigInteger(), sa.Identity(always=False), primary_key=True),
        sa.Column("user_id", sa.BigInteger(), nullable=False),
        sa.Column("token_hash", sa.Text(), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("used_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False),
        sa.ForeignKeyConstraint(["user_id"], ["users.id"], ondelete="CASCADE"),
    )
    op.create_index(
        "ix_email_verification_tokens_user_id",
        "email_verification_tokens",
        ["user_id"],
        unique=False,
    )
    op.create_index(
        "ix_email_verification_tokens_token_hash",
        "email_verification_tokens",
        ["token_hash"],
        unique=True,
    )

    op.create_table(
        "password_reset_tokens",
        sa.Column("id", sa.BigInteger(), sa.Identity(always=False), primary_key=True),
        sa.Column("user_id", sa.BigInteger(), nullable=False),
        sa.Column("token_hash", sa.Text(), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("used_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), nullable=False),
        sa.ForeignKeyConstraint(["user_id"], ["users.id"], ondelete="CASCADE"),
    )
    op.create_index("ix_password_reset_tokens_user_id", "password_reset_tokens", ["user_id"], unique=False)
    op.create_index("ix_password_reset_tokens_token_hash", "password_reset_tokens", ["token_hash"], unique=True)

    op.create_table(
        "downloads",
        sa.Column("id", sa.BigInteger(), sa.Identity(always=False), primary_key=True),
        sa.Column("book_id", sa.BigInteger(), nullable=False),
        sa.Column("user_id", sa.BigInteger(), nullable=True),
        sa.Column("downloaded_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("ip_address", postgresql.INET(), nullable=True),
        sa.Column("user_agent", sa.Text(), nullable=True),
        sa.ForeignKeyConstraint(["book_id"], ["books.id"], ondelete="CASCADE"),
        sa.ForeignKeyConstraint(["user_id"], ["users.id"], ondelete="SET NULL"),
    )
    op.create_index("ix_downloads_book_id", "downloads", ["book_id"], unique=False)
    op.create_index("ix_downloads_user_id", "downloads", ["user_id"], unique=False)


def downgrade() -> None:
    op.drop_index("ix_downloads_user_id", table_name="downloads")
    op.drop_index("ix_downloads_book_id", table_name="downloads")
    op.drop_table("downloads")

    op.drop_index("ix_password_reset_tokens_token_hash", table_name="password_reset_tokens")
    op.drop_index("ix_password_reset_tokens_user_id", table_name="password_reset_tokens")
    op.drop_table("password_reset_tokens")

    op.drop_index("ix_email_verification_tokens_token_hash", table_name="email_verification_tokens")
    op.drop_index("ix_email_verification_tokens_user_id", table_name="email_verification_tokens")
    op.drop_table("email_verification_tokens")

    op.drop_index("ix_books_storage_key", table_name="books")
    op.drop_index("ix_books_uploaded_by", table_name="books")
    op.drop_index("ix_books_category_id", table_name="books")
    op.drop_index("ix_books_status", table_name="books")
    op.drop_index("ix_books_author", table_name="books")
    op.drop_index("ix_books_title", table_name="books")
    op.drop_table("books")

    op.drop_index("ix_categories_name", table_name="categories")
    op.drop_table("categories")

    op.drop_index("ix_users_role", table_name="users")
    op.drop_index("ix_users_email", table_name="users")
    op.drop_index("ix_users_username", table_name="users")
    op.drop_table("users")

    bind = op.get_bind()
    book_status_enum.drop(bind, checkfirst=True)
    account_status_enum.drop(bind, checkfirst=True)
    user_role_enum.drop(bind, checkfirst=True)


---

4. Frontend folder structure and all frontend files with full code

File: frontend/package.json

{
  "name": "library-management-frontend",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.7.7",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.27.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.8",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "typescript": "^5.6.2",
    "vite": "^5.4.8"
  }
}

File: frontend/tsconfig.json

{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": false,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "baseUrl": "./src"
  },
  "include": ["src", "vite.config.ts"]
}

File: frontend/vite.config.ts

import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173
  }
});

File: frontend/index.html

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1.0"
    />
    <title>Library Management System</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>

File: frontend/src/vite-env.d.ts

/// <reference types="vite/client" />

File: frontend/src/main.tsx

import React from "react";
import ReactDOM from "react-dom/client";
import { BrowserRouter } from "react-router-dom";

import App from "./App";
import { AuthProvider } from "./context/AuthContext";
import "./styles/global.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <BrowserRouter>
      <AuthProvider>
        <App />
      </AuthProvider>
    </BrowserRouter>
  </React.StrictMode>
);

File: frontend/src/App.tsx

import { Navigate, Outlet, Route, Routes } from "react-router-dom";

import { AppLayout } from "./components/layout/AppLayout";
import { ProtectedRoute } from "./components/common/ProtectedRoute";
import { RoleRoute } from "./components/common/RoleRoute";
import { AddBookPage } from "./pages/AddBookPage";
import { AdminBooksPage } from "./pages/AdminBooksPage";
import { AdminDashboardPage } from "./pages/AdminDashboardPage";
import { ArchivedBooksPage } from "./pages/ArchivedBooksPage";
import { BookDetailPage } from "./pages/BookDetailPage";
import { BooksPage } from "./pages/BooksPage";
import { CategoryManagementPage } from "./pages/CategoryManagementPage";
import { ChangePasswordPage } from "./pages/ChangePasswordPage";
import { DashboardPage } from "./pages/DashboardPage";
import { ForgotPasswordPage } from "./pages/ForgotPasswordPage";
import { LoginPage } from "./pages/LoginPage";
import { NotFoundPage } from "./pages/NotFoundPage";
import { ProfilePage } from "./pages/ProfilePage";
import { RegisterPage } from "./pages/RegisterPage";
import { ResetPasswordPage } from "./pages/ResetPasswordPage";
import { UserManagementPage } from "./pages/UserManagementPage";
import { VerifyEmailPage } from "./pages/VerifyEmailPage";

function ProtectedAppShell() {
  return (
    <ProtectedRoute>
      <AppLayout>
        <Outlet />
      </AppLayout>
    </ProtectedRoute>
  );
}

function AdminShell() {
  return (
    <RoleRoute allowedRoles={["admin"]}>
      <Outlet />
    </RoleRoute>
  );
}

export default function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/register" element={<RegisterPage />} />
      <Route path="/forgot-password" element={<ForgotPasswordPage />} />
      <Route path="/reset-password" element={<ResetPasswordPage />} />
      <Route path="/verify-email" element={<VerifyEmailPage />} />

      <Route element={<ProtectedAppShell />}>
        <Route index element={<Navigate to="/dashboard" replace />} />
        <Route path="/dashboard" element={<DashboardPage />} />
        <Route path="/books" element={<BooksPage />} />
        <Route path="/books/:bookId" element={<BookDetailPage />} />
        <Route path="/books/add" element={<AddBookPage />} />
        <Route path="/profile" element={<ProfilePage />} />
        <Route path="/change-password" element={<ChangePasswordPage />} />

        <Route element={<AdminShell />}>
          <Route path="/admin/dashboard" element={<AdminDashboardPage />} />
          <Route path="/admin/books" element={<AdminBooksPage />} />
          <Route path="/admin/books/archived" element={<ArchivedBooksPage />} />
          <Route path="/admin/users" element={<UserManagementPage />} />
          <Route path="/admin/categories" element={<CategoryManagementPage />} />
        </Route>
      </Route>

      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  );
}

File: frontend/src/api/client.ts

import axios from "axios";

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || "http://localhost:8000/api/v1"
});

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("library_access_token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default apiClient;

File: frontend/src/api/auth.ts

import apiClient from "./client";
import type {
  ActionResponse,
  AuthResponse,
  ChangePasswordPayload,
  ForgotPasswordPayload,
  LoginPayload,
  RegisterPayload,
  RegisterResponse,
  ResetPasswordPayload,
  VerifyEmailPayload
} from "../types/auth";

export const authApi = {
  register: async (payload: RegisterPayload) => {
    const { data } = await apiClient.post<RegisterResponse>("/auth/register", payload);
    return data;
  },

  login: async (payload: LoginPayload) => {
    const { data } = await apiClient.post<AuthResponse>("/auth/login", payload);
    return data;
  },

  verifyEmail: async (payload: VerifyEmailPayload) => {
    const { data } = await apiClient.post<ActionResponse>("/auth/verify-email", payload);
    return data;
  },

  resendVerification: async (email: string) => {
    const { data } = await apiClient.post<ActionResponse>("/auth/verify-email/resend", { email });
    return data;
  },

  forgotPassword: async (payload: ForgotPasswordPayload) => {
    const { data } = await apiClient.post<ActionResponse>("/auth/forgot-password", payload);
    return data;
  },

  resetPassword: async (payload: ResetPasswordPayload) => {
    const { data } = await apiClient.post<ActionResponse>("/auth/reset-password", payload);
    return data;
  },

  changePassword: async (payload: ChangePasswordPayload) => {
    const { data } = await apiClient.post<ActionResponse>("/auth/change-password", payload);
    return data;
  }
};

File: frontend/src/api/users.ts

import apiClient from "./client";
import type { User, UserSelfUpdatePayload } from "../types/user";

export const usersApi = {
  getMe: async () => {
    const { data } = await apiClient.get<User>("/users/me");
    return data;
  },

  updateMe: async (payload: UserSelfUpdatePayload) => {
    const { data } = await apiClient.patch<User>("/users/me", payload);
    return data;
  }
};

File: frontend/src/api/books.ts

import apiClient from "./client";
import type { Book, BookDownloadResponse, BooksResponse, BookUpdatePayload } from "../types/book";

export interface BookListParams {
  search?: string;
  author?: string;
  category_id?: number;
  status?: "active" | "archived" | "inactive";
  include_archived?: boolean;
  page?: number;
  page_size?: number;
}

export const booksApi = {
  listBooks: async (params: BookListParams) => {
    const { data } = await apiClient.get<BooksResponse>("/books", { params });
    return data;
  },

  getBook: async (bookId: number) => {
    const { data } = await apiClient.get<Book>(`/books/${bookId}`);
    return data;
  },

  createBook: async (payload: {
    title: string;
    author: string;
    description?: string;
    category_id?: number;
    file: File;
  }) => {
    const formData = new FormData();
    formData.append("title", payload.title);
    formData.append("author", payload.author);
    if (payload.description) formData.append("description", payload.description);
    if (payload.category_id) formData.append("category_id", String(payload.category_id));
    formData.append("file", payload.file);

    const { data } = await apiClient.post<Book>("/books", formData, {
      headers: { "Content-Type": "multipart/form-data" }
    });
    return data;
  },

  updateBook: async (bookId: number, payload: BookUpdatePayload) => {
    const { data } = await apiClient.patch<Book>(`/books/${bookId}`, payload);
    return data;
  },

  archiveBook: async (bookId: number) => {
    const { data } = await apiClient.post<Book>(`/books/${bookId}/archive`);
    return data;
  },

  restoreBook: async (bookId: number) => {
    const { data } = await apiClient.post<Book>(`/books/${bookId}/restore`);
    return data;
  },

  getDownloadLink: async (bookId: number) => {
    const { data } = await apiClient.post<BookDownloadResponse>(`/books/${bookId}/download`);
    return data;
  }
};

File: frontend/src/api/categories.ts

import apiClient from "./client";
import type {
  Category,
  CategoryCreatePayload,
  CategoryListResponse,
  CategoryUpdatePayload
} from "../types/category";

export const categoriesApi = {
  listCategories: async (includeInactive = false) => {
    const { data } = await apiClient.get<CategoryListResponse>("/categories", {
      params: { include_inactive: includeInactive }
    });
    return data;
  },

  createCategory: async (payload: CategoryCreatePayload) => {
    const { data } = await apiClient.post<Category>("/categories", payload);
    return data;
  },

  updateCategory: async (categoryId: number, payload: CategoryUpdatePayload) => {
    const { data } = await apiClient.patch<Category>(`/categories/${categoryId}`, payload);
    return data;
  }
};

File: frontend/src/api/admin.ts

import apiClient from "./client";
import type { DashboardSummary, User, UsersResponse, AdminUserUpdatePayload } from "../types/user";

export const adminApi = {
  getDashboard: async () => {
    const { data } = await apiClient.get<DashboardSummary>("/admin/dashboard");
    return data;
  },

  listUsers: async (params: { search?: string; page?: number; page_size?: number }) => {
    const { data } = await apiClient.get<UsersResponse>("/admin/users", { params });
    return data;
  },

  updateUser: async (userId: number, payload: AdminUserUpdatePayload) => {
    const { data } = await apiClient.patch<User>(`/admin/users/${userId}`, payload);
    return data;
  }
};

File: frontend/src/types/common.ts

export interface PaginationMeta {
  page: number;
  page_size: number;
  total: number;
  total_pages: number;
}

export interface MessageResponse {
  message: string;
}

File: frontend/src/types/user.ts

import type { PaginationMeta } from "./common";

export type UserRole = "admin" | "normal_user";
export type AccountStatus = "active" | "inactive" | "blocked";

export interface User {
  id: number;
  full_name: string;
  username: string;
  email: string;
  role: UserRole;
  account_status: AccountStatus;
  is_active: boolean;
  is_verified: boolean;
  must_change_password_on_first_login: boolean;
  created_at: string;
  updated_at: string;
  last_login: string | null;
}

export interface UserBrief {
  id: number;
  full_name: string;
  username: string;
  role: UserRole;
}

export interface UserSelfUpdatePayload {
  full_name?: string;
  username?: string;
}

export interface AdminUserUpdatePayload {
  role?: UserRole;
  account_status?: AccountStatus;
  is_active?: boolean;
  is_verified?: boolean;
  must_change_password_on_first_login?: boolean;
}

export interface UsersResponse {
  items: User[];
  meta: PaginationMeta;
}

export interface DashboardSummary {
  total_users: number;
  total_books: number;
  active_books: number;
  archived_books: number;
  total_downloads: number;
}

File: frontend/src/types/auth.ts

import type { User } from "./user";

export interface RegisterPayload {
  full_name: string;
  username: string;
  email: string;
  password: string;
}

export interface LoginPayload {
  email: string;
  password: string;
}

export interface RegisterResponse {
  message: string;
  user: User;
  verification_url?: string | null;
}

export interface AuthResponse {
  access_token: string;
  token_type: string;
  user: User;
}

export interface ActionResponse {
  message: string;
  action_url?: string | null;
}

export interface ForgotPasswordPayload {
  email: string;
}

export interface ResetPasswordPayload {
  token: string;
  new_password: string;
}

export interface VerifyEmailPayload {
  token: string;
}

export interface ChangePasswordPayload {
  current_password: string;
  new_password: string;
}

File: frontend/src/types/category.ts

export interface Category {
  id: number;
  name: string;
  description: string | null;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}

export interface CategoryListResponse {
  items: Category[];
}

export interface CategoryCreatePayload {
  name: string;
  description?: string;
}

export interface CategoryUpdatePayload {
  name?: string;
  description?: string;
  is_active?: boolean;
}

File: frontend/src/types/book.ts

import type { Category } from "./category";
import type { PaginationMeta } from "./common";
import type { UserBrief } from "./user";

export type BookStatus = "active" | "archived" | "inactive";

export interface Book {
  id: number;
  title: string;
  author: string;
  description: string | null;
  file_name: string;
  storage_key: string;
  mime_type: string;
  file_size: number;
  status: BookStatus;
  download_count: number;
  created_at: string;
  updated_at: string;
  archived_at: string | null;
  category: Category | null;
  uploader: UserBrief;
  archiver: UserBrief | null;
}

export interface BooksResponse {
  items: Book[];
  meta: PaginationMeta;
}

export interface BookUpdatePayload {
  title?: string;
  author?: string;
  description?: string;
  category_id?: number | null;
}

export interface BookDownloadResponse {
  download_url: string;
  expires_in_seconds: number;
}

File: frontend/src/context/AuthContext.tsx

import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState,
  type ReactNode
} from "react";

import { authApi } from "../api/auth";
import { usersApi } from "../api/users";
import type { LoginPayload } from "../types/auth";
import type { User } from "../types/user";

interface AuthContextValue {
  user: User | null;
  token: string | null;
  isLoading: boolean;
  login: (payload: LoginPayload) => Promise<User>;
  logout: () => void;
  refreshUser: () => Promise<void>;
  setSession: (token: string, user: User) => void;
}

const ACCESS_TOKEN_KEY = "library_access_token";
const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(() => localStorage.getItem(ACCESS_TOKEN_KEY));
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const logout = useCallback(() => {
    localStorage.removeItem(ACCESS_TOKEN_KEY);
    setToken(null);
    setUser(null);
  }, []);

  const setSession = useCallback((nextToken: string, nextUser: User) => {
    localStorage.setItem(ACCESS_TOKEN_KEY, nextToken);
    setToken(nextToken);
    setUser(nextUser);
  }, []);

  const refreshUser = useCallback(async () => {
    if (!localStorage.getItem(ACCESS_TOKEN_KEY)) {
      setIsLoading(false);
      return;
    }

    try {
      const currentUser = await usersApi.getMe();
      setUser(currentUser);
    } catch {
      logout();
    } finally {
      setIsLoading(false);
    }
  }, [logout]);

  useEffect(() => {
    void refreshUser();
  }, [refreshUser]);

  const login = useCallback(
    async (payload: LoginPayload) => {
      const response = await authApi.login(payload);
      setSession(response.access_token, response.user);
      return response.user;
    },
    [setSession]
  );

  const value = useMemo<AuthContextValue>(
    () => ({
      user,
      token,
      isLoading,
      login,
      logout,
      refreshUser,
      setSession
    }),
    [user, token, isLoading, login, logout, refreshUser, setSession]
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuthContext() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuthContext must be used within AuthProvider.");
  }
  return context;
}

File: frontend/src/hooks/useAuth.ts

import { useAuthContext } from "../context/AuthContext";

export function useAuth() {
  return useAuthContext();
}

File: frontend/src/components/common/ProtectedRoute.tsx

import { Navigate } from "react-router-dom";
import type { ReactNode } from "react";

import { useAuth } from "../../hooks/useAuth";
import { LoadingState } from "./LoadingState";

export function ProtectedRoute({ children }: { children: ReactNode }) {
  const { user, isLoading } = useAuth();

  if (isLoading) {
    return <LoadingState text="Checking your session..." />;
  }

  if (!user) {
    return <Navigate to="/login" replace />;
  }

  return <>{children}</>;
}

File: frontend/src/components/common/RoleRoute.tsx

import { Navigate } from "react-router-dom";
import type { ReactNode } from "react";

import { useAuth } from "../../hooks/useAuth";
import type { UserRole } from "../../types/user";

export function RoleRoute({
  children,
  allowedRoles
}: {
  children: ReactNode;
  allowedRoles: UserRole[];
}) {
  const { user } = useAuth();

  if (!user) {
    return <Navigate to="/login" replace />;
  }

  if (!allowedRoles.includes(user.role)) {
    return <Navigate to="/dashboard" replace />;
  }

  return <>{children}</>;
}

File: frontend/src/components/common/LoadingState.tsx

export function LoadingState({ text = "Loading..." }: { text?: string }) {
  return (
    <div className="loading-state">
      <div className="spinner" />
      <p>{text}</p>
    </div>
  );
}

File: frontend/src/components/common/EmptyState.tsx

export function EmptyState({
  title,
  description
}: {
  title: string;
  description: string;
}) {
  return (
    <div className="empty-state">
      <h3>{title}</h3>
      <p>{description}</p>
    </div>
  );
}

File: frontend/src/components/common/PageHeader.tsx

import type { ReactNode } from "react";

export function PageHeader({
  title,
  description,
  action
}: {
  title: string;
  description?: string;
  action?: ReactNode;
}) {
  return (
    <div className="page-header">
      <div>
        <h1>{title}</h1>
        {description ? <p>{description}</p> : null}
      </div>
      {action ? <div>{action}</div> : null}
    </div>
  );
}

File: frontend/src/components/layout/AppLayout.tsx

import { NavLink, useNavigate } from "react-router-dom";
import type { ReactNode } from "react";

import { useAuth } from "../../hooks/useAuth";

export function AppLayout({ children }: { children: ReactNode }) {
  const { user, logout } = useAuth();
  const navigate = useNavigate();

  const handleLogout = () => {
    logout();
    navigate("/login");
  };

  return (
    <div className="app-shell">
      <aside className="sidebar">
        <div className="brand">
          <h2>Library LMS</h2>
          <p>{user?.role === "admin" ? "Admin Console" : "User Workspace"}</p>
        </div>

        <nav className="sidebar-nav">
          <NavLink to="/dashboard">Dashboard</NavLink>
          <NavLink to="/books">Books</NavLink>
          <NavLink to="/books/add">Add Book</NavLink>
          <NavLink to="/profile">Profile</NavLink>
          <NavLink to="/change-password">Change Password</NavLink>

          {user?.role === "admin" && (
            <>
              <div className="nav-section-title">Admin</div>
              <NavLink to="/admin/dashboard">Admin Dashboard</NavLink>
              <NavLink to="/admin/books">Manage Books</NavLink>
              <NavLink to="/admin/books/archived">Archived Books</NavLink>
              <NavLink to="/admin/users">Users</NavLink>
              <NavLink to="/admin/categories">Categories</NavLink>
            </>
          )}
        </nav>
      </aside>

      <main className="main-content">
        <header className="topbar">
          <div>
            <strong>{user?.full_name}</strong>
            <span className="muted"> @{user?.username}</span>
          </div>
          <button className="button secondary" onClick={handleLogout}>
            Logout
          </button>
        </header>

        <section className="page-content">{children}</section>
      </main>
    </div>
  );
}

File: frontend/src/components/books/BookFilters.tsx

import type { Category } from "../../types/category";

interface Props {
  search: string;
  author: string;
  categoryId: string;
  categories: Category[];
  onSearchChange: (value: string) => void;
  onAuthorChange: (value: string) => void;
  onCategoryChange: (value: string) => void;
}

export function BookFilters({
  search,
  author,
  categoryId,
  categories,
  onSearchChange,
  onAuthorChange,
  onCategoryChange
}: Props) {
  return (
    <div className="filters-grid">
      <input
        className="input"
        placeholder="Search by title or author"
        value={search}
        onChange={(event) => onSearchChange(event.target.value)}
      />
      <input
        className="input"
        placeholder="Author"
        value={author}
        onChange={(event) => onAuthorChange(event.target.value)}
      />
      <select
        className="input"
        value={categoryId}
        onChange={(event) => onCategoryChange(event.target.value)}
      >
        <option value="">All categories</option>
        {categories.map((category) => (
          <option key={category.id} value={String(category.id)}>
            {category.name}
          </option>
        ))}
      </select>
    </div>
  );
}

File: frontend/src/components/books/BookCard.tsx

import { Link } from "react-router-dom";

import type { Book } from "../../types/book";

export function BookCard({ book }: { book: Book }) {
  return (
    <article className="card book-card">
      <div className="book-card-top">
        <span className={`status-badge status-${book.status}`}>{book.status}</span>
        <span className="muted">{book.category?.name ?? "Uncategorized"}</span>
      </div>
      <h3>{book.title}</h3>
      <p className="muted">by {book.author}</p>
      <p className="book-description">{book.description || "No description available."}</p>
      <div className="book-meta">
        <span>Downloads: {book.download_count}</span>
        <span>Uploaded by: {book.uploader.username}</span>
      </div>
      <Link className="button primary small" to={`/books/${book.id}`}>
        View details
      </Link>
    </article>
  );
}

File: frontend/src/pages/LoginPage.tsx

import { useState } from "react";
import { Link, useNavigate } from "react-router-dom";

import { useAuth } from "../hooks/useAuth";

export function LoginPage() {
  const navigate = useNavigate();
  const { login } = useAuth();

  const [form, setForm] = useState({ email: "", password: "" });
  const [error, setError] = useState("");
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    setError("");
    setSubmitting(true);

    try {
      const user = await login(form);
      if (user.must_change_password_on_first_login) {
        navigate("/change-password");
      } else if (user.role === "admin") {
        navigate("/admin/dashboard");
      } else {
        navigate("/dashboard");
      }
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Login failed.");
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="auth-page">
      <form className="auth-card" onSubmit={handleSubmit}>
        <h1>Welcome back</h1>
        <p>Log in to access your digital library workspace.</p>

        {error ? <div className="alert error">{error}</div> : null}

        <label>
          Email
          <input
            className="input"
            type="email"
            value={form.email}
            onChange={(event) => setForm((prev) => ({ ...prev, email: event.target.value }))}
            required
          />
        </label>

        <label>
          Password
          <input
            className="input"
            type="password"
            value={form.password}
            onChange={(event) => setForm((prev) => ({ ...prev, password: event.target.value }))}
            required
          />
        </label>

        <button className="button primary full" disabled={submitting} type="submit">
          {submitting ? "Logging in..." : "Log in"}
        </button>

        <div className="auth-links">
          <Link to="/forgot-password">Forgot password?</Link>
          <Link to="/register">Create account</Link>
        </div>
      </form>
    </div>
  );
}

File: frontend/src/pages/RegisterPage.tsx

import { useState } from "react";
import { Link } from "react-router-dom";

import { authApi } from "../api/auth";

export function RegisterPage() {
  const [form, setForm] = useState({
    full_name: "",
    username: "",
    email: "",
    password: ""
  });
  const [error, setError] = useState("");
  const [success, setSuccess] = useState("");
  const [verificationUrl, setVerificationUrl] = useState<string | null>(null);
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    setError("");
    setSuccess("");
    setVerificationUrl(null);
    setSubmitting(true);

    try {
      const response = await authApi.register(form);
      setSuccess(response.message);
      setVerificationUrl(response.verification_url ?? null);
      setForm({ full_name: "", username: "", email: "", password: "" });
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Registration failed.");
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="auth-page">
      <form className="auth-card" onSubmit={handleSubmit}>
        <h1>Create account</h1>
        <p>Register as a normal user and start uploading or reading books.</p>

        {error ? <div className="alert error">{error}</div> : null}
        {success ? <div className="alert success">{success}</div> : null}
        {verificationUrl ? (
          <div className="alert info">
            Development verification link:{" "}
            <a href={verificationUrl} target="_blank" rel="noreferrer">
              Open verification link
            </a>
          </div>
        ) : null}

        <label>
          Full name
          <input
            className="input"
            value={form.full_name}
            onChange={(event) => setForm((prev) => ({ ...prev, full_name: event.target.value }))}
            required
          />
        </label>

        <label>
          Username
          <input
            className="input"
            value={form.username}
            onChange={(event) => setForm((prev) => ({ ...prev, username: event.target.value }))}
            required
          />
        </label>

        <label>
          Email
          <input
            className="input"
            type="email"
            value={form.email}
            onChange={(event) => setForm((prev) => ({ ...prev, email: event.target.value }))}
            required
          />
        </label>

        <label>
          Password
          <input
            className="input"
            type="password"
            value={form.password}
            onChange={(event) => setForm((prev) => ({ ...prev, password: event.target.value }))}
            required
          />
        </label>

        <button className="button primary full" type="submit" disabled={submitting}>
          {submitting ? "Creating account..." : "Create account"}
        </button>

        <div className="auth-links">
          <Link to="/login">Already have an account?</Link>
        </div>
      </form>
    </div>
  );
}

File: frontend/src/pages/ForgotPasswordPage.tsx

import { useState } from "react";

import { authApi } from "../api/auth";

export function ForgotPasswordPage() {
  const [email, setEmail] = useState("");
  const [message, setMessage] = useState("");
  const [actionUrl, setActionUrl] = useState<string | null>(null);
  const [error, setError] = useState("");

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    setError("");
    setMessage("");
    setActionUrl(null);

    try {
      const response = await authApi.forgotPassword({ email });
      setMessage(response.message);
      setActionUrl(response.action_url ?? null);
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Could not process your request.");
    }
  };

  return (
    <div className="auth-page">
      <form className="auth-card" onSubmit={handleSubmit}>
        <h1>Forgot password</h1>
        <p>Enter your email address to generate a password reset link.</p>

        {error ? <div className="alert error">{error}</div> : null}
        {message ? <div className="alert success">{message}</div> : null}
        {actionUrl ? (
          <div className="alert info">
            Development reset link:{" "}
            <a href={actionUrl} target="_blank" rel="noreferrer">
              Open reset link
            </a>
          </div>
        ) : null}

        <label>
          Email
          <input
            className="input"
            type="email"
            value={email}
            onChange={(event) => setEmail(event.target.value)}
            required
          />
        </label>

        <button className="button primary full" type="submit">
          Send reset link
        </button>
      </form>
    </div>
  );
}

File: frontend/src/pages/ResetPasswordPage.tsx

import { useMemo, useState } from "react";
import { useLocation } from "react-router-dom";

import { authApi } from "../api/auth";

export function ResetPasswordPage() {
  const location = useLocation();
  const initialToken = useMemo(() => new URLSearchParams(location.search).get("token") || "", [location.search]);

  const [token, setToken] = useState(initialToken);
  const [newPassword, setNewPassword] = useState("");
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    setError("");
    setMessage("");

    try {
      const response = await authApi.resetPassword({ token, new_password: newPassword });
      setMessage(response.message);
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Reset password failed.");
    }
  };

  return (
    <div className="auth-page">
      <form className="auth-card" onSubmit={handleSubmit}>
        <h1>Reset password</h1>
        <p>Enter the token and your new password.</p>

        {error ? <div className="alert error">{error}</div> : null}
        {message ? <div className="alert success">{message}</div> : null}

        <label>
          Reset token
          <input className="input" value={token} onChange={(event) => setToken(event.target.value)} required />
        </label>

        <label>
          New password
          <input
            className="input"
            type="password"
            value={newPassword}
            onChange={(event) => setNewPassword(event.target.value)}
            required
          />
        </label>

        <button className="button primary full" type="submit">
          Reset password
        </button>
      </form>
    </div>
  );
}

File: frontend/src/pages/VerifyEmailPage.tsx

import { useMemo, useState } from "react";
import { useLocation } from "react-router-dom";

import { authApi } from "../api/auth";

export function VerifyEmailPage() {
  const location = useLocation();
  const initialToken = useMemo(() => new URLSearchParams(location.search).get("token") || "", [location.search]);

  const [token, setToken] = useState(initialToken);
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");

  const handleVerify = async (event: React.FormEvent) => {
    event.preventDefault();
    setError("");
    setMessage("");

    try {
      const response = await authApi.verifyEmail({ token });
      setMessage(response.message);
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Verification failed.");
    }
  };

  return (
    <div className="auth-page">
      <form className="auth-card" onSubmit={handleVerify}>
        <h1>Verify email</h1>
        <p>Use the verification token from your email or development link.</p>

        {error ? <div className="alert error">{error}</div> : null}
        {message ? <div className="alert success">{message}</div> : null}

        <label>
          Verification token
          <input className="input" value={token} onChange={(event) => setToken(event.target.value)} required />
        </label>

        <button className="button primary full" type="submit">
          Verify email
        </button>
      </form>
    </div>
  );
}

File: frontend/src/pages/DashboardPage.tsx

import { Link } from "react-router-dom";

import { useAuth } from "../hooks/useAuth";
import { PageHeader } from "../components/common/PageHeader";

export function DashboardPage() {
  const { user } = useAuth();

  return (
    <div className="stack">
      <PageHeader
        title="Dashboard"
        description="Your entry point to browse, upload, and manage library resources."
      />

      {user?.must_change_password_on_first_login ? (
        <div className="alert warning">
          You are required to change your password before continuing normal use.
          <div className="spacer-top">
            <Link className="button primary small" to="/change-password">
              Change password now
            </Link>
          </div>
        </div>
      ) : null}

      <div className="dashboard-grid">
        <Link className="card action-card" to="/books">
          <h3>Browse Books</h3>
          <p>Search, filter, and open the current active library collection.</p>
        </Link>

        <Link className="card action-card" to="/books/add">
          <h3>Upload a PDF</h3>
          <p>Add a new PDF book to the system with category and metadata.</p>
        </Link>

        <Link className="card action-card" to="/profile">
          <h3>Update Profile</h3>
          <p>Manage your display information and username.</p>
        </Link>

        <Link className="card action-card" to="/change-password">
          <h3>Change Password</h3>
          <p>Rotate your password and keep your account secure.</p>
        </Link>
      </div>

      {user?.role === "admin" ? (
        <div className="alert info">
          You are logged in as an admin. Use the admin menu from the sidebar to manage users,
          books, archived content, and categories.
        </div>
      ) : null}
    </div>
  );
}

File: frontend/src/pages/BooksPage.tsx

import { useEffect, useState } from "react";

import { booksApi } from "../api/books";
import { categoriesApi } from "../api/categories";
import { BookCard } from "../components/books/BookCard";
import { BookFilters } from "../components/books/BookFilters";
import { EmptyState } from "../components/common/EmptyState";
import { LoadingState } from "../components/common/LoadingState";
import { PageHeader } from "../components/common/PageHeader";
import type { Book } from "../types/book";
import type { Category } from "../types/category";

export function BooksPage() {
  const [books, setBooks] = useState<Book[]>([]);
  const [categories, setCategories] = useState<Category[]>([]);
  const [search, setSearch] = useState("");
  const [author, setAuthor] = useState("");
  const [categoryId, setCategoryId] = useState("");
  const [loading, setLoading] = useState(true);

  const loadData = async () => {
    setLoading(true);
    try {
      const [bookResponse, categoryResponse] = await Promise.all([
        booksApi.listBooks({
          search: search || undefined,
          author: author || undefined,
          category_id: categoryId ? Number(categoryId) : undefined,
          page: 1,
          page_size: 24
        }),
        categoriesApi.listCategories(false)
      ]);
      setBooks(bookResponse.items);
      setCategories(categoryResponse.items);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    void loadData();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const handleFilterSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    await loadData();
  };

  return (
    <div className="stack">
      <PageHeader title="Books" description="Browse active books, search by title/author, and filter by category." />

      <form className="card" onSubmit={handleFilterSubmit}>
        <BookFilters
          search={search}
          author={author}
          categoryId={categoryId}
          categories={categories}
          onSearchChange={setSearch}
          onAuthorChange={setAuthor}
          onCategoryChange={setCategoryId}
        />
        <div className="actions-row">
          <button className="button primary" type="submit">
            Apply filters
          </button>
        </div>
      </form>

      {loading ? <LoadingState text="Loading books..." /> : null}

      {!loading && books.length === 0 ? (
        <EmptyState title="No books found" description="Try changing your filters or upload a new book." />
      ) : null}

      <div className="book-grid">
        {books.map((book) => (
          <BookCard key={book.id} book={book} />
        ))}
      </div>
    </div>
  );
}

File: frontend/src/pages/BookDetailPage.tsx

import { useEffect, useState } from "react";
import { useParams } from "react-router-dom";

import { booksApi } from "../api/books";
import { EmptyState } from "../components/common/EmptyState";
import { LoadingState } from "../components/common/LoadingState";
import { PageHeader } from "../components/common/PageHeader";
import type { Book } from "../types/book";

export function BookDetailPage() {
  const { bookId } = useParams();
  const [book, setBook] = useState<Book | null>(null);
  const [loading, setLoading] = useState(true);
  const [downloading, setDownloading] = useState(false);

  useEffect(() => {
    const load = async () => {
      if (!bookId) return;
      setLoading(true);
      try {
        const data = await booksApi.getBook(Number(bookId));
        setBook(data);
      } finally {
        setLoading(false);
      }
    };
    void load();
  }, [bookId]);

  const handleDownload = async () => {
    if (!book) return;
    setDownloading(true);
    try {
      const response = await booksApi.getDownloadLink(book.id);
      window.open(response.download_url, "_blank", "noopener,noreferrer");
    } finally {
      setDownloading(false);
    }
  };

  if (loading) {
    return <LoadingState text="Loading book details..." />;
  }

  if (!book) {
    return <EmptyState title="Book not found" description="The requested book does not exist or is not visible." />;
  }

  return (
    <div className="stack">
      <PageHeader title={book.title} description={`by ${book.author}`} />

      <div className="card detail-card">
        <div className="detail-grid">
          <div>
            <strong>Status</strong>
            <p>{book.status}</p>
          </div>
          <div>
            <strong>Category</strong>
            <p>{book.category?.name ?? "Uncategorized"}</p>
          </div>
          <div>
            <strong>Uploaded by</strong>
            <p>{book.uploader.full_name}</p>
          </div>
          <div>
            <strong>Downloads</strong>
            <p>{book.download_count}</p>
          </div>
          <div>
            <strong>File name</strong>
            <p>{book.file_name}</p>
          </div>
          <div>
            <strong>File size</strong>
            <p>{(book.file_size / 1024 / 1024).toFixed(2)} MB</p>
          </div>
        </div>

        <div className="detail-section">
          <h3>Description</h3>
          <p>{book.description || "No description available."}</p>
        </div>

        <div className="actions-row">
          <button className="button primary" onClick={handleDownload} disabled={downloading}>
            {downloading ? "Preparing download..." : "Download PDF"}
          </button>
        </div>
      </div>
    </div>
  );
}

File: frontend/src/pages/AddBookPage.tsx

import { useEffect, useState } from "react";
import { useNavigate } from "react-router-dom";

import { booksApi } from "../api/books";
import { categoriesApi } from "../api/categories";
import { PageHeader } from "../components/common/PageHeader";
import type { Category } from "../types/category";

export function AddBookPage() {
  const navigate = useNavigate();
  const [categories, setCategories] = useState<Category[]>([]);
  const [form, setForm] = useState({
    title: "",
    author: "",
    description: "",
    category_id: ""
  });
  const [file, setFile] = useState<File | null>(null);
  const [error, setError] = useState("");
  const [submitting, setSubmitting] = useState(false);

  useEffect(() => {
    const loadCategories = async () => {
      const response = await categoriesApi.listCategories(false);
      setCategories(response.items);
    };
    void loadCategories();
  }, []);

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();

    if (!file) {
      setError("Please select a PDF file.");
      return;
    }

    setError("");
    setSubmitting(true);

    try {
      const response = await booksApi.createBook({
        title: form.title,
        author: form.author,
        description: form.description || undefined,
        category_id: form.category_id ? Number(form.category_id) : undefined,
        file
      });
      navigate(`/books/${response.id}`);
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Could not upload the book.");
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="stack">
      <PageHeader title="Add Book" description="Upload a PDF and attach its metadata." />

      <form className="card form-grid" onSubmit={handleSubmit}>
        {error ? <div className="alert error">{error}</div> : null}

        <label>
          Title
          <input
            className="input"
            value={form.title}
            onChange={(event) => setForm((prev) => ({ ...prev, title: event.target.value }))}
            required
          />
        </label>

        <label>
          Author
          <input
            className="input"
            value={form.author}
            onChange={(event) => setForm((prev) => ({ ...prev, author: event.target.value }))}
            required
          />
        </label>

        <label>
          Category
          <select
            className="input"
            value={form.category_id}
            onChange={(event) => setForm((prev) => ({ ...prev, category_id: event.target.value }))}
          >
            <option value="">Uncategorized</option>
            {categories.map((category) => (
              <option key={category.id} value={String(category.id)}>
                {category.name}
              </option>
            ))}
          </select>
        </label>

        <label>
          PDF file
          <input
            className="input"
            type="file"
            accept="application/pdf,.pdf"
            onChange={(event) => setFile(event.target.files?.[0] ?? null)}
            required
          />
        </label>

        <label className="full-width">
          Description
          <textarea
            className="input textarea"
            value={form.description}
            onChange={(event) => setForm((prev) => ({ ...prev, description: event.target.value }))}
          />
        </label>

        <div className="actions-row full-width">
          <button className="button primary" type="submit" disabled={submitting}>
            {submitting ? "Uploading..." : "Upload book"}
          </button>
        </div>
      </form>
    </div>
  );
}

File: frontend/src/pages/ProfilePage.tsx

import { useState } from "react";

import { usersApi } from "../api/users";
import { PageHeader } from "../components/common/PageHeader";
import { useAuth } from "../hooks/useAuth";

export function ProfilePage() {
  const { user, refreshUser } = useAuth();
  const [fullName, setFullName] = useState(user?.full_name ?? "");
  const [username, setUsername] = useState(user?.username ?? "");
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    setMessage("");
    setError("");

    try {
      await usersApi.updateMe({ full_name: fullName, username });
      await refreshUser();
      setMessage("Profile updated successfully.");
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Could not update profile.");
    }
  };

  return (
    <div className="stack">
      <PageHeader title="Profile" description="Update your account profile information." />

      <form className="card form-grid" onSubmit={handleSubmit}>
        {message ? <div className="alert success">{message}</div> : null}
        {error ? <div className="alert error">{error}</div> : null}

        <label>
          Full name
          <input className="input" value={fullName} onChange={(event) => setFullName(event.target.value)} required />
        </label>

        <label>
          Username
          <input className="input" value={username} onChange={(event) => setUsername(event.target.value)} required />
        </label>

        <label>
          Email
          <input className="input" value={user?.email ?? ""} disabled />
        </label>

        <label>
          Role
          <input className="input" value={user?.role ?? ""} disabled />
        </label>

        <div className="actions-row full-width">
          <button className="button primary" type="submit">
            Save changes
          </button>
        </div>
      </form>
    </div>
  );
}

File: frontend/src/pages/ChangePasswordPage.tsx

import { useState } from "react";

import { authApi } from "../api/auth";
import { PageHeader } from "../components/common/PageHeader";

export function ChangePasswordPage() {
  const [form, setForm] = useState({ current_password: "", new_password: "" });
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    setMessage("");
    setError("");

    try {
      const response = await authApi.changePassword(form);
      setMessage(response.message);
      setForm({ current_password: "", new_password: "" });
    } catch (err: any) {
      setError(err?.response?.data?.detail || "Could not change password.");
    }
  };

  return (
    <div className="stack">
      <PageHeader title="Change Password" description="Rotate your password and secure your account." />

      <form className="card form-grid" onSubmit={handleSubmit}>
        {message ? <div className="alert success">{message}</div> : null}
        {error ? <div className="alert error">{error}</div> : null}

        <label>
          Current password
          <input
            className="input"
            type="password"
            value={form.current_password}
            onChange={(event) =>
              setForm((prev) => ({ ...prev, current_password: event.target.value }))
            }
            required
          />
        </label>

        <label>
          New password
          <input
            className="input"
            type="password"
            value={form.new_password}
            onChange={(event) =>
              setForm((prev) => ({ ...prev, new_password: event.target.value }))
            }
            required
          />
        </label>

        <div className="actions-row full-width">
          <button className="button primary" type="submit">
            Update password
          </button>
        </div>
      </form>
    </div>
  );
}

File: frontend/src/pages/AdminDashboardPage.tsx

import { useEffect, useState } from "react";

import { adminApi } from "../api/admin";
import { LoadingState } from "../components/common/LoadingState";
import { PageHeader } from "../components/common/PageHeader";
import type { DashboardSummary } from "../types/user";

export function AdminDashboardPage() {
  const [summary, setSummary] = useState<DashboardSummary | null>(null);

  useEffect(() => {
    const load = async () => {
      const data = await adminApi.getDashboard();
      setSummary(data);
    };
    void load();
  }, []);

  if (!summary) {
    return <LoadingState text="Loading admin dashboard..." />;
  }

  return (
    <div className="stack">
      <PageHeader title="Admin Dashboard" description="High-level metrics for your library system." />

      <div className="dashboard-grid">
        <div className="card stat-card">
          <h3>Total users</h3>
          <p>{summary.total_users}</p>
        </div>
        <div className="card stat-card">
          <h3>Total books</h3>
          <p>{summary.total_books}</p>
        </div>
        <div className="card stat-card">
          <h3>Active books</h3>
          <p>{summary.active_books}</p>
        </div>
        <div className="card stat-card">
          <h3>Archived books</h3>
          <p>{summary.archived_books}</p>
        </div>
        <div className="card stat-card">
          <h3>Total downloads</h3>
          <p>{summary.total_downloads}</p>
        </div>
      </div>
    </div>
  );
}

File: frontend/src/pages/AdminBooksPage.tsx

import { useEffect, useState } from "react";

import { booksApi } from "../api/books";
import { categoriesApi } from "../api/categories";
import { BookFilters } from "../components/books/BookFilters";
import { EmptyState } from "../components/common/EmptyState";
import { LoadingState } from "../components/common/LoadingState";
import { PageHeader } from "../components/common/PageHeader";
import type { Book } from "../types/book";
import type { Category } from "../types/category";

export function AdminBooksPage() {
  const [books, setBooks] = useState<Book[]>([]);
  const [categories, setCategories] = useState<Category[]>([]);
  const [search, setSearch] = useState("");
  const [author, setAuthor] = useState("");
  const [categoryId, setCategoryId] = useState("");
  const [loading, setLoading] = useState(true);

  const load = async () => {
    setLoading(true);
    try {
      const [bookResponse, categoryResponse] = await Promise.all([
        booksApi.listBooks({
          search: search || undefined,
          author: author || undefined,
          category_id: categoryId ? Number(categoryId) : undefined,
          include_archived: true,
          page: 1,
          page_size: 50
        }),
        categoriesApi.listCategories(true)
      ]);
      setBooks(bookResponse.items);
      setCategories(categoryResponse.items);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    void load();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const handleArchive = async (bookId: number) => {
    await booksApi.archiveBook(bookId);
    await load();
  };

  const handleRestore = async (bookId: number) => {
    await booksApi.restoreBook(bookId);
    await load();
  };

  return (
    <div className="stack">
      <PageHeader title="Manage Books" description="Review all books and archive or restore them as needed." />

      <form
        className="card"
        onSubmit={(event) => {
          event.preventDefault();
          void load();
        }}
      >
        <BookFilters
          search={search}
          author={author}
          categoryId={categoryId}
          categories={categories}
          onSearchChange={setSearch}
          onAuthorChange={setAuthor}
          onCategoryChange={setCategoryId}
        />
        <div className="actions-row">
          <button className="button primary" type="submit">
            Refresh
          </button>
        </div>
      </form>

      {loading ? <LoadingState text="Loading admin books..." /> : null}

      {!loading && books.length === 0 ? (
        <EmptyState title="No books found" description="No books match the current filters." />
      ) : null}

      <div className="card table-card">
        <table className="table">
          <thead>
            <tr>
              <th>Title</th>
              <th>Author</th>
              <th>Status</th>
              <th>Category</th>
              <th>Downloads</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {books.map((book) => (
              <tr key={book.id}>
                <td>{book.title}</td>
                <td>{book.author}</td>
                <td>{book.status}</td>
                <td>{book.category?.name ?? "Uncategorized"}</td>
                <td>{book.download_count}</td>
                <td>
                  <div className="actions-inline">
                    {book.status === "archived" ? (
                      <button className="button secondary small" onClick={() => void handleRestore(book.id)}>
                        Restore
                      </button>
                    ) : (
                      <button className="button danger small" onClick={() => void handleArchive(book.id)}>
                        Archive
                      </button>
                    )}
                  </div>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

File: frontend/src/pages/ArchivedBooksPage.tsx

import { useEffect, useState } from "react";

import { booksApi } from "../api/books";
import { EmptyState } from "../components/common/EmptyState";
import { LoadingState } from "../components/common/LoadingState";
import { PageHeader } from "../components/common/PageHeader";
import type { Book } from "../types/book";

export function ArchivedBooksPage() {
  const [books, setBooks] = useState<Book[]>([]);
  const [loading, setLoading] = useState(true);

  const load = async () => {
    setLoading(true);
    try {
      const response = await booksApi.listBooks({
        include_archived: true,
        status: "archived",
        page: 1,
        page_size: 50
      });
      setBooks(response.items);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    void load();
  }, []);

  const handleRestore = async (bookId: number) => {
    await booksApi.restoreBook(bookId);
    await load();
  };

  return (
    <div className="stack">
      <PageHeader title="Archived Books" description="Restore archived books back to the active catalog." />

      {loading ? <LoadingState text="Loading archived books..." /> : null}

      {!loading && books.length === 0 ? (
        <EmptyState title="No archived books" description="There are no archived books right now." />
      ) : null}

      <div className="card table-card">
        <table className="table">
          <thead>
            <tr>
              <th>Title</th>
              <th>Author</th>
              <th>Archived at</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            {books.map((book) => (
              <tr key={book.id}>
                <td>{book.title}</td>
                <td>{book.author}</td>
                <td>{book.archived_at ? new Date(book.archived_at).toLocaleString() : "-"}</td>
                <td>
                  <button className="button secondary small" onClick={() => void handleRestore(book.id)}>
                    Restore
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

File: frontend/src/pages/UserManagementPage.tsx

import { useEffect, useState } from "react";

import { adminApi } from "../api/admin";
import { LoadingState } from "../components/common/LoadingState";
import { PageHeader } from "../components/common/PageHeader";
import type { AccountStatus, User, UserRole } from "../types/user";

export function UserManagementPage() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);

  const load = async () => {
    setLoading(true);
    try {
      const response = await adminApi.listUsers({ page: 1, page_size: 100 });
      setUsers(response.items);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    void load();
  }, []);

  const updateUser = async (
    userId: number,
    payload: Partial<{
      role: UserRole;
      account_status: AccountStatus;
      is_active: boolean;
      is_verified: boolean;
      must_change_password_on_first_login: boolean;
    }>
  ) => {
    const updated = await adminApi.updateUser(userId, payload);
    setUsers((prev) => prev.map((user) => (user.id === userId ? updated : user)));
  };

  if (loading) {
    return <LoadingState text="Loading users..." />;
  }

  return (
    <div className="stack">
      <PageHeader title="User Management" description="Review users, roles, and account status." />

      <div className="card table-card">
        <table className="table">
          <thead>
            <tr>
              <th>User</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Verified</th>
              <th>Force Password Change</th>
            </tr>
          </thead>
          <tbody>
            {users.map((user) => (
              <tr key={user.id}>
                <td>
                  <strong>{user.full_name}</strong>
                  <div className="muted">@{user.username}</div>
                </td>
                <td>{user.email}</td>
                <td>
                  <select
                    className="input small-input"
                    value={user.role}
                    onChange={(event) =>
                      void updateUser(user.id, { role: event.target.value as UserRole })
                    }
                  >
                    <option value="normal_user">normal_user</option>
                    <option value="admin">admin</option>
                  </select>
                </td>
                <td>
                  <select
                    className="input small-input"
                    value={user.account_status}
                    onChange={(event) =>
                      void updateUser(user.id, {
                        account_status: event.target.value as AccountStatus
                      })
                    }
                  >
                    <option value="active">active</option>
                    <option value="inactive">inactive</option>
                    <option value="blocked">blocked</option>
                  </select>
                </td>
                <td>
                  <input
                    type="checkbox"
                    checked={user.is_verified}
                    onChange={(event) =>
                      void updateUser(user.id, { is_verified: event.target.checked })
                    }
                  />
                </td>
                <td>
                  <input
                    type="checkbox"
                    checked={user.must_change_password_on_first_login}
                    onChange={(event) =>
                      void updateUser(user.id, {
                        must_change_password_on_first_login: event.target.checked
                      })
                    }
                  />
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

File: frontend/src/pages/CategoryManagementPage.tsx

import { useEffect, useState } from "react";

import { categoriesApi } from "../api/categories";
import { PageHeader } from "../components/common/PageHeader";
import type { Category } from "../types/category";

export function CategoryManagementPage() {
  const [categories, setCategories] = useState<Category[]>([]);
  const [newCategory, setNewCategory] = useState({ name: "", description: "" });
  const [message, setMessage] = useState("");

  const load = async () => {
    const response = await categoriesApi.listCategories(true);
    setCategories(response.items);
  };

  useEffect(() => {
    void load();
  }, []);

  const handleCreate = async (event: React.FormEvent) => {
    event.preventDefault();
    await categoriesApi.createCategory(newCategory);
    setNewCategory({ name: "", description: "" });
    setMessage("Category created successfully.");
    await load();
  };

  const handleToggle = async (category: Category) => {
    await categoriesApi.updateCategory(category.id, { is_active: !category.is_active });
    await load();
  };

  return (
    <div className="stack">
      <PageHeader title="Category Management" description="Create categories and toggle their active state." />

      {message ? <div className="alert success">{message}</div> : null}

      <form className="card form-grid" onSubmit={handleCreate}>
        <label>
          Name
          <input
            className="input"
            value={newCategory.name}
            onChange={(event) =>
              setNewCategory((prev) => ({ ...prev, name: event.target.value }))
            }
            required
          />
        </label>

        <label>
          Description
          <input
            className="input"
            value={newCategory.description}
            onChange={(event) =>
              setNewCategory((prev) => ({ ...prev, description: event.target.value }))
            }
          />
        </label>

        <div className="actions-row full-width">
          <button className="button primary" type="submit">
            Add category
          </button>
        </div>
      </form>

      <div className="card table-card">
        <table className="table">
          <thead>
            <tr>
              <th>Name</th>
              <th>Description</th>
              <th>Active</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            {categories.map((category) => (
              <tr key={category.id}>
                <td>{category.name}</td>
                <td>{category.description || "-"}</td>
                <td>{category.is_active ? "Yes" : "No"}</td>
                <td>
                  <button className="button secondary small" onClick={() => void handleToggle(category)}>
                    {category.is_active ? "Deactivate" : "Activate"}
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}

File: frontend/src/pages/NotFoundPage.tsx

import { Link } from "react-router-dom";

export function NotFoundPage() {
  return (
    <div className="auth-page">
      <div className="auth-card">
        <h1>Page not found</h1>
        <p>The page you are looking for does not exist.</p>
        <Link to="/dashboard" className="button primary full">
          Go to dashboard
        </Link>
      </div>
    </div>
  );
}

File: frontend/src/styles/global.css

:root {
  --bg: #0b1020;
  --bg-soft: #121931;
  --card: #ffffff;
  --card-muted: #f7f8fc;
  --text: #0f172a;
  --muted: #64748b;
  --primary: #2563eb;
  --primary-dark: #1d4ed8;
  --danger: #dc2626;
  --warning: #d97706;
  --success: #16a34a;
  --border: #e2e8f0;
  --shadow: 0 10px 30px rgba(15, 23, 42, 0.08);
  font-family: Inter, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

* {
  box-sizing: border-box;
}

html,
body,
#root {
  margin: 0;
  min-height: 100%;
  background: #f1f5f9;
  color: var(--text);
}

body {
  line-height: 1.5;
}

a {
  color: inherit;
  text-decoration: none;
}

.auth-page {
  min-height: 100vh;
  display: grid;
  place-items: center;
  padding: 2rem;
  background:
    radial-gradient(circle at top left, rgba(37, 99, 235, 0.18), transparent 25%),
    linear-gradient(135deg, var(--bg), #101827 60%, #0b1224);
}

.auth-card {
  width: 100%;
  max-width: 440px;
  background: rgba(255, 255, 255, 0.96);
  padding: 2rem;
  border-radius: 20px;
  box-shadow: var(--shadow);
}

.auth-card h1 {
  margin-top: 0;
  margin-bottom: 0.5rem;
}

.auth-card p {
  color: var(--muted);
  margin-bottom: 1.5rem;
}

.auth-links {
  margin-top: 1rem;
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  font-size: 0.95rem;
  color: var(--primary);
}

.input,
.textarea,
select.input {
  width: 100%;
  padding: 0.85rem 1rem;
  border: 1px solid var(--border);
  border-radius: 12px;
  margin-top: 0.35rem;
  font-size: 0.95rem;
  background: white;
}

.textarea {
  min-height: 120px;
  resize: vertical;
}

label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.9rem;
}

.button {
  border: none;
  border-radius: 12px;
  padding: 0.85rem 1.2rem;
  cursor: pointer;
  font-weight: 600;
  transition: 0.2s ease;
}

.button:hover {
  transform: translateY(-1px);
}

.button.primary {
  background: var(--primary);
  color: white;
}

.button.primary:hover {
  background: var(--primary-dark);
}

.button.secondary {
  background: #e2e8f0;
  color: var(--text);
}

.button.danger {
  background: var(--danger);
  color: white;
}

.button.full {
  width: 100%;
}

.button.small {
  padding: 0.55rem 0.9rem;
  font-size: 0.85rem;
}

.app-shell {
  min-height: 100vh;
  display: grid;
  grid-template-columns: 280px 1fr;
}

.sidebar {
  background: var(--bg);
  color: white;
  padding: 1.5rem;
}

.brand h2 {
  margin: 0 0 0.35rem 0;
}

.brand p {
  margin: 0;
  color: rgba(255, 255, 255, 0.72);
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
  margin-top: 1.75rem;
}

.sidebar-nav a {
  color: rgba(255, 255, 255, 0.82);
  padding: 0.75rem 0.9rem;
  border-radius: 10px;
}

.sidebar-nav a.active,
.sidebar-nav a:hover {
  background: rgba(255, 255, 255, 0.08);
  color: white;
}

.nav-section-title {
  margin-top: 1rem;
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: rgba(255, 255, 255, 0.5);
}

.main-content {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.topbar {
  background: white;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid var(--border);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.page-content {
  padding: 1.5rem;
}

.page-header {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: end;
}

.page-header h1 {
  margin: 0 0 0.35rem 0;
}

.page-header p {
  margin: 0;
  color: var(--muted);
}

.stack {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.card {
  background: var(--card);
  border-radius: 18px;
  box-shadow: var(--shadow);
  padding: 1.25rem;
}

.book-grid,
.dashboard-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}

.stat-card p {
  font-size: 2rem;
  margin: 0.5rem 0 0 0;
  font-weight: 700;
}

.action-card h3,
.book-card h3 {
  margin-top: 0;
}

.book-card-top,
.book-meta,
.actions-row,
.actions-inline {
  display: flex;
  gap: 0.75rem;
  align-items: center;
  flex-wrap: wrap;
}

.actions-row {
  margin-top: 1rem;
}

.actions-inline .button {
  margin-right: 0.5rem;
}

.book-description {
  color: var(--muted);
  min-height: 72px;
}

.muted {
  color: var(--muted);
}

.status-badge {
  display: inline-flex;
  border-radius: 999px;
  padding: 0.2rem 0.7rem;
  font-size: 0.8rem;
  font-weight: 700;
  text-transform: capitalize;
}

.status-active {
  background: rgba(22, 163, 74, 0.12);
  color: var(--success);
}

.status-archived {
  background: rgba(220, 38, 38, 0.12);
  color: var(--danger);
}

.status-inactive {
  background: rgba(217, 119, 6, 0.12);
  color: var(--warning);
}

.loading-state,
.empty-state {
  background: white;
  border-radius: 18px;
  padding: 2rem;
  text-align: center;
  box-shadow: var(--shadow);
}

.spinner {
  width: 42px;
  height: 42px;
  border: 4px solid rgba(37, 99, 235, 0.2);
  border-top-color: var(--primary);
  border-radius: 999px;
  margin: 0 auto 1rem auto;
  animation: spin 0.8s linear infinite;
}

.table-card {
  overflow-x: auto;
}

.table {
  width: 100%;
  border-collapse: collapse;
}

.table th,
.table td {
  text-align: left;
  padding: 0.9rem 0.75rem;
  border-bottom: 1px solid var(--border);
  vertical-align: middle;
}

.table th {
  font-size: 0.9rem;
  color: var(--muted);
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1rem;
}

.full-width {
  grid-column: 1 / -1;
}

.filters-grid {
  display: grid;
  grid-template-columns: 2fr 1.2fr 1fr;
  gap: 0.8rem;
}

.detail-card {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.detail-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 1rem;
}

.detail-section h3 {
  margin-bottom: 0.5rem;
}

.alert {
  border-radius: 12px;
  padding: 0.85rem 1rem;
  font-size: 0.95rem;
}

.alert.error {
  background: rgba(220, 38, 38, 0.1);
  color: #991b1b;
}

.alert.success {
  background: rgba(22, 163, 74, 0.12);
  color: #166534;
}

.alert.warning {
  background: rgba(217, 119, 6, 0.12);
  color: #92400e;
}

.alert.info {
  background: rgba(37, 99, 235, 0.12);
  color: #1d4ed8;
}

.small-input {
  min-width: 130px;
  padding: 0.55rem 0.75rem;
}

.spacer-top {
  margin-top: 0.75rem;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@media (max-width: 1024px) {
  .app-shell {
    grid-template-columns: 1fr;
  }

  .sidebar {
    padding-bottom: 1rem;
  }

  .filters-grid,
  .detail-grid,
  .form-grid {
    grid-template-columns: 1fr;
  }

  .page-header {
    flex-direction: column;
    align-items: start;
  }
}


---

5. Environment/config files

File: .gitignore

# Python
__pycache__/
*.py[cod]
*.sqlite3
.backend_venv/
backend/.venv/
backend/.env

# Frontend
frontend/node_modules/
frontend/dist/
frontend/.env

# OS / editors
.DS_Store
.vscode/
.idea/

# Logs
*.log

File: backend/.env.example

APP_NAME=Library Management System
APP_ENV=development
FRONTEND_URL=http://localhost:5173
BACKEND_CORS_ORIGINS=http://localhost:5173

DATABASE_URL=postgresql+psycopg://library_app_user:CHANGE_ME@192.168.121.251:5432/library_db

JWT_SECRET_KEY=CHANGE_THIS_TO_A_LONG_RANDOM_SECRET
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60

AWS_REGION=ap-southeast-1
S3_BUCKET_NAME=library-management-system-dev
S3_PREFIX=books/
S3_PRESIGNED_URL_EXPIRE_SECONDS=900
MAX_UPLOAD_SIZE_MB=25

INITIAL_ADMIN_EMAIL=admin@example.com
INITIAL_ADMIN_USERNAME=admin
INITIAL_ADMIN_PASSWORD=ChangeMe123!
INITIAL_ADMIN_FULL_NAME=System Administrator

EMAIL_SENDER_NAME=Library Management System
EMAIL_SENDER_ADDRESS=noreply@example.com

File: frontend/.env.example

VITE_API_BASE_URL=http://localhost:8000/api/v1


---

6. Database/migration notes

Current schema alignment

এই codebase তোমার existing schema decisions-এর সাথে aligned:

users

categories

books

email_verification_tokens

password_reset_tokens

downloads


Migration flow

1. প্রথমে backend dependencies install করবে


2. .env তৈরি করবে


3. তারপর Alembic migration run করবে


4. app start করলে initial admin seed হবে, যদি already exist না করে



Important

app runtime user হবে library_app_user

PostgreSQL superuser postgres app runtime-এ use করবে না

books table-এ S3 metadata থাকবে:

storage_key

file_name

mime_type

file_size



If you already created schema manually

যদি তুমি already manual SQL দিয়ে schema create করে থাকো, দুইটা option আছে:

Option A — keep manual DB, skip first migration

যদি tables already match করে, Alembic stamp use করতে পারো:

alembic stamp head

Option B — let Alembic own the schema

খালি/clean DB হলে:

alembic upgrade head


---

7. Setup instructions

Backend setup

cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env

তারপর .env edit করো:

DATABASE_URL

JWT_SECRET_KEY

AWS_REGION

S3_BUCKET_NAME

INITIAL_ADMIN_*


Migration run

alembic upgrade head

Frontend setup

cd frontend
npm install
cp .env.example .env

তারপর frontend/.env-এ API base URL বসাও।


---

8. Run instructions

Run backend

cd backend
source .venv/bin/activate
uvicorn app.main:app --reload

Backend default URL:

http://localhost:8000

Run frontend

cd frontend
npm run dev

Frontend default URL:

http://localhost:5173

First-time admin behavior

app start-এর সময় যদি admin না থাকে:

env variables থেকে initial admin create হবে

admin verified থাকবে

must_change_password_on_first_login = true



---

9. API summary

Auth

POST /api/v1/auth/register

POST /api/v1/auth/login

POST /api/v1/auth/verify-email

POST /api/v1/auth/verify-email/resend

POST /api/v1/auth/forgot-password

POST /api/v1/auth/reset-password

POST /api/v1/auth/change-password


Users

GET /api/v1/users/me

PATCH /api/v1/users/me


Categories

GET /api/v1/categories

POST /api/v1/categories (admin)

PATCH /api/v1/categories/{category_id} (admin)


Books

GET /api/v1/books

GET /api/v1/books/{book_id}

POST /api/v1/books

PATCH /api/v1/books/{book_id} (admin)

POST /api/v1/books/{book_id}/archive (admin)

POST /api/v1/books/{book_id}/restore (admin)

POST /api/v1/books/{book_id}/download


Admin

GET /api/v1/admin/dashboard

GET /api/v1/admin/users

PATCH /api/v1/admin/users/{user_id}



---

10. Notes on how to modify the project later safely

If you change DB schema

এই order follow করবে:

1. models


2. schemas


3. service logic


4. API router


5. frontend types


6. frontend API client


7. UI pages


8. Alembic migration



If you change auth behavior

backend/app/core/security.py

backend/app/services/auth_service.py

frontend/src/context/AuthContext.tsx

frontend/src/api/auth.ts


If you change S3 behavior

backend/app/services/storage_service.py

backend/.env

বই download/upload flow-এর জন্য book_service.py


If you change business rules

সবচেয়ে আগে change করবে:

service layer

তারপর route protections

তারপর frontend UI actions


Best safe modification order

Backend

1. models


2. migration


3. repositories


4. services


5. API endpoints


6. schemas



Frontend

1. types


2. api/*


3. context/hooks


4. components


5. pages




---

What to configure in .env

Backend .env

Must configure:

DATABASE_URL

JWT_SECRET_KEY

AWS_REGION

S3_BUCKET_NAME

S3_PREFIX

INITIAL_ADMIN_EMAIL

INITIAL_ADMIN_USERNAME

INITIAL_ADMIN_PASSWORD

INITIAL_ADMIN_FULL_NAME


Frontend .env

Must configure:

VITE_API_BASE_URL



---

What order you should create the files in

Backend creation order

1. requirements.txt


2. core/config.py


3. core/database.py


4. models/*


5. db/base.py


6. repositories/*


7. schemas/*


8. services/*


9. api/deps.py


10. api/v1/endpoints/*


11. api/v1/api.py


12. main.py


13. alembic.ini


14. migrations/*


15. .env.example



Frontend creation order

1. package.json


2. tsconfig.json


3. vite.config.ts


4. src/types/*


5. src/api/*


6. src/context/AuthContext.tsx


7. src/hooks/useAuth.ts


8. src/components/common/*


9. src/components/layout/*


10. src/components/books/*


11. src/pages/*


12. src/App.tsx


13. src/main.tsx


14. src/styles/global.css


15. .env.example




---

এখন তুমি এটা review করো।
তারপর তুমি চাইলে আমি next step-এ এই same codebase-এর জন্য targeted file fixes / changes করতে পারব—যেমন:

শুধু auth tighten করা

শুধু S3 flow change করা

শুধু admin panel better করা

শুধু frontend polish করা

শুধু migration adjust করা
