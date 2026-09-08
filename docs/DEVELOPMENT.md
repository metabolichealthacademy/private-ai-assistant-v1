# Development Guide

## Getting Started

### Prerequisites

- Python 3.10+
- Git
- PostgreSQL 12+ (or Docker)
- Virtual environment tool (venv, conda, etc.)

### Initial Setup

```bash
# Clone the repository
git clone https://github.com/metabolichealthacademy/private-ai-assistant-v1.git
cd private-ai-assistant-v1

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
make install

# Set up environment variables
cp .env.example .env
# Edit .env with your configuration

# Start development server
make dev
```

## Development Workflow

### 1. Create a Feature Branch

```bash
# Create branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name

# Branch naming conventions:
# - feature/description    - New features
# - fix/description        - Bug fixes
# - docs/description       - Documentation
# - refactor/description   - Code refactoring
# - test/description       - Test additions
```

### 2. Make Changes

```bash
# Make your code changes
# Write tests for new features
# Update documentation as needed
```

### 3. Run Tests & Linters

```bash
# Run all tests
make test

# Run linters
make lint

# Format code
make format

# Run security checks
make security
```

### 4. Commit Changes

```bash
# Stage changes
git add .

# Commit with descriptive message
git commit -m "Add: Description of changes

Longer explanation of what was changed and why.
Reference any related issues with #123."

# Commit message format:
# [TYPE]: Brief description (50 chars max)
# 
# Detailed explanation (72 chars per line)
#
# Types: Add, Fix, Update, Refactor, Docs, Test, Chore
# Related issues: #123, #456
```

### 5. Push and Create Pull Request

```bash
# Push to remote
git push origin feature/your-feature-name

# Create pull request on GitHub
# - Add clear title and description
# - Reference related issues
# - Ensure all checks pass
```

### 6. Code Review & Merge

```bash
# Address review comments
# Push additional commits
# Once approved, merge to develop

# Delete feature branch after merge
git branch -d feature/your-feature-name
```

## Project Structure

```
src/
├── app/              # FastAPI application
│   ├── main.py       # Application entry point
│   ├── routes/       # API endpoints
│   └── middleware.py # Custom middleware
├── models/           # Data models
│   ├── database.py   # ORM models
│   └── schemas.py    # Pydantic schemas
├── services/         # Business logic
├── utils/            # Utility functions
└── __init__.py

tests/
├── unit/             # Unit tests
├── integration/      # Integration tests
├── conftest.py       # Pytest configuration
└── fixtures/         # Test fixtures

docs/                 # Documentation
.github/workflows/    # CI/CD workflows
docker/               # Docker configuration
scripts/              # Utility scripts
```

## Adding New Features

### 1. Database Model

Create ORM model in `src/models/database.py`:

```python
from sqlalchemy import Column, String, Integer, DateTime
from sqlalchemy.orm import declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    email = Column(String(255), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
```

### 2. Pydantic Schema

Create validation schema in `src/models/schemas.py`:

```python
from pydantic import BaseModel, Field
from typing import Optional
from datetime import datetime

class UserBase(BaseModel):
    email: str = Field(..., min_length=5, max_length=255)

class UserCreate(UserBase):
    password: str = Field(..., min_length=12)

class UserResponse(UserBase):
    id: int
    created_at: datetime
    
    class Config:
        from_attributes = True
```

### 3. Service Layer

Create business logic in `src/services/user.py`:

```python
from sqlalchemy.orm import Session
from src.models.database import User
from src.models.schemas import UserCreate
from src.utils.exceptions import UserAlreadyExists

class UserService:
    @staticmethod
    def create_user(db: Session, user: UserCreate) -> User:
        existing = db.query(User).filter(User.email == user.email).first()
        if existing:
            raise UserAlreadyExists()
        
        new_user = User(email=user.email)
        db.add(new_user)
        db.commit()
        db.refresh(new_user)
        return new_user
```

### 4. API Route

Create endpoint in `src/app/routes/users.py`:

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from src.models.schemas import UserCreate, UserResponse
from src.services.user import UserService
from src.utils.exceptions import UserAlreadyExists

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserResponse)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    try:
        return UserService.create_user(db, user)
    except UserAlreadyExists:
        raise HTTPException(status_code=400, detail="User already exists")
```

### 5. Tests

Create tests in `tests/test_users.py`:

```python
import pytest
from fastapi.testclient import TestClient
from src.app.main import app

client = TestClient(app)

class TestUserCreation:
    def test_create_user_success(self):
        response = client.post("/users/", json={
            "email": "test@example.com",
            "password": "SecurePass123!"
        })
        assert response.status_code == 201
        assert response.json()["email"] == "test@example.com"
    
    def test_create_user_invalid_email(self):
        response = client.post("/users/", json={
            "email": "invalid",
            "password": "SecurePass123!"
        })
        assert response.status_code == 422
```

## Testing

### Unit Tests

```bash
# Run specific test file
pytest tests/unit/test_services.py -v

# Run specific test class
pytest tests/unit/test_services.py::TestUserService -v

# Run specific test
pytest tests/unit/test_services.py::TestUserService::test_create_user -v
```

### Integration Tests

```bash
# Run integration tests only
pytest tests/integration/ -v

# Run with database
pytest tests/integration/ -v --db=postgresql
```

### Coverage

```bash
# Generate coverage report
pytest tests/ --cov=src --cov-report=html

# View coverage
open htmlcov/index.html
```

### Test Fixtures

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from src.models.database import Base

@pytest.fixture(scope="session")
def test_db():
    engine = create_engine("sqlite:///./test.db")
    Base.metadata.create_all(bind=engine)
    yield engine
    Base.metadata.drop_all(bind=engine)

@pytest.fixture
def db_session(test_db):
    connection = test_db.connect()
    transaction = connection.begin()
    session = sessionmaker(bind=connection)()
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()
```

## Code Quality

### Linting

```bash
# Lint with flake8
flake8 src/ tests/

# Lint with pylint
pylint src/

# Type check with mypy
mypy src/
```

### Formatting

```bash
# Format with black
black src/ tests/

# Sort imports with isort
isort src/ tests/
```

### Pre-commit Hooks

```bash
# Install pre-commit
pip install pre-commit

# Create .pre-commit-config.yaml
cat > .pre-commit-config.yaml << EOF
repos:
  - repo: https://github.com/psf/black
    rev: 23.12.0
    hooks:
      - id: black
  
  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks:
      - id: isort
  
  - repo: https://github.com/PyCQA/flake8
    rev: 6.1.0
    hooks:
      - id: flake8
EOF

# Install git hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

## Database Migrations

### Create Migration

```bash
# Auto-generate migration
alembic revision --autogenerate -m "Add users table"

# Manual migration
alembic revision -m "Add users table"
```

### Apply Migrations

```bash
# Apply all migrations
alembic upgrade head

# Apply specific version
alembic upgrade ae1027a6acf

# Downgrade one migration
alembic downgrade -1
```

### View Migration History

```bash
# Show current version
alembic current

# Show all versions
alembic history --verbose
```

## Debugging

### Using Debugger

```python
# Add breakpoint in code
import pdb; pdb.set_trace()

# Or use breakpoint() in Python 3.7+
breakpoint()

# Python debugger commands:
# n - next line
# s - step into
# c - continue
# l - list code
# p variable - print variable
# h - help
```

### Logging

```python
import logging

logger = logging.getLogger(__name__)

logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")
logger.critical("Critical message")
```

### API Documentation

Interactive API docs available at:
- **Swagger UI:** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc

## Environment Variables

Commonly used environment variables:

```env
# Application
APP_ENV=development
DEBUG=True
API_PORT=8000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/private_ai
DATABASE_POOL_SIZE=10

# Security
SECRET_KEY=your-secret-key
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# External APIs
OPENAI_API_KEY=sk-...
```

## Performance Optimization

### Database Query Optimization

```python
# Use select() for better performance
from sqlalchemy import select

# Good: Explicit select
query = select(User).where(User.id == user_id)

# Avoid N+1 queries
from sqlalchemy.orm import joinedload
query = select(User).options(joinedload(User.posts))
```

### Caching

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive_operation(param: str) -> str:
    # Expensive computation
    return result
```

## Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [pytest Documentation](https://docs.pytest.org/)

## Support

For questions or help:
- Check existing documentation
- Search GitHub issues
- Ask in team discussions
- Contact the development team
