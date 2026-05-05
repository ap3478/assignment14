# FastAPI Calculator with Authentication & Database

A full-stack calculator API built with FastAPI, PostgreSQL, and JWT authentication. Supports BREAD (Browse, Read, Edit, Add, Delete) operations on calculations, user registration/login, and polymorphic calculation models stored in a database.

## Project Structure

```
assignment14/
├── app/
│   ├── auth/
│   │   ├── dependencies.py      # Auth dependency injection
│   │   ├── jwt.py               # JWT token creation/verification, password hashing
│   │   └── redis.py             # Token blacklist
│   ├── core/
│   │   ├── __init__.py
│   │   └── config.py            # Pydantic settings (DATABASE_URL, JWT keys)
│   ├── models/
│   │   ├── __init__.py
│   │   ├── calculation.py       # Polymorphic calculation models
│   │   └── user.py              # User model with auth methods
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── calculation.py       # Pydantic request/response schemas
│   │   ├── token.py             # Token schemas
│   │   └── user.py              # User schemas with password validation
│   ├── database.py              # SQLAlchemy engine, session, Base
│   └── main.py                  # FastAPI app with all endpoints
├── tests/
│   ├── unit/                    # Unit tests for arithmetic operations
│   ├── integration/             # Schema, model, and API tests
│   └── e2e/                     # Playwright browser tests
├── templates/
│   └── index.html               # Calculator frontend
├── docker-compose.yml           # Container orchestration
├── Dockerfile                   # Web service image definition
├── init-db.sh                   # Database initialization script
├── requirements.txt             # Python dependencies
├── pytest.ini                   # Pytest configuration
├── .env                         # Local environment variables (not in git)
└── README.md
```

## Prerequisites

Before you begin, install the following tools on your machine.

### 1. Python 3.9+

**macOS (Homebrew):**
```bash
brew install python@3.10
python3 --version
```

**Windows:**
- Download from [python.org](https://www.python.org/downloads/)
- During install, check **"Add Python to PATH"**
- Open a terminal and verify: `python --version`

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv
```

### 2. Visual Studio Code

- Download from [code.visualstudio.com](https://code.visualstudio.com/)
- Install the following extensions (search in Extensions sidebar):
  - **Python** (by Microsoft) — IntelliSense, linting, debugging
  - **Pylance** — type checking and auto-imports
  - **Docker** (by Microsoft) — manage containers from VS Code
  - **Thunder Client** or **REST Client** — test API endpoints (optional)

**Configure the Python interpreter in VS Code:**
1. Open the project folder in VS Code
2. Press `Cmd+Shift+P` (Mac) or `Ctrl+Shift+P` (Windows/Linux)
3. Type **"Python: Select Interpreter"**
4. Choose the one inside your `venv/` folder

### 3. Docker & Docker Compose

**macOS:**
- Download [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/)
- Install and start Docker Desktop from Applications
- Verify: `docker --version` and `docker compose version`

**Windows:**
- Download [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)
- Enable WSL 2 backend during installation if prompted
- Verify: `docker --version` and `docker compose version`

**Linux:**
```bash
sudo apt install docker.io docker-compose-v2
sudo systemctl start docker
sudo usermod -aG docker $USER   # lets you run docker without sudo (log out/in after)
```

### 4. Git

**macOS:**
```bash
xcode-select --install   # installs git along with developer tools
```

**Windows:**
- Download from [git-scm.com](https://git-scm.com/download/win)

**Linux:**
```bash
sudo apt install git
```

## Project Setup

### Clone the repository

```bash
git clone <your-repo-url>
cd assignment14
```

### Create and activate a virtual environment

```bash
python3 -m venv venv

# macOS / Linux
source venv/bin/activate

# Windows
venv\Scripts\activate
```

### Install dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
pip install bcrypt==4.0.1    # required for passlib compatibility
```

## Docker Architecture

The application runs as three Docker containers orchestrated via `docker-compose.yml`:

### Services Overview

| Service | Image | Container Port | Host Port | Purpose |
|---------|-------|---------------|-----------|---------|
| **web** | Built from `./Dockerfile` | 8000 | 8000 | FastAPI application server |
| **db** | `postgres:17` | 5432 | 5435 | PostgreSQL database |
| **pgadmin** | `dpage/pgadmin4` | 80 | 5052 | Database administration UI |

All three services communicate over a shared Docker bridge network called `app-network`.

### Service Details

**web (FastAPI Application)**
- Built from the project's `Dockerfile`
- Mounts the project directory (`.:/app`) for live code reloading
- Runs `uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload`
- Waits for the `db` service to pass its healthcheck before starting
- Environment variables configured for database connection, JWT authentication, and bcrypt hashing

**db (PostgreSQL 17)**
- Uses the official `postgres:17` image
- Creates the `fastapi_db` database on first startup via `POSTGRES_DB`
- Runs `init-db.sh` from `docker-entrypoint-initdb.d/` for additional database initialization (e.g., creating a test database)
- Data persisted in a named Docker volume (`postgres_data`) so it survives container restarts
- Healthcheck runs `pg_isready` every 10 seconds to verify the database is accepting connections
- Host port **5435** maps to container port **5432** (avoids conflict with any local Postgres on the default 5432)

**pgadmin (Database Admin UI)**
- Provides a web-based interface for inspecting and managing the PostgreSQL database
- Login credentials: `admin@example.com` / `admin`
- Data persisted in a named Docker volume (`pgadmin_data`)
- Only starts after the `db` service is healthy

### Networking

```
┌─────────────────────────────────────────────────┐
│                 app-network (bridge)             │
│                                                  │
│  ┌──────────┐   ┌──────────┐   ┌─────────────┐  │
│  │   web    │   │    db    │   │   pgadmin   │  │
│  │ :8000    │──▶│ :5432    │◀──│ :80         │  │
│  └──────────┘   └──────────┘   └─────────────┘  │
│       │              │               │           │
└───────┼──────────────┼───────────────┼───────────┘
        │              │               │
   Host:8000      Host:5435       Host:5052
```

- **Container-to-container:** Services use service names as hostnames (e.g., `db:5432`)
- **Host-to-container:** Your Mac/PC connects via mapped ports (e.g., `localhost:5435`)

### Volumes

| Volume | Mount Point | Purpose |
|--------|-------------|---------|
| `postgres_data` | `/var/lib/postgresql/data` | Persists database files across restarts |
| `pgadmin_data` | `/var/lib/pgadmin` | Persists pgAdmin config and saved servers |

### Environment Variables (web service)

| Variable | Value | Description |
|----------|-------|-------------|
| `DATABASE_URL` | `postgresql://postgres:postgres@db:5432/fastapi_db` | Database connection string (container network) |
| `TEST_DATABASE_URL` | `postgresql://postgres:postgres@db:5432/fastapi_test_db` | Test database connection string |
| `JWT_SECRET_KEY` | `super-secret-key-for-jwt-min-32-chars` | Secret key for signing access tokens |
| `JWT_REFRESH_SECRET_KEY` | `super-refresh-secret-key-min-32-chars` | Secret key for signing refresh tokens |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | Access token lifetime in minutes |
| `REFRESH_TOKEN_EXPIRE_DAYS` | `7` | Refresh token lifetime in days |
| `BCRYPT_ROUNDS` | `12` | Number of bcrypt hashing rounds |

## Running the Application

### Option A — Run everything with Docker (recommended)

```bash
docker compose up --build
```

This starts all three services. Access them at:
- **FastAPI app:** http://localhost:8000
- **Swagger UI:** http://localhost:8000/docs
- **pgAdmin:** http://localhost:5052

To stop:
```bash
docker compose down
```

To stop and wipe all database data:
```bash
docker compose down -v
```

### Option B — Run locally (database in Docker)

Start only the database:
```bash
docker compose up -d db
```

Run the FastAPI app from your terminal:
```bash
source venv/bin/activate
uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

### Common Docker Commands

```bash
# View running containers
docker compose ps

# View logs for a specific service
docker compose logs db
docker compose logs web

# Restart a single service
docker compose restart web

# Rebuild after changing Dockerfile or requirements
docker compose up --build

# Open a psql shell inside the database container
docker compose exec db psql -U postgres -d fastapi_db

# Check if the database is healthy
docker compose exec db pg_isready -U postgres -d fastapi_db

# Stop all containers and remove volumes (fresh start)
docker compose down -v
```

## Using Swagger UI

FastAPI generates interactive API documentation automatically. Open your browser and go to:

```
http://localhost:8000/docs
```

### Step 1 — Register a user

1. In Swagger UI, expand **POST /auth/register**
2. Click **"Try it out"**
3. Enter a JSON body like:
   ```json
   {
     "username": "testuser",
     "email": "test@example.com",
     "password": "SecurePass123!",
     "confirm_password": "SecurePass123!",
     "first_name": "Test",
     "last_name": "User"
   }
   ```
4. Click **"Execute"**
5. You should see a `201 Created` response with the user details

> **Password requirements:** minimum 8 characters, at least one uppercase letter, one lowercase letter, one digit, and one special character (`!@#$%^&*` etc.)

### Step 2 — Log in and get a token

1. Click the **Authorize** button (lock icon at the top right of the page)
2. Enter your `username` and `password`
3. Click **"Authorize"**, then **"Close"**
4. Swagger will now include your Bearer token in all subsequent requests

Alternatively, use the **POST /auth/login** endpoint directly:
1. Expand **POST /auth/login**
2. Click **"Try it out"**
3. Enter:
   ```json
   {
     "username": "testuser",
     "password": "SecurePass123!"
   }
   ```
4. Click **"Execute"**
5. Copy the `access_token` from the response

## BREAD Operations on Calculations

All calculation endpoints require authentication. Make sure you have authorized in Swagger UI (Step 2 above) before using them. BREAD stands for **B**rowse, **R**ead, **E**dit, **A**dd, **D**elete.

### Add — Create a New Calculation

**`POST /calculations`**

Creates a new calculation, computes the result using the polymorphic model, and persists it to the database. The authenticated user is automatically assigned as the owner.

**Request body:**
```json
{
  "type": "addition",
  "inputs": [10, 5, 3]
}
```

**Supported types and their behavior:**

| Type | Operation | Example Inputs | Result |
|------|-----------|---------------|--------|
| `addition` | Sums all inputs | `[10, 5, 3]` | `18.0` |
| `subtraction` | Subtracts subsequent values from the first | `[20, 5, 3]` | `12.0` |
| `multiplication` | Multiplies all inputs together | `[2, 3, 4]` | `24.0` |
| `division` | Divides the first value by subsequent values sequentially | `[100, 2, 5]` | `10.0` |

**Response (`201 Created`):**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "user_id": "98765432-abcd-ef01-2345-678901234567",
  "type": "addition",
  "inputs": [10, 5, 3],
  "result": 18.0,
  "created_at": "2025-01-15T10:30:00",
  "updated_at": "2025-01-15T10:30:00"
}
```

**Validation rules:**
- At least 2 inputs are required
- Division by zero is rejected (zero cannot appear in any position after the first)
- Calculation type must be one of the four supported types (case-insensitive)

**Error example (`400 Bad Request`):**
```json
{
  "detail": "Cannot divide by zero."
}
```

### Browse — List All Calculations

**`GET /calculations`**

Returns all calculations belonging to the authenticated user, ordered by creation time.

**Response (`200 OK`):**
```json
[
  {
    "id": "a1b2c3d4-...",
    "type": "addition",
    "inputs": [10, 5, 3],
    "result": 18.0,
    "created_at": "2025-01-15T10:30:00",
    "updated_at": "2025-01-15T10:30:00"
  },
  {
    "id": "b2c3d4e5-...",
    "type": "multiplication",
    "inputs": [2, 3, 4],
    "result": 24.0,
    "created_at": "2025-01-15T10:31:00",
    "updated_at": "2025-01-15T10:31:00"
  }
]
```

If you have no calculations yet, this returns an empty list `[]`.

### Read — Get a Specific Calculation

**`GET /calculations/{calc_id}`**

Retrieves a single calculation by its UUID. You can only access calculations that belong to you.

**Example:** `GET /calculations/a1b2c3d4-e5f6-7890-abcd-ef1234567890`

**Response (`200 OK`):**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "type": "addition",
  "inputs": [10, 5, 3],
  "result": 18.0,
  "created_at": "2025-01-15T10:30:00",
  "updated_at": "2025-01-15T10:30:00"
}
```

**Error responses:**
- `400 Bad Request` — the `calc_id` is not a valid UUID format
- `404 Not Found` — no calculation with that ID exists for your user

### Edit — Update a Calculation

**`PUT /calculations/{calc_id}`**

Updates the inputs of an existing calculation and recomputes the result. The calculation type cannot be changed after creation.

**Example:** `PUT /calculations/a1b2c3d4-e5f6-7890-abcd-ef1234567890`

**Request body:**
```json
{
  "inputs": [100, 50]
}
```

**Response (`200 OK`):**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "type": "addition",
  "inputs": [100, 50],
  "result": 150.0,
  "created_at": "2025-01-15T10:30:00",
  "updated_at": "2025-01-15T11:00:00"
}
```

Notice that `result` has been recomputed based on the new inputs, and `updated_at` has changed.

**Error responses:**
- `400 Bad Request` — invalid UUID format
- `404 Not Found` — calculation not found or doesn't belong to you

### Delete — Remove a Calculation

**`DELETE /calculations/{calc_id}`**

Permanently deletes a calculation. You can only delete calculations that belong to you.

**Example:** `DELETE /calculations/a1b2c3d4-e5f6-7890-abcd-ef1234567890`

**Response:** `204 No Content` (empty body — the calculation has been removed)

**Error responses:**
- `400 Bad Request` — invalid UUID format
- `404 Not Found` — calculation not found or doesn't belong to you

### BREAD Operations Summary

| Operation | Method | Endpoint | Request Body | Response Code | Description |
|-----------|--------|----------|-------------|---------------|-------------|
| **B**rowse | GET | `/calculations` | — | 200 | List all your calculations |
| **R**ead | GET | `/calculations/{id}` | — | 200 | Get one calculation by UUID |
| **E**dit | PUT | `/calculations/{id}` | `{"inputs": [...]}` | 200 | Update inputs and recompute |
| **A**dd | POST | `/calculations` | `{"type": "...", "inputs": [...]}` | 201 | Create a new calculation |
| **D**elete | DELETE | `/calculations/{id}` | — | 204 | Remove a calculation |

### Example Workflow Using curl

```bash
# Set your token (replace with actual token from /auth/login)
TOKEN="eyJhbGciOiJIUzI1NiIs..."

# Add — create a calculation
curl -X POST http://localhost:8000/calculations \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type": "multiplication", "inputs": [6, 7]}'

# Browse — list all calculations
curl http://localhost:8000/calculations \
  -H "Authorization: Bearer $TOKEN"

# Read — get one calculation (replace with actual UUID from the Add response)
curl http://localhost:8000/calculations/a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  -H "Authorization: Bearer $TOKEN"

# Edit — update inputs
curl -X PUT http://localhost:8000/calculations/a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"inputs": [10, 10]}'

# Delete — remove a calculation
curl -X DELETE http://localhost:8000/calculations/a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  -H "Authorization: Bearer $TOKEN"
```

## Running Tests

### Unit tests

```bash
source venv/bin/activate
pytest tests/unit/ -v
```

### Integration tests

```bash
pytest tests/integration/ -v
```

### E2E tests (requires Playwright browsers)

```bash
playwright install chromium
pytest tests/e2e/ -v
```

### All tests with coverage

```bash
pytest --cov=app --cov-report=term-missing
```