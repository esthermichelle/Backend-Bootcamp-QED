# SwiftDrop — The Project We're Building

> A delivery platform (like Talabat/Uber Eats) built from scratch as a learning project.

---

## What is SwiftDrop?

A complete delivery platform where:
- **Customers** browse restaurants, place orders, and track delivery
- **Drivers** accept and deliver orders
- **Restaurant Owners** manage their menu and incoming orders
- **Admins** oversee everything

```mermaid
graph TB
    C[Customer] -->|Places Order| O[Order System]
    R[Restaurant] -->|Prepares Food| O
    D[Driver] -->|Delivers Order| O
    A[Admin] -->|Manages Everything| O
    
    style C fill:#06b6d4,color:#fff
    style R fill:#eab308,color:#000
    style D fill:#22c55e,color:#fff
    style A fill:#ef4444,color:#fff
    style O fill:#6366f1,color:#fff
```

---

## System Architecture

```mermaid
graph TB
    subgraph Client
        WEB[Web App<br/>React]
    end

    subgraph Backend
        API[FastAPI<br/>REST API]
    end

    subgraph Data
        PG[PostgreSQL<br/>Main Database]
        REDIS[Redis<br/>Cache + Sessions]
    end

    subgraph Services
        CELERY[Celery<br/>Background Tasks]
        MAIL[Email Service]
    end

    WEB <-->|JSON| API
    API <-->|SQL| PG
    API <-->|Cache| REDIS
    API -->|Queue| CELERY
    CELERY -->|Send| MAIL

    style WEB fill:#06b6d4,color:#fff
    style API fill:#6366f1,color:#fff
    style PG fill:#22c55e,color:#fff
    style REDIS fill:#ef4444,color:#fff
    style CELERY fill:#eab308,color:#000
    style MAIL fill:#8b5cf6,color:#fff
```

**How it works:**
- The **Web App** (React) sends JSON requests to the **Backend** (FastAPI)
- The **Backend** validates the request, applies business logic, and talks to **PostgreSQL** for data
- **Redis** is used for caching frequently accessed data (like restaurant lists) and storing sessions
- **Celery** handles things that shouldn't block the user — like sending emails, processing images, or generating reports
- The Backend responds with **JSON** back to the Frontend

---

## Database Schema

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    USER {
        int id PK
        string name
        string email
        string role
        string password_hash
        bool is_verified
        datetime created_at
    }
    
    RESTAURANT ||--o{ MENU_ITEM : has
    RESTAURANT ||--o{ ORDER : receives
    RESTAURANT {
        int id PK
        string name
        string description
        string address
        string phone
        int owner_id FK
        float rating
        bool is_active
        string image_url
    }
    
    MENU_ITEM {
        int id PK
        string name
        string description
        float price
        int restaurant_id FK
        string category
        bool available
        string image_url
    }
    
    ORDER ||--|{ ORDER_ITEM : contains
    ORDER {
        int id PK
        int customer_id FK
        int restaurant_id FK
        int driver_id FK
        string status
        float subtotal
        float delivery_fee
        float total
        string address
        datetime created_at
    }
    
    ORDER_ITEM {
        int id PK
        int order_id FK
        int menu_item_id FK
        int quantity
        float price_at_time
    }
    
    USER ||--o{ REVIEW : writes
    REVIEW {
        int id PK
        int user_id FK
        int restaurant_id FK
        int rating
        string comment
        datetime created_at
    }
```

---

## API Endpoints

### Auth (`/api/auth`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/register` | Create a new account |
| POST | `/login` | Login and receive JWT token |
| POST | `/logout` | Invalidate the current token |
| POST | `/refresh` | Refresh an expired token |
| POST | `/forgot-password` | Send password reset email |
| POST | `/reset-password` | Reset password with token |
| GET | `/verify-email/{token}` | Verify email address |
| GET | `/me` | Get current user profile |
| PUT | `/me` | Update current user profile |

### Restaurants (`/api/restaurants`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | List all restaurants (with pagination, search, filters) |
| GET | `/{id}` | Get restaurant details |
| POST | `/` | Create a restaurant (owner only) |
| PUT | `/{id}` | Update restaurant info (owner only) |
| DELETE | `/{id}` | Delete restaurant (owner/admin only) |
| GET | `/{id}/menu` | Get full menu for a restaurant |
| POST | `/{id}/menu` | Add a menu item (owner only) |
| PUT | `/{id}/menu/{item_id}` | Update a menu item |
| DELETE | `/{id}/menu/{item_id}` | Remove a menu item |
| GET | `/{id}/reviews` | Get all reviews for a restaurant |
| GET | `/{id}/stats` | Get restaurant stats — orders, revenue, ratings (owner only) |

### Orders (`/api/orders`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/` | Place a new order |
| GET | `/` | List my orders (customer) |
| GET | `/{id}` | Get order details |
| PUT | `/{id}/status` | Update order status (restaurant/driver) |
| POST | `/{id}/cancel` | Cancel an order (customer, before preparing) |
| GET | `/restaurant` | List incoming orders (restaurant owner) |
| GET | `/available` | List orders available for pickup (drivers) |
| POST | `/{id}/accept` | Accept an order for delivery (driver) |
| POST | `/{id}/reject` | Reject an order (driver) |

### Cart (`/api/cart`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Get current cart |
| POST | `/items` | Add item to cart |
| PUT | `/items/{id}` | Update item quantity |
| DELETE | `/items/{id}` | Remove item from cart |
| DELETE | `/` | Clear entire cart |

### Reviews (`/api/reviews`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/` | Write a review (after delivery) |
| PUT | `/{id}` | Edit my review |
| DELETE | `/{id}` | Delete my review |

### Delivery (`/api/delivery`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| PUT | `/location` | Update driver location |
| GET | `/{order_id}/track` | Track driver location (WebSocket) |
| GET | `/{order_id}/eta` | Get estimated time of arrival |
| GET | `/earnings` | Get driver earnings summary |
| GET | `/history` | Get driver delivery history |

### Admin (`/api/admin`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/users` | List all users |
| PUT | `/users/{id}/role` | Change user role |
| PUT | `/users/{id}/ban` | Ban/unban a user |
| GET | `/restaurants` | List all restaurants (with approval status) |
| PUT | `/restaurants/{id}/approve` | Approve a restaurant |
| GET | `/stats` | Platform-wide statistics |
| GET | `/orders` | List all orders (with filters) |

---

## Order Flow

Every order goes through this lifecycle:

```mermaid
stateDiagram-v2
    [*] --> Placed: Customer places order
    Placed --> Confirmed: Restaurant accepts
    Placed --> Cancelled: Restaurant rejects / Customer cancels
    Confirmed --> Preparing: Kitchen starts cooking
    Preparing --> Ready: Food is ready for pickup
    Ready --> PickedUp: Driver picks up the order
    PickedUp --> Delivered: Driver delivers to customer
    Delivered --> [*]
    Cancelled --> [*]
```

**Status transitions and who triggers them:**

| From | To | Triggered By |
|------|----|-------------|
| — | Placed | Customer (places order) |
| Placed | Confirmed | Restaurant (accepts) |
| Placed | Cancelled | Restaurant (rejects) or Customer (cancels) |
| Confirmed | Preparing | Restaurant (starts cooking) |
| Preparing | Ready | Restaurant (food ready) |
| Ready | PickedUp | Driver (picks up) |
| PickedUp | Delivered | Driver (delivers) |

---

## Project Structure

```
swiftdrop/
├── app/
│   ├── main.py              # FastAPI app — creates the app, includes routers
│   ├── config.py             # Settings from environment variables
│   ├── database.py           # Database connection and session management
│   ├── dependencies.py       # Shared dependencies (get_current_user, etc.)
│   │
│   ├── models/               # Database Models (SQLModel)
│   │   ├── user.py           # User, Role enum
│   │   ├── restaurant.py     # Restaurant, MenuItem, Category
│   │   ├── order.py          # Order, OrderItem, OrderStatus enum
│   │   └── review.py         # Review
│   │
│   ├── schemas/              # Pydantic Schemas (what the API accepts/returns)
│   │   ├── auth.py           # LoginRequest, RegisterRequest, TokenResponse
│   │   ├── restaurant.py     # RestaurantCreate, RestaurantResponse, MenuItemCreate
│   │   ├── order.py          # OrderCreate, OrderResponse, OrderStatusUpdate
│   │   └── review.py         # ReviewCreate, ReviewResponse
│   │
│   ├── api/                  # API Route handlers
│   │   ├── auth.py           # /register, /login, /logout, /me
│   │   ├── restaurants.py    # Restaurant + Menu CRUD
│   │   ├── orders.py         # Order creation, status updates
│   │   ├── cart.py           # Cart management
│   │   ├── reviews.py        # Review CRUD
│   │   ├── delivery.py       # Driver endpoints
│   │   └── admin.py          # Admin dashboard endpoints
│   │
│   ├── services/             # Business Logic (the "brain")
│   │   ├── auth_service.py   # Password hashing, JWT creation, verification
│   │   ├── order_service.py  # Order validation, status transitions, price calculation
│   │   ├── delivery_service.py  # Driver assignment, ETA calculation
│   │   └── notification_service.py  # Email/SMS via Celery
│   │
│   ├── workers/              # Background Tasks
│   │   ├── celery_app.py     # Celery configuration
│   │   └── tasks.py          # Email tasks, cleanup tasks
│   │
│   ├── migrations/           # Alembic database migrations
│   │   └── versions/         # Migration files (one per schema change)
│   │
│   └── tests/                # Tests
│       ├── test_auth.py
│       ├── test_restaurants.py
│       ├── test_orders.py
│       └── conftest.py       # Shared test fixtures
│
├── docker-compose.yml        # PostgreSQL + Redis + App
├── Dockerfile
├── requirements.txt
├── .env.example              # Template for environment variables
└── README.md
```

---

## What We Build in Each Sprint

### Sprint 0 — Setup
- Initialize the GitHub repo and project structure
- Set up Docker Compose (PostgreSQL + Redis)
- Configure Alembic for migrations
- Agree on Git workflow (feature branches → pull requests)
- Set up linting (Black, isort, flake8)

### Sprint 1 — Auth & Users
- User model + migration
- `POST /register` — create account with hashed password
- `POST /login` — verify credentials, return JWT
- `POST /logout` — invalidate token
- `GET /me` — get profile using token
- Email verification flow
- Role-based access control (Customer, Driver, Restaurant, Admin)
- **Tests:** register, login, protected routes

### Sprint 2 — Restaurants & Menu
- Restaurant + MenuItem models
- `GET /restaurants` — list with search, filter by category/rating, pagination
- `POST /restaurants` — create (owner only)
- `GET /restaurants/{id}/menu` — full menu
- `POST /restaurants/{id}/menu` — add menu item with image upload
- Owner dashboard: `GET /restaurants/{id}/stats`
- **Tests:** CRUD operations, authorization

### Sprint 3 — Orders & Payment
- Order + OrderItem models
- Cart endpoints (add, update, remove, clear)
- `POST /orders` — create order from cart, calculate totals
- Order status flow (state machine)
- `PUT /orders/{id}/status` — restaurant updates status
- Payment integration (Stripe test mode)
- Email notifications on order status change (Celery)
- **Tests:** order creation, status transitions, payment

### Sprint 4 — Delivery
- Driver registration and verification
- `GET /orders/available` — orders waiting for a driver
- `POST /orders/{id}/accept` — driver accepts delivery
- `PUT /delivery/location` — driver sends GPS coordinates
- `GET /delivery/{id}/track` — customer tracks driver (WebSocket)
- ETA calculation
- **Tests:** driver flow, location updates

### Sprint 5 — Reviews & Polish
- `POST /reviews` — write a review after delivery
- Restaurant average rating calculation
- Admin endpoints: user management, platform stats
- Rate limiting (Redis-based)
- Security audit (OWASP checklist)
- API documentation cleanup (Swagger)
- **Tests:** reviews, admin operations

### Sprint 6 — Deploy
- Dockerize the full application
- CI/CD pipeline with GitHub Actions (lint → test → build → deploy)
- Deploy to cloud server
- Nginx reverse proxy + SSL certificate
- Monitoring and health checks (`/health` endpoint)
- Final documentation

---

## How We Work

Everyone builds the **full project** on their own machine. When it's time to present, each member explains a specific part of the architecture to the group.

### Git Workflow

```mermaid
gitGraph
    commit id: "init"
    branch develop
    commit id: "project setup"
    branch feature/auth
    commit id: "user model"
    commit id: "register endpoint"
    commit id: "login endpoint"
    checkout develop
    merge feature/auth id: "PR #1: Auth"
    branch feature/restaurants
    commit id: "restaurant model"
    commit id: "CRUD endpoints"
    checkout develop
    merge feature/restaurants id: "PR #2: Restaurants"
    checkout main
    merge develop id: "Release v1"
```

**Rules:**
- Never push directly to `main` or `develop`
- Every feature gets its own branch
- Create a Pull Request when done
- Tests must pass before merging

### Commit Messages
```
feat: add user registration endpoint
fix: resolve JWT token expiration issue
docs: update API documentation
test: add order service unit tests
refactor: extract password hashing to service
```

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Language | Python | Easy to learn, massive ecosystem |
| Framework | FastAPI | Modern, fast, automatic API docs |
| Database | PostgreSQL | Reliable, powerful, industry standard |
| ORM | SQLModel | Built on SQLAlchemy, works perfectly with FastAPI |
| Migrations | Alembic | Safe database schema changes |
| Cache | Redis | Fast in-memory storage for sessions and caching |
| Tasks | Celery | Background job processing (emails, reports) |
| Containers | Docker | Consistent environments everywhere |
| CI/CD | GitHub Actions | Automated testing and deployment |
| Frontend | React | Component-based UI (later phase) |

---

## Example Request / Response

### Register a new user
```http
POST /api/auth/register
Content-Type: application/json

{
    "name": "Ahmed",
    "email": "ahmed@gmail.com",
    "password": "securePass123",
    "role": "customer"
}
```

**Response (201 Created):**
```json
{
    "id": 1,
    "name": "Ahmed",
    "email": "ahmed@gmail.com",
    "role": "customer",
    "is_verified": false,
    "message": "Account created. Check your email to verify."
}
```

### Place an order
```http
POST /api/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json

{
    "restaurant_id": 5,
    "items": [
        { "menu_item_id": 12, "quantity": 2 },
        { "menu_item_id": 15, "quantity": 1 }
    ],
    "delivery_address": "123 Main St, Cairo"
}
```

**Response (201 Created):**
```json
{
    "id": 42,
    "status": "placed",
    "restaurant": "Pizza Palace",
    "items": [
        { "name": "Margherita Pizza", "quantity": 2, "price": 120.00 },
        { "name": "Garlic Bread", "quantity": 1, "price": 45.00 }
    ],
    "subtotal": 285.00,
    "delivery_fee": 25.00,
    "total": 310.00,
    "estimated_delivery": "35-45 min"
}
```
