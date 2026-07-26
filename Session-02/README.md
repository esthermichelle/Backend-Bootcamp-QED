# FastAPI Revision Sheet — SwiftDrop Project

> **Purpose:** Deep revision material for a lecture. All examples use **SwiftDrop** — an instant package delivery platform with entities: `Package`, `Order`, `Courier`, `Customer`, `DeliveryStatus`.

---

## Table of Contents

1. [Path Parameters](#1-path-parameters)
2. [Query Parameters](#2-query-parameters)
3. [Pydantic Models](#3-pydantic-models)
4. [CRUD Operations](#4-crud-operations)
5. [Custom Response](#5-custom-response)
6. [Error Handling](#6-error-handling)
7. [Comparison Tables](#comparison-tables)
8. [HTTP Status Codes](#http-status-codes-table)
9. [Cheat Sheet](#cheat-sheet)
10. [Interactive Questions](#interactive-questions-for-the-lecture)

---

## 1. Path Parameters

### Definition

A path parameter is a variable segment embedded directly in the URL path, declared with `{param_name}` in the route decorator and received as a function argument with a type hint.

### Why It Exists

Without path parameters, you'd need a separate hardcoded route for every single resource — `/order-1`, `/order-2`, etc. Path parameters let you define **one route** that dynamically captures the identifier from the URL and passes it to your function.

### Benefits

- **Automatic type conversion:** FastAPI converts the raw string from the URL to the declared Python type (`int`, `str`, `UUID`, etc.).
- **Automatic validation:** If the conversion fails (e.g., `"abc"` for an `int` param), FastAPI returns a `422 Validation Error` without you writing any checking code.
- **Auto-documented:** The parameter appears in the OpenAPI schema at `/docs` with its type, constraints, and description.

### Code Example

```python
from fastapi import FastAPI, Path
from typing import Annotated
from enum import Enum

app = FastAPI()

# ─── Simple path parameter ───────────────────────────
@app.get("/orders/{order_id}")
def get_order(order_id: int):
    """Fetch a specific order by its numeric ID."""
    return {"order_id": order_id, "status": "in_transit"}


# ─── Path parameter with validation via Path() ───────
@app.get("/couriers/{courier_id}")
def get_courier(
    courier_id: Annotated[int, Path(
        title="Courier ID",
        description="The unique numeric ID of the courier",
        ge=1  # must be >= 1
    )]
):
    return {"courier_id": courier_id, "name": "Ahmed"}


# ─── Multiple path parameters ────────────────────────
@app.get("/couriers/{courier_id}/deliveries/{delivery_id}")
def get_courier_delivery(courier_id: int, delivery_id: int):
    """Get a specific delivery for a specific courier."""
    return {
        "courier_id": courier_id,
        "delivery_id": delivery_id,
        "status": "delivered"
    }


# ─── Enum path parameter (predefined values) ─────────
class DeliveryStatus(str, Enum):
    pending = "pending"
    picked_up = "picked_up"
    in_transit = "in_transit"
    delivered = "delivered"
    cancelled = "cancelled"

@app.get("/orders/status/{status}")
def get_orders_by_status(status: DeliveryStatus):
    """Get all orders with a specific status. Only accepts valid enum values."""
    return {"status": status, "orders": ["ORD-101", "ORD-205"]}
```

### Tips & Common Mistakes

| Tip | Explanation |
|-----|-------------|
| **Route order matters** | `/orders/active` must be declared **before** `/orders/{order_id}`, otherwise FastAPI tries to parse `"active"` as an `int` and fails with 422. |
| **Always add type hints** | `def get_order(order_id)` (no type hint) treats it as `str` with no validation. Always write `order_id: int`. |
| **Use `Path()` for extra constraints** | `Path(ge=1, le=10000)` enforces numeric ranges. Without it you'd need manual `if` checks. |
| **Don't put optional data in the path** | Paths identify **which** resource. Filtering/sorting belongs in query parameters. |
| **Use Enum for fixed sets** | If a path param can only be one of a few values (like status), use a `str, Enum` class — FastAPI validates it and shows the options in `/docs`. |

---

## 2. Query Parameters

### Definition

A query parameter is a key-value pair appended after `?` in the URL (e.g., `?city=cairo&limit=10`). In FastAPI, any function parameter that is **not** in the URL path is automatically treated as a query parameter.

### Why It Exists

Path parameters identify **which** resource. But how do you filter, sort, search, or paginate a collection? You can't put all of that in the path — you'd end up with absurd URLs like `/orders/cairo/in_transit/newest/page-2`. Query parameters solve this by providing a clean, optional, key-value mechanism for modifying **how** you retrieve a collection.

### Benefits

- **Optional by default:** Give a parameter a default value and it becomes optional. No default = required.
- **Multiple filters at once:** `?city=cairo&status=pending&sort=newest` — each is independent.
- **Type-safe:** `limit: int = 10` ensures the value is an integer. `?limit=abc` returns 422.
- **Validation with `Query()`:** Add constraints like `min_length`, `max_length`, `regex`, `ge`, `le`.

### Code Example

```python
from fastapi import FastAPI, Query
from typing import Annotated

app = FastAPI()

# ─── Basic query parameters ──────────────────────────
@app.get("/packages")
def list_packages(
    city: str | None = None,
    status: str | None = None,
    min_weight: float = 0.0,
    sort_by: str = "created_at",
    limit: int = 20,
    offset: int = 0
):
    """
    List packages with optional filtering, sorting, and pagination.

    GET /packages                                 → all, defaults
    GET /packages?city=cairo                      → filter by city
    GET /packages?status=pending&limit=5          → pending, first 5
    GET /packages?min_weight=2.5&sort_by=weight   → heavy packages, sorted
    """
    return {
        "filters": {"city": city, "status": status, "min_weight": min_weight},
        "sort_by": sort_by,
        "pagination": {"limit": limit, "offset": offset}
    }


# ─── Required query parameter ────────────────────────
@app.get("/packages/search")
def search_packages(
    q: Annotated[str, Query(
        min_length=2,
        max_length=100,
        description="Search term for package description or tracking code"
    )]
):
    """
    Search is required — you can't call /packages/search without ?q=...
    GET /packages/search?q=laptop  → ✅
    GET /packages/search           → ❌ 422 Error
    """
    return {"query": q, "results": []}


# ─── Boolean query parameter ─────────────────────────
@app.get("/couriers")
def list_couriers(
    available_only: bool = False
):
    """
    FastAPI converts: ?available_only=true / yes / 1 / on  → True
                      ?available_only=false / no / 0 / off → False
    """
    return {"available_only": available_only, "couriers": []}


# ─── List query parameter (multiple values) ──────────
@app.get("/orders/filter")
def filter_orders(
    status: Annotated[list[str] | None, Query()] = None
):
    """
    GET /orders/filter?status=pending&status=in_transit
    → status = ["pending", "in_transit"]
    """
    return {"statuses": status}
```

### Tips & Common Mistakes

| Tip | Explanation |
|-----|-------------|
| **Required vs Optional** | `q: str` → required (no default). `q: str = ""` → optional. `q: str \| None = None` → optional, returns `None` if not provided. |
| **Don't use query params for large data** | Long JSON objects, file contents, or arrays with 50 items belong in the **request body**, not the URL. URLs have length limits (~2000 chars). |
| **Use `Query()` for validation** | `Query(min_length=1, max_length=50)` prevents empty strings and absurdly long inputs. |
| **Bool conversion** | `?active=yes`, `?active=1`, `?active=true`, `?active=on` all become `True`. This is FastAPI-specific — don't assume other frameworks do this. |
| **Pagination pattern** | Always use `limit` + `offset` (or `page` + `page_size`). Never return unbounded collections in production. |

---

## 3. Pydantic Models

### Definition

A Pydantic model is a Python class inheriting from `BaseModel` that defines the **structure, types, and validation rules** for incoming or outgoing JSON data. FastAPI uses it to automatically parse, validate, and document request bodies and response shapes.

### Why It Exists

Without Pydantic, you'd receive raw dictionaries from `request.json()` and manually check every field: *"Is `name` present? Is it a string? Is `weight` a positive float?"* — dozens of `if` statements. Pydantic replaces all of that with a single class declaration. If the data doesn't match, the client gets a detailed 422 error automatically.

### Benefits

- **Declarative validation:** Define rules once in the model; every request is validated automatically.
- **Auto-documentation:** The model's fields, types, and constraints appear in the OpenAPI schema.
- **Type safety:** Your IDE gives you autocomplete and type checking inside the function.
- **Serialization:** `.model_dump()` converts the model to a dict; `.model_dump_json()` to a JSON string.
- **Separation of concerns:** Different schemas for Create, Update, and Response keep your API clean.

### Code Example

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field, field_validator
from datetime import datetime

app = FastAPI()

# ─── CREATE schema (what the client sends) ───────────
class PackageCreate(BaseModel):
    sender_name: str = Field(min_length=2, max_length=100)
    sender_phone: str = Field(pattern=r"^\+?[0-9]{10,15}$")
    recipient_name: str = Field(min_length=2, max_length=100)
    recipient_address: str = Field(min_length=5)
    weight_kg: float = Field(gt=0, le=50, description="Weight in kilograms, max 50")
    description: str | None = Field(default=None, max_length=500)
    fragile: bool = False
    priority: str = Field(default="standard")

    @field_validator("priority")
    @classmethod
    def validate_priority(cls, v: str) -> str:
        allowed = {"standard", "express", "same_day"}
        if v not in allowed:
            raise ValueError(f"Priority must be one of: {allowed}")
        return v


# ─── UPDATE schema (partial update — all fields optional) ─────
class PackageUpdate(BaseModel):
    recipient_name: str | None = Field(default=None, min_length=2, max_length=100)
    recipient_address: str | None = Field(default=None, min_length=5)
    weight_kg: float | None = Field(default=None, gt=0, le=50)
    description: str | None = None
    fragile: bool | None = None
    priority: str | None = None

    @field_validator("priority")
    @classmethod
    def validate_priority(cls, v: str | None) -> str | None:
        if v is not None:
            allowed = {"standard", "express", "same_day"}
            if v not in allowed:
                raise ValueError(f"Priority must be one of: {allowed}")
        return v


# ─── RESPONSE schema (what the server returns) ───────
class PackageResponse(BaseModel):
    id: int
    tracking_code: str
    sender_name: str
    recipient_name: str
    recipient_address: str
    weight_kg: float
    fragile: bool
    priority: str
    status: str
    created_at: datetime
    estimated_delivery: datetime | None = None


# ─── Nested model ────────────────────────────────────
class OrderItem(BaseModel):
    package_id: int = Field(ge=1)
    quantity: int = Field(ge=1, le=100)
    special_instructions: str | None = None

class OrderCreate(BaseModel):
    customer_id: int = Field(ge=1)
    pickup_address: str = Field(min_length=5)
    items: list[OrderItem] = Field(min_length=1, max_length=20)
    coupon_code: str | None = None


# ─── Usage in endpoints ──────────────────────────────
@app.post("/packages", status_code=201)
def create_package(package: PackageCreate):
    return {
        "id": 1,
        "tracking_code": "SD-20240101-001",
        **package.model_dump(),
        "status": "pending",
        "created_at": datetime.now().isoformat()
    }
```

### Field() vs Body() Comparison

| Feature | `Field()` | `Body()` |
|---------|-----------|----------|
| **Used in** | Inside a Pydantic model class | In the function signature |
| **Purpose** | Define constraints on a model field | Define constraints on a body parameter |
| **Where declared** | `weight: float = Field(gt=0)` | `weight: Annotated[float, Body(gt=0)]` |
| **When to use** | ✅ Almost always — cleaner, reusable | When you need a single value from the body without a full model |
| **Reusability** | The model can be used in multiple endpoints | Tied to one function |

```python
# Field() — inside the model (preferred)
class PackageCreate(BaseModel):
    weight_kg: float = Field(gt=0, le=50)

# Body() — in the function (rare, for single values)
from fastapi import Body
@app.post("/packages/{package_id}/note")
def add_note(
    package_id: int,
    note: Annotated[str, Body(min_length=1, max_length=500, embed=True)]
):
    return {"package_id": package_id, "note": note}
# Client sends: {"note": "Handle with care"}
```

### Tips & Common Mistakes

| Tip | Explanation |
|-----|-------------|
| **Separate Create / Update / Response models** | Never use the same model for all three. `Create` has no `id`. `Update` has all fields optional. `Response` includes server-generated fields. |
| **Use `field_validator` for custom logic** | When `Field()` constraints aren't enough (e.g., "priority must be one of 3 values"), use `@field_validator`. |
| **`model_dump(exclude_unset=True)`** | Critical for PATCH updates — only returns fields the client actually sent, so you don't overwrite existing data with `None`. |
| **Nested models validate recursively** | If `OrderCreate` has `items: list[OrderItem]`, Pydantic validates every item in the list automatically. |
| **Don't use `dict` as a type hint** | `data: dict` gives you zero validation. Always create a Pydantic model. |
| **Default values are NOT validated** | `Field(default="invalid_value")` won't trigger the validator at creation time — only when data comes from the client. |

---

## 4. CRUD Operations

### Definition

CRUD stands for **Create, Read, Update, Delete** — the four fundamental operations for managing any resource. In FastAPI, each maps to an HTTP method and a route decorator.

### Why It Exists

Every data-driven application needs to let users create new records, read existing ones, modify them, and remove them. CRUD is the universal pattern that standardizes these operations across any API, making it predictable for frontend developers, mobile apps, and third-party integrations.

### Benefits

- **Standardized:** Any developer who sees `POST /packages` immediately knows it creates a package.
- **HTTP-native:** Each operation maps to the semantically correct HTTP method.
- **Predictable URLs:** Resource-based routing (`/packages`, `/packages/{id}`) is intuitive.
- **Tooling support:** API clients (Swagger, Postman, frontend code generators) understand CRUD automatically.

### HTTP Methods Comparison

| Method | CRUD Op | Idempotent? | Has Body? | Success Code | Example |
|--------|---------|-------------|-----------|-------------|---------|
| `POST` | **C**reate | No | Yes | 201 | Create a new package |
| `GET` | **R**ead | Yes | No | 200 | Get package details |
| `PUT` | **U**pdate (full) | Yes | Yes | 200 | Replace all package fields |
| `PATCH` | **U**pdate (partial) | Yes | Yes | 200 | Update just the address |
| `DELETE` | **D**elete | Yes | No | 200 or 204 | Remove a package |

> **Idempotent** = calling it multiple times has the same effect as calling it once. `POST` is NOT idempotent — calling it 3 times creates 3 packages.

### PUT vs PATCH Comparison

| Aspect | PUT | PATCH |
|--------|-----|-------|
| **What it does** | Replaces the **entire** resource | Updates **only** the fields you send |
| **Fields required** | All fields (or they reset to defaults) | Only the fields you want to change |
| **If you omit a field** | It gets overwritten (set to null/default) | It stays unchanged |
| **Pydantic model** | All fields required | All fields optional (`| None = None`) |
| **Use case** | "Replace this package with new data" | "Just change the address" |

### Code Example — Full SwiftDrop CRUD

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from datetime import datetime

app = FastAPI()

# ─── In-memory database ──────────────────────────────
packages_db: dict[int, dict] = {}
next_id = 1

# ─── Schemas ─────────────────────────────────────────
class PackageCreate(BaseModel):
    sender_name: str = Field(min_length=2)
    recipient_name: str = Field(min_length=2)
    recipient_address: str = Field(min_length=5)
    weight_kg: float = Field(gt=0, le=50)
    fragile: bool = False
    priority: str = "standard"

class PackageUpdate(BaseModel):
    """All fields optional — for PATCH (partial update)."""
    recipient_name: str | None = None
    recipient_address: str | None = None
    weight_kg: float | None = Field(default=None, gt=0, le=50)
    fragile: bool | None = None
    priority: str | None = None


# ─── CREATE ──────────────────────────────────────────
@app.post("/packages", status_code=201)
def create_package(data: PackageCreate):
    global next_id
    package = {
        "id": next_id,
        "tracking_code": f"SD-{next_id:05d}",
        **data.model_dump(),
        "status": "pending",
        "created_at": datetime.now().isoformat()
    }
    packages_db[next_id] = package
    next_id += 1
    return package


# ─── READ (collection) ──────────────────────────────
@app.get("/packages")
def list_packages(
    status: str | None = None,
    priority: str | None = None,
    limit: int = 20,
    offset: int = 0
):
    results = list(packages_db.values())

    if status:
        results = [p for p in results if p["status"] == status]
    if priority:
        results = [p for p in results if p["priority"] == priority]

    return {
        "total": len(results),
        "packages": results[offset : offset + limit]
    }


# ─── READ (single) ──────────────────────────────────
@app.get("/packages/{package_id}")
def get_package(package_id: int):
    if package_id not in packages_db:
        raise HTTPException(status_code=404, detail="Package not found")
    return packages_db[package_id]


# ─── UPDATE (full — PUT) ────────────────────────────
@app.put("/packages/{package_id}")
def replace_package(package_id: int, data: PackageCreate):
    """Full replacement — all fields required."""
    if package_id not in packages_db:
        raise HTTPException(status_code=404, detail="Package not found")

    existing = packages_db[package_id]
    updated = {
        "id": existing["id"],
        "tracking_code": existing["tracking_code"],
        **data.model_dump(),
        "status": existing["status"],
        "created_at": existing["created_at"]
    }
    packages_db[package_id] = updated
    return updated


# ─── UPDATE (partial — PATCH) ───────────────────────
@app.patch("/packages/{package_id}")
def update_package(package_id: int, data: PackageUpdate):
    """Partial update — only sent fields are updated."""
    if package_id not in packages_db:
        raise HTTPException(status_code=404, detail="Package not found")

    existing = packages_db[package_id]
    update_data = data.model_dump(exclude_unset=True)  # ← KEY: only fields the client sent

    for field, value in update_data.items():
        existing[field] = value

    return existing


# ─── DELETE ──────────────────────────────────────────
@app.delete("/packages/{package_id}")
def delete_package(package_id: int):
    if package_id not in packages_db:
        raise HTTPException(status_code=404, detail="Package not found")

    del packages_db[package_id]
    return {"message": f"Package {package_id} deleted"}
```

### Tips & Common Mistakes

| Tip | Explanation |
|-----|-------------|
| **Use `model_dump(exclude_unset=True)` for PATCH** | Without it, you can't distinguish "client sent `fragile=None`" from "client didn't send `fragile` at all". |
| **Don't use GET for actions** | `GET /packages/5/cancel` is wrong. Use `PATCH /packages/5` with `{"status": "cancelled"}` or `POST /packages/5/cancel`. |
| **Return the created/updated resource** | After POST or PUT/PATCH, return the full object — the client needs the server-generated fields (id, tracking_code, created_at). |
| **PUT replaces, PATCH patches** | If you only support one, choose PATCH — it's more flexible and what most APIs need. |
| **POST is not idempotent** | Two identical POST requests create two packages. PUT with the same data produces the same result both times. |

---

## 5. Custom Response

### Definition

A custom response lets you control **what the client receives** beyond the default JSON — including the status code, headers, response format (HTML, XML, plain text, file download), and the documented response schema.

### Why It Exists

FastAPI defaults to `JSONResponse` with status code `200` and `Content-Type: application/json`. But real APIs need more: `201` for creation, `204` for deletion with no body, `Location` headers for redirects, file downloads as binary streams, or HTML pages. Custom responses give you full control over the HTTP response.

### Benefits

- **Correct HTTP semantics:** Using `201 Created` instead of `200 OK` for POST.
- **Documentation accuracy:** `response_model` tells the OpenAPI schema exactly what shape the response has.
- **Performance:** `ORJSONResponse` is faster than the default JSON encoder for large payloads.
- **Flexibility:** Return files, redirects, streaming responses, or custom headers.

### response_model vs JSONResponse Comparison

| Feature | `response_model` parameter | `JSONResponse` / manual return |
|---------|---------------------------|-------------------------------|
| **Validation** | ✅ Pydantic validates the output before sending | ❌ No output validation — you could return anything |
| **Documentation** | ✅ Appears in OpenAPI/Swagger | ❌ Shows as generic response unless you add `responses={}` |
| **Filtering** | ✅ Strips fields not in the model (security!) | ❌ Returns exactly what you give it |
| **Use case** | Standard responses where you want type safety | Custom status codes, headers, non-JSON formats |

### Code Example

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse, Response, RedirectResponse
from pydantic import BaseModel
from datetime import datetime

app = FastAPI()

class PackageResponse(BaseModel):
    id: int
    tracking_code: str
    sender_name: str
    recipient_name: str
    status: str
    created_at: datetime

class PackageCreate(BaseModel):
    sender_name: str
    recipient_name: str
    recipient_address: str
    weight_kg: float


# ─── response_model: filters the output ─────────────
@app.get("/packages/{package_id}", response_model=PackageResponse)
def get_package(package_id: int):
    """
    Even if the internal dict has extra fields (like internal_notes
    or cost_price), response_model ensures ONLY PackageResponse
    fields are sent to the client.
    """
    internal_data = {
        "id": package_id,
        "tracking_code": "SD-00001",
        "sender_name": "Ahmed",
        "recipient_name": "Sara",
        "status": "in_transit",
        "created_at": datetime.now(),
        "internal_notes": "VIP customer",     # ← will be STRIPPED
        "cost_price": 45.00                    # ← will be STRIPPED
    }
    return internal_data  # response_model filters out non-declared fields


# ─── Custom status code in decorator ─────────────────
@app.post("/packages", response_model=PackageResponse, status_code=201)
def create_package(data: PackageCreate):
    """Returns 201 Created instead of the default 200."""
    return {
        "id": 1,
        "tracking_code": "SD-00001",
        "sender_name": data.sender_name,
        "recipient_name": data.recipient_name,
        "status": "pending",
        "created_at": datetime.now()
    }


# ─── 204 No Content (empty body) ─────────────────────
@app.delete("/packages/{package_id}", status_code=204)
def delete_package(package_id: int):
    """
    204 means success but no body.
    Return Response with no content.
    """
    # ... delete logic here ...
    return Response(status_code=204)


# ─── Custom headers ──────────────────────────────────
@app.get("/packages/{package_id}/receipt")
def get_receipt(package_id: int):
    content = {"package_id": package_id, "receipt_number": "RCP-2024-001"}
    return JSONResponse(
        content=content,
        headers={
            "X-Receipt-ID": "RCP-2024-001",
            "Cache-Control": "no-store"
        }
    )


# ─── Redirect ────────────────────────────────────────
@app.get("/track/{tracking_code}")
def track_package(tracking_code: str):
    """Redirect to the tracking page."""
    return RedirectResponse(
        url=f"https://track.swiftdrop.com/{tracking_code}",
        status_code=307
    )


# ─── Multiple response schemas ───────────────────────
class ErrorResponse(BaseModel):
    detail: str
    error_code: str

@app.get(
    "/orders/{order_id}",
    response_model=PackageResponse,
    responses={
        404: {"model": ErrorResponse, "description": "Order not found"},
        403: {"model": ErrorResponse, "description": "Not authorized to view this order"}
    }
)
def get_order(order_id: int):
    """Documents both success and error responses in Swagger."""
    return {"id": order_id, ...}
```

### Tips & Common Mistakes

| Tip | Explanation |
|-----|-------------|
| **Always use `response_model` for GET endpoints** | It acts as a security filter — prevents accidentally leaking internal fields like passwords or cost prices. |
| **`status_code` in decorator vs in `Response()`** | The decorator value is for documentation (Swagger shows it). `Response(status_code=...)` is the actual runtime value. They should match. |
| **204 must have no body** | If you set `status_code=204` but return a dict, some clients will break. Return `Response(status_code=204)`. |
| **Use `responses={}` for error docs** | Without it, Swagger only shows the success response. With it, your API docs show what errors are possible — critical for frontend developers. |
| **`response_model_exclude_unset=True`** | If a field is optional and wasn't set, it won't appear in the response. Useful for clean, minimal JSON. |

---

## 6. Error Handling

### Definition

Error handling in FastAPI is the mechanism for returning meaningful, structured error responses when something goes wrong — whether it's invalid input (422), a missing resource (404), unauthorized access (401), or an unexpected server crash (500).

### Why It Exists

Without explicit error handling, any unhandled exception causes a generic `500 Internal Server Error` with no useful information. The client gets `Internal Server Error` and has no idea what went wrong. Proper error handling gives the client **actionable information**: what failed, why, and how to fix it.

### Benefits

- **Predictable error format:** Clients always know the shape of error responses.
- **Correct status codes:** 404 for missing resources, 400 for bad requests, 422 for validation failures — not 500 for everything.
- **Security:** Custom handlers prevent leaking stack traces or internal details to the client.
- **Debugging:** Structured errors make debugging easier for frontend developers and API consumers.

### HTTPException vs Custom Exception Handler Comparison

| Feature | `HTTPException` | `@app.exception_handler()` |
|---------|----------------|---------------------------|
| **Scope** | Raised in a specific endpoint, one place at a time | Catches exceptions **globally** across all endpoints |
| **Use case** | "This specific package was not found" | "Whenever ANY `ValueError` happens anywhere, return this format" |
| **Flexibility** | Fixed JSON format: `{"detail": "..."}` | Full control — return any response format |
| **When to use** | Most of the time — for specific, expected errors | When you want consistent error formatting app-wide, or need to handle third-party exceptions |

### Code Example

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel

app = FastAPI()

packages_db: dict[int, dict] = {
    1: {"id": 1, "tracking_code": "SD-00001", "status": "in_transit", "recipient_name": "Sara"}
}

# ─── Basic HTTPException ─────────────────────────────
@app.get("/packages/{package_id}")
def get_package(package_id: int):
    if package_id not in packages_db:
        raise HTTPException(
            status_code=404,
            detail=f"Package with ID {package_id} not found"
        )
    return packages_db[package_id]


# ─── HTTPException with custom headers ───────────────
@app.patch("/packages/{package_id}/cancel")
def cancel_package(package_id: int):
    if package_id not in packages_db:
        raise HTTPException(status_code=404, detail="Package not found")

    package = packages_db[package_id]

    if package["status"] == "delivered":
        raise HTTPException(
            status_code=400,
            detail="Cannot cancel a delivered package",
            headers={"X-Error-Code": "PACKAGE_ALREADY_DELIVERED"}
        )

    if package["status"] == "cancelled":
        raise HTTPException(
            status_code=409,  # Conflict
            detail="Package is already cancelled"
        )

    package["status"] = "cancelled"
    return package


# ─── Multiple validation checks ──────────────────────
class PackageCreate(BaseModel):
    sender_name: str
    recipient_name: str
    weight_kg: float
    priority: str = "standard"

@app.post("/packages")
def create_package(data: PackageCreate):
    errors = []

    if data.weight_kg > 50:
        errors.append("Weight exceeds 50kg limit")

    if data.priority not in {"standard", "express", "same_day"}:
        errors.append(f"Invalid priority: {data.priority}")

    if data.sender_name == data.recipient_name:
        errors.append("Sender and recipient cannot be the same person")

    if errors:
        raise HTTPException(
            status_code=400,
            detail={"message": "Validation failed", "errors": errors}
        )

    return {"message": "Package created", "data": data.model_dump()}


# ═══════════════════════════════════════════════════════
# CUSTOM EXCEPTION HANDLERS (Global)
# ═══════════════════════════════════════════════════════

# ─── Custom exception class ──────────────────────────
class PackageNotFoundError(Exception):
    def __init__(self, package_id: int):
        self.package_id = package_id

@app.exception_handler(PackageNotFoundError)
async def package_not_found_handler(request: Request, exc: PackageNotFoundError):
    """This handler catches PackageNotFoundError ANYWHERE in the app."""
    return JSONResponse(
        status_code=404,
        content={
            "error": "PACKAGE_NOT_FOUND",
            "message": f"Package {exc.package_id} does not exist",
            "documentation": "https://api.swiftdrop.com/docs#packages"
        }
    )

# Now you can just raise it without try/except:
@app.get("/packages/{package_id}/track")
def track_package(package_id: int):
    if package_id not in packages_db:
        raise PackageNotFoundError(package_id)  # Caught by the global handler
    return {"tracking": "..."}


# ─── Override the default 422 validation error ───────
@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    """
    Replace FastAPI's default 422 response with a cleaner format.
    Instead of Pydantic's verbose error, return something simpler.
    """
    errors = []
    for error in exc.errors():
        field = " → ".join(str(loc) for loc in error["loc"])
        errors.append({
            "field": field,
            "message": error["msg"],
            "type": error["type"]
        })

    return JSONResponse(
        status_code=422,
        content={
            "error": "VALIDATION_ERROR",
            "message": "The request data is invalid",
            "details": errors
        }
    )


# ─── Catch-all for unexpected errors ─────────────────
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """
    Safety net — catches any unhandled exception.
    In production, log the real error but don't expose it to the client.
    """
    # In real app: logger.error(f"Unhandled: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": "INTERNAL_ERROR",
            "message": "An unexpected error occurred. Please try again later."
        }
    )
```

### Tips & Common Mistakes

| Tip | Explanation |
|-----|-------------|
| **Don't return 200 for errors** | Never `return {"error": "not found"}` with status 200. Use `raise HTTPException(status_code=404)`. Frontend code relies on status codes to detect errors. |
| **Don't expose stack traces in production** | The catch-all handler should log the real error but return a generic message to the client. Stack traces leak internal details. |
| **Use custom exception classes for repeated patterns** | If you check "package not found" in 10 endpoints, create `PackageNotFoundError` and handle it globally — DRY. |
| **`detail` can be any JSON-serializable value** | It doesn't have to be a string. `detail={"errors": [...]}` works too. |
| **Order of exception handlers matters** | More specific handlers (`PackageNotFoundError`) are checked before generic ones (`Exception`). |

---

## Comparison Tables

### Path vs Query vs Body (Pydantic Model)

| Aspect | Path Parameter | Query Parameter | Request Body (Pydantic) |
|--------|---------------|----------------|------------------------|
| **Location** | In the URL path: `/packages/{id}` | After `?`: `?status=pending` | In the HTTP body (JSON) |
| **Purpose** | Identify **which** resource | **Filter, sort, paginate, search** | Send **data payload** for create/update |
| **Required?** | Always required | Optional by default (if has default) | Required (unless all fields have defaults) |
| **Data size** | Small (IDs, slugs) | Small to medium (filters) | Any size (full objects, nested data) |
| **HTTP methods** | Any | Mostly GET | POST, PUT, PATCH |
| **Multiple values** | One per `{param}` | Yes: `?status=a&status=b` | Yes: nested objects, lists |
| **Example** | `GET /packages/42` | `GET /packages?city=cairo` | `POST /packages {"sender": "..."}` |

### When to use each — Decision Guide

```
"I need to get/modify ONE specific thing"    → Path Parameter
"I need to filter/search/sort a collection"  → Query Parameter
"I need to send a complex data object"       → Request Body (Pydantic)
```

---

## HTTP Status Codes Table

| Code | Name | When It's Used | SwiftDrop Example |
|------|------|---------------|-------------------|
| **200** | OK | Successful GET, PUT, PATCH | `GET /packages/1` → returns the package |
| **201** | Created | Successful POST that created a resource | `POST /packages` → new package created |
| **202** | Accepted | Request received but processing hasn't finished | `POST /packages/1/schedule-pickup` → pickup queued for processing |
| **204** | No Content | Successful DELETE or action with no response body | `DELETE /packages/1` → deleted, nothing to return |
| **400** | Bad Request | Client sent logically invalid data (passes validation but fails business rules) | `POST /orders` with a delivery date in the past |
| **401** | Unauthorized | No authentication provided or token expired | `GET /orders` without an API key or JWT |
| **403** | Forbidden | Authenticated but not authorized for this action | A customer trying to access `GET /admin/all-orders` |
| **404** | Not Found | The requested resource doesn't exist | `GET /packages/99999` → no package with that ID |
| **409** | Conflict | Action conflicts with current resource state | Trying to cancel a package that's already delivered |
| **422** | Unprocessable Entity | Request body fails Pydantic validation | `POST /packages` with `weight_kg: "heavy"` instead of a number |
| **429** | Too Many Requests | Rate limit exceeded | Client sent 100 requests in 1 second |
| **500** | Internal Server Error | Unhandled server-side exception | Database connection crashed mid-request |

### Quick Memory Aid

```
2xx = ✅ Everything is fine
4xx = ❌ Client's fault (fix your request)
5xx = 💥 Server's fault (we messed up)
```

---

## Cheat Sheet

> Print this. Review it 5 minutes before the lecture.

```
┌─────────────────────────────────────────────────────────────┐
│                    FastAPI CHEAT SHEET                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PATH PARAMS     /packages/{id}     → identify ONE thing    │
│  QUERY PARAMS    ?status=pending    → filter/sort/paginate  │
│  REQUEST BODY    { JSON payload }   → send data (POST/PUT)  │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PYDANTIC                                                   │
│  class PackageCreate(BaseModel):                            │
│      name: str = Field(min_length=2)                        │
│      weight: float = Field(gt=0)                            │
│      desc: str | None = None        # optional              │
│                                                             │
│  Create schema  → all required fields                       │
│  Update schema  → all fields optional (| None)              │
│  Response schema → includes server-generated fields         │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CRUD                                                       │
│  POST   /packages          → Create   (201)                 │
│  GET    /packages          → Read all (200)                 │
│  GET    /packages/{id}     → Read one (200)                 │
│  PUT    /packages/{id}     → Full update (200)              │
│  PATCH  /packages/{id}     → Partial update (200)           │
│  DELETE /packages/{id}     → Delete (204)                   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CUSTOM RESPONSE                                            │
│  response_model=Schema     → filters output, documents API  │
│  status_code=201           → set in decorator               │
│  JSONResponse(content, headers)  → full control             │
│  Response(status_code=204) → empty body                     │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ERROR HANDLING                                             │
│  raise HTTPException(404, detail="Not found")  → specific  │
│  @app.exception_handler(MyError)               → global    │
│  RequestValidationError handler                → custom 422│
│                                                             │
│  STATUS CODES:  200 OK | 201 Created | 204 No Content      │
│  400 Bad Request | 401 Unauthorized | 403 Forbidden         │
│  404 Not Found | 422 Validation Error | 500 Server Error    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Interactive Questions for the Lecture

### Q1 — Path or Query?

> "A customer wants to get all packages sent to **Cairo** that are currently **in transit**, sorted by **newest first**, showing only the **first 10**. What's the endpoint?"

**Expected answer:**
```
GET /packages?city=cairo&status=in_transit&sort_by=created_at&order=desc&limit=10
```
All filters are query params because we're filtering a **collection**, not identifying one resource.

---

### Q2 — What's Wrong With This Code?

Show this code and ask them to find ALL the problems:

```python
@app.get("/couriers/{courier_id}")
def get_courier(courier_id):
    courier = find_courier(courier_id)
    if not courier:
        return {"error": "Not found"}
    return courier

@app.get("/couriers/available")
def get_available(): ...
```

**Expected answers:**
1. `courier_id` has no type hint — no validation, treated as string.
2. Error returns `200 OK` with `{"error": "Not found"}` instead of raising `HTTPException(404)`.
3. `/couriers/available` is declared AFTER `/couriers/{courier_id}` — FastAPI will try to parse `"available"` as the courier_id.

---

### Q3 — PUT or PATCH?

> "A courier wants to update only their phone number. Should we use PUT or PATCH? What happens if we use the wrong one?"

**Expected answer:**
- **PATCH** — only update the phone number.
- If we use **PUT** with only the phone number, all other fields (name, vehicle, etc.) would be overwritten with null/defaults — we'd lose data.
- PATCH with `model_dump(exclude_unset=True)` ensures only the sent field is updated.

---

### Q4 — Design Challenge

> "A customer wants to cancel their order. Design the endpoint: Method? Path? What status codes could it return?"

**Expected answer:**
```
PATCH /orders/{order_id}  with body: {"status": "cancelled"}
— OR —
POST /orders/{order_id}/cancel  (action endpoint)

Status codes:
  200 → Successfully cancelled
  404 → Order not found
  400 → Order already delivered (can't cancel)
  401 → Not logged in
  403 → Not the owner of this order
```
