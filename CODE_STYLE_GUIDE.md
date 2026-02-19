# Code Style Guide

This document defines the coding standards for the GSIS Cloud Monorepo. All contributors must follow these conventions to maintain consistency, readability, and quality across the codebase.

**Languages covered:**

- [Python (Google Style Guide)](#python-style-guide)
- [JavaScript / TypeScript (Neostandard + Nuxt ESLint)](#javascript--typescript-style-guide)
- [Go (Effective Go + Google Style + Fiber)](#go-style-guide)

---

## Python Style Guide

Based on [Google's Python Style Guide](https://google.github.io/styleguide/pyguide.html), enforced via **Ruff**, **Black**, and **mypy**.

### Tooling

| Tool    | Purpose              | Config Location         |
| ------- | -------------------- | ----------------------- |
| Black   | Code formatter       | `pyproject.toml`        |
| Ruff    | Linter               | `pyproject.toml`        |
| mypy    | Static type checker  | `pyproject.toml`        |
| pytest  | Test runner          | `pyproject.toml`        |

### Formatting

- **Line length:** 100 characters max.
- **Indentation:** 4 spaces. No tabs.
- **Target version:** Python 3.11+.
- **Formatter:** Black with `line-length = 100` and `target-version = ['py311']`.

```python
# ✅ Good: Within 100 characters
user = get_user_by_email(email="admin@example.com")

# ❌ Bad: Exceeds line length — break it up
result = some_very_long_function_name(first_argument="value", second_argument="value", third_argument="another_value", fourth="more")

# ✅ Good: Multi-line break
result = some_very_long_function_name(
    first_argument="value",
    second_argument="value",
    third_argument="another_value",
    fourth="more",
)
```

### Imports

Follow Google's import ordering and grouping rules:

1. Standard library imports
2. Related third-party imports
3. Local application/library imports

Each group separated by a blank line. Use absolute imports.

```python
# ✅ Good
import os
import sys

from fastapi import FastAPI, APIRouter
from pydantic import BaseModel

from models.user import User
from utils.logging import log_info
```

```python
# ❌ Bad: Mixed groups, wildcard imports
from os import *
from models.user import User
import sys
from fastapi import FastAPI
```

- **No wildcard imports** (`from module import *`).
- **One import per line** for `from` imports when importing multiple names — unless they logically group together and fit within the line limit.

```python
# ✅ Acceptable: Logical grouping that fits
from fastapi import APIRouter, Path, Query, Body

# ✅ Also acceptable: Multi-line for many imports
from fastapi import (
    APIRouter,
    Path,
    Query,
    Body,
    Depends,
    HTTPException,
)
```

### Naming Conventions

| Type                  | Convention              | Example                       |
| --------------------- | ----------------------- | ----------------------------- |
| Modules               | `snake_case`            | `user_routes.py`              |
| Packages              | `snake_case`            | `utils/`                      |
| Classes               | `PascalCase`            | `UserCreateResponse`          |
| Functions / Methods   | `snake_case`            | `get_user_by_id()`            |
| Variables             | `snake_case`            | `user_email`                  |
| Constants             | `UPPER_SNAKE_CASE`      | `MAX_RETRIES`                 |
| Enums                 | `PascalCase` (class), `UPPER_SNAKE_CASE` (members) | `RoleEnum.ADMIN` |
| Type Variables        | `PascalCase`            | `T`, `ResponseT`             |
| Private               | Leading underscore      | `_internal_helper()`          |

```python
# ✅ Good
class UserCreateRequestBody(BaseModel):
    first_name: str
    last_name: str

MAX_RETRY_COUNT = 3

def get_settings():
    return Settings()
```

```python
# ❌ Bad
class usercreaterequestbody(BaseModel):  # Not PascalCase
    FirstName: str  # Not snake_case
    
maxRetryCount = 3  # Not UPPER_SNAKE_CASE for constant
```

### Type Annotations

All functions **must** have type annotations for parameters and return values. Use modern Python 3.11+ syntax.

```python
# ✅ Good: Full type annotations using modern syntax
def get_user_by_id(user_id: str) -> UserWithId | None:
    ...

def process_roles(roles: list[RoleEnum]) -> dict[str, bool]:
    ...

# Use Annotated for field metadata (Pydantic / FastAPI)
from typing import Annotated
from pydantic import Field

class UserId(BaseModel):
    uid: Annotated[str, Field(description="The unique id of the user")]
```

```python
# ❌ Bad: Missing annotations
def get_user_by_id(user_id):
    ...
```

- Use `X | None` instead of `Optional[X]`.
- Use `list[str]` instead of `List[str]`.
- Use `dict[str, int]` instead of `Dict[str, int]`.
- Use `Annotated` with `Field` for Pydantic models.

### Docstrings

Use Google-style docstrings for modules, classes, and public functions.

```python
def calculate_discount(order_total: float, discount_rate: float) -> float:
    """Calculate the discount amount for an order.

    Args:
        order_total: The total value of the order in dollars.
        discount_rate: The discount rate as a decimal (e.g., 0.15 for 15%).

    Returns:
        The calculated discount amount.

    Raises:
        ValueError: If order_total is negative or discount_rate is out of range.
    """
    if order_total < 0:
        raise ValueError(f"Invalid order_total: {order_total}")
    return order_total * discount_rate
```

- **Module docstrings:** Required at the top of each module with a brief description of purpose.
- **Class docstrings:** Describe the class purpose and any important attributes.
- **Function docstrings:** Required for all public functions. Include `Args`, `Returns`, and `Raises` sections when applicable.
- **One-liner docstrings** are acceptable for trivially simple functions.

### Classes and Pydantic Models

- Prefer **composition and multiple inheritance** for Pydantic models to build up from reusable parts.
- Keep models focused — one responsibility per model.

```python
# ✅ Good: Composable Pydantic models
class UserEmail(BaseModel):
    email: Annotated[str, Field(description="The email of the user")]

class UserFirstLastName(BaseModel):
    first_name: Annotated[str, Field(description="The first name of the user")]
    last_name: Annotated[str, Field(description="The last name of the user")]

class User(UserFirstLastName, UserEmail):
    pass

class UserWithRoles(UserRoles, UserFirstLastName, UserEmail):
    pass
```

### Error Handling

- Use specific exception types. Never use bare `except:`.
- Re-raise or wrap exceptions with meaningful messages.
- Use `HTTPException` for API error responses.
- Log errors with full context and stack traces.

```python
# ✅ Good
try:
    user = db.get(user_id)
except KeyError:
    raise HTTPException(status_code=404, detail=f"User {user_id} not found")
except Exception as e:
    log_error(
        header="User Fetch",
        action="Fetching user failed",
        error=str(e),
        stack=traceback.format_exc(),
    )
    raise HTTPException(status_code=500, detail="Internal server error")
```

```python
# ❌ Bad: Bare except, silent failure
try:
    user = db.get(user_id)
except:
    pass
```

### FastAPI Conventions

- **Routers:** Each resource gets its own router in `routes/`. Use `prefix` and `tags`.
- **Models:** Request/response models live in `models/`. Use descriptive class names ending with `RequestBody`, `Response`, etc.
- **Status codes:** Explicitly set `status_code` on create endpoints (201).
- **Dependencies:** Use `Depends()` for shared logic (auth, config).

```python
router = APIRouter(
    prefix="/users",
    tags=["user"],
)

@router.post("/", status_code=201)
async def user_create(
    body: Annotated[UserCreateRequestBody, Body(description="The request body for creating a user")],
) -> UserCreateResponse:
    ...
```

### Testing

- Test files: `test_*.py` or `*_test.py` in the `tests/` directory.
- Use **pytest** with `pytest-cov`.
- Descriptive test names: `test_should_return_404_when_user_not_found`.
- Follow **Arrange-Act-Assert** pattern.

```python
def test_should_create_user_with_valid_email():
    # Arrange
    body = UserCreateRequestBody(
        email="test@example.com",
        first_name="Jane",
        last_name="Doe",
        roles=[RoleEnum.USER],
    )

    # Act
    response = client.post("/users/", json=body.model_dump())

    # Assert
    assert response.status_code == 201
    assert "uid" in response.json()
```

### Things to Avoid

- Mutable default arguments (`def f(items=[]):`).
- Global mutable state (except module-level configuration via `lru_cache`).
- `print()` for logging — use the project's structured logging utilities.
- Magic numbers and hard-coded strings — use constants or config.
- Deeply nested code (max 3-4 levels).
- Commented-out code (use version control instead).

---

## JavaScript / TypeScript Style Guide

Based on **[Neostandard](https://github.com/neostandard/neostandard)** with project-specific overrides, enforced via ESLint 9 flat config. The web portal uses **Nuxt 4**, **Vue 3**, and **TypeScript 5**.

### Tooling

| Tool              | Purpose              | Config Location                    |
| ----------------- | -------------------- | ---------------------------------- |
| ESLint 9          | Linter               | `web-portal/eslint.config.mjs`     |
| Neostandard       | Base ruleset         | via `neostandard` package          |
| @stylistic/eslint | Stylistic rules      | via `@stylistic/eslint-plugin`     |
| TypeScript 5      | Type checking        | `web-portal/tsconfig.json`         |

### ESLint Configuration

The project uses Nuxt's ESLint integration combined with Neostandard in `noStyle` mode, with custom stylistic overrides:

```javascript
import withNuxt from './.nuxt/eslint.config.mjs'
import neostandard from 'neostandard'
import stylistic from '@stylistic/eslint-plugin'

export default withNuxt(
    ...neostandard({ noStyle: true }),
    {
        plugins: {
            '@stylistic': stylistic,
        },
        rules: {
            '@stylistic/indent': ['error', 4],
            '@stylistic/comma-dangle': ['error', 'always-multiline'],
        },
    },
)
```

### Formatting

- **Indentation:** 4 spaces. No tabs.
- **Trailing commas:** **Always** on multiline constructs (arrays, objects, parameters, imports).
- **Semicolons:** None (Neostandard default — no semicolons).
- **Quotes:** Single quotes for strings (Neostandard default).
- **Line length:** Keep lines readable; aim for under 100 characters where practical.

```javascript
// ✅ Good
const user = {
    firstName: 'Jane',
    lastName: 'Doe',
    roles: [
        'admin',
        'user',
    ],
}

// ❌ Bad: Wrong indentation, missing trailing commas, double quotes, semicolons
const user = {
  firstName: "Jane",
  lastName: "Doe",
  roles: [
    "admin",
    "user"
  ]
};
```

### Naming Conventions

| Type                  | Convention            | Example                       |
| --------------------- | --------------------- | ----------------------------- |
| Files (components)    | `PascalCase.vue`      | `UserProfile.vue`             |
| Files (composables)   | `camelCase.ts`        | `useAuth.ts`                  |
| Files (utils)         | `camelCase.ts`        | `formatDate.ts`               |
| Variables             | `camelCase`           | `userName`                    |
| Constants             | `UPPER_SNAKE_CASE`    | `MAX_RETRIES`                 |
| Functions             | `camelCase`           | `getUserById()`               |
| Composables           | `use` prefix          | `useUserStore()`              |
| Components            | `PascalCase`          | `<UserProfile />`             |
| Interfaces / Types    | `PascalCase`          | `UserResponse`                |
| Enums                 | `PascalCase`          | `UserRole`                    |
| Boolean variables     | `is`/`has`/`should`   | `isActive`, `hasPermission`   |

```typescript
// ✅ Good
const isAuthenticated = ref(false)
const MAX_LOGIN_ATTEMPTS = 5

function getUserById(id: string): Promise<User> {
    ...
}

// ❌ Bad
const authenticated = ref(false)  // Missing boolean prefix
const max_login_attempts = 5      // Not UPPER_SNAKE_CASE
function GetUserById(id) { ... }  // Not camelCase, missing types
```

### TypeScript

- **Strict mode:** Enabled via Nuxt's TypeScript configuration.
- **Type all function parameters and return types.**
- **Prefer `interface` for object shapes** and `type` for unions/intersections.
- **Avoid `any`** — use `unknown` if the type is truly unknown.
- **Use `as const`** for literal type narrowing.

```typescript
// ✅ Good
interface User {
    id: string
    email: string
    roles: UserRole[]
}

type ApiResponse<T> = {
    data: T
    success: boolean
}

function getUser(id: string): Promise<User | null> {
    ...
}

// ❌ Bad
function getUser(id: any): any {
    ...
}
```

### Vue / Nuxt Conventions

#### Component Structure

Use the `<script setup>` syntax with TypeScript. Order template sections consistently:

```vue
<script setup lang="ts">
// 1. Imports
import type { User } from '~/types/user'

// 2. Props & Emits
const props = defineProps<{
    userId: string
}>()

const emit = defineEmits<{
    (e: 'update', user: User): void
}>()

// 3. Composables / State
const { data: user } = await useFetch(`/api/users/${props.userId}`)

// 4. Computed / Watchers
const fullName = computed(() => `${user.value?.firstName} ${user.value?.lastName}`)

// 5. Methods
function handleUpdate() {
    emit('update', user.value!)
}
</script>

<template>
    <div>
        <h1>{{ fullName }}</h1>
        <button @click="handleUpdate">
            Update
        </button>
    </div>
</template>
```

#### Template Rules

- **Indentation:** 4 spaces inside `<template>`, `<script>`, and `<style>`.
- **Self-closing components:** `<UserProfile />` (not `<UserProfile></UserProfile>`).
- **Attribute ordering:** Follow Vue's recommended order — `v-if`, `v-for`, `v-bind`, event handlers, then static attributes.
- **Multi-line attributes:** When a component has 3+ attributes, put each on its own line.

```vue
<!-- ✅ Good: Multi-line attributes -->
<UserCard
    :user="currentUser"
    :is-editable="hasPermission"
    @update="handleUpdate"
/>

<!-- ❌ Bad: Crammed on one line -->
<UserCard :user="currentUser" :is-editable="hasPermission" @update="handleUpdate" />
```

#### Nuxt Auto-Imports

Nuxt auto-imports Vue APIs, composables, and utilities. Do **not** manually import auto-imported items:

```typescript
// ✅ Good: Auto-imported by Nuxt
const route = useRoute()
const count = ref(0)
const doubled = computed(() => count.value * 2)

// ❌ Bad: Unnecessary manual imports
import { ref, computed } from 'vue'
import { useRoute } from 'vue-router'
```

### Variables and Declarations

- Use `const` by default. Use `let` only when reassignment is needed.
- Never use `var`.
- Declare one variable per statement.

```javascript
// ✅ Good
const userName = 'Jane'
let retryCount = 0

// ❌ Bad
var userName = 'Jane'
let a = 1, b = 2, c = 3
```

### Functions

- Prefer **arrow functions** for anonymous callbacks and inline expressions.
- Use **named function declarations** for top-level and exported functions.
- Keep functions small and focused.

```javascript
// ✅ Good: Named declaration for top-level
function calculateTotal(items) {
    return items.reduce((sum, item) => sum + item.price, 0)
}

// ✅ Good: Arrow for callbacks
const activeUsers = users.filter((user) => user.isActive)

// ❌ Bad: Arrow for top-level (harder to debug in stack traces)
const calculateTotal = (items) => {
    return items.reduce((sum, item) => sum + item.price, 0)
}
```

### Error Handling

- Always handle errors in async operations.
- Use try/catch with meaningful error context.
- Never silently swallow errors.

```typescript
// ✅ Good
try {
    const user = await fetchUser(userId)
    return user
} catch (error) {
    console.error(`Failed to fetch user ${userId}:`, error)
    throw createError({
        statusCode: 500,
        statusMessage: 'Failed to fetch user',
    })
}

// ❌ Bad
try {
    const user = await fetchUser(userId)
} catch (e) {
    // silently ignored
}
```

### Module Exports

- Use **named exports** over default exports (except for Nuxt pages/layouts which require defaults).
- Group related exports in index files.

```typescript
// ✅ Good: Named exports
export function formatDate(date: Date): string { ... }
export function formatCurrency(amount: number): string { ... }

// ❌ Bad: Default export for utility
export default function formatDate(date: Date): string { ... }
```

### Testing

- Use **Vitest** for unit and component tests.
- Test files: `*.test.ts` or `*.spec.ts`.
- Descriptive test names using sentence-style.
- Follow **Arrange-Act-Assert** pattern.

```typescript
import { describe, it, expect } from 'vitest'

describe('formatCurrency', () => {
    it('should format positive amounts with two decimal places', () => {
        // Arrange
        const amount = 1234.5

        // Act
        const result = formatCurrency(amount)

        // Assert
        expect(result).toBe('$1,234.50')
    }),

    it('should handle zero correctly', () => {
        expect(formatCurrency(0)).toBe('$0.00')
    }),
})
```

### Things to Avoid

- `var` declarations.
- `any` type in TypeScript.
- Semicolons (Neostandard enforces no-semi).
- Double quotes for strings (use single quotes).
- `console.log` in production code (use proper logging or remove).
- Deeply nested callbacks (use async/await).
- Unused variables and imports (ESLint will flag these).
- `==` loose equality (use `===` strict equality).
- Inline styles in Vue templates (use Tailwind CSS classes or scoped styles).

---

## Go Style Guide

Based on [Effective Go](https://go.dev/doc/effective_go), [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments), and [Google's Go Style Guide](https://google.github.io/styleguide/go/). The backend API server runs on **Go 1.22+** with the **[Fiber](https://gofiber.io/)** framework (built on [Fasthttp](https://github.com/valyala/fasthttp)), deployed to **Cloud Run**.

### Tooling

| Tool            | Purpose                | Config Location     |
| --------------- | ---------------------- | ------------------- |
| `go fmt`        | Code formatter         | Built-in            |
| `go vet`        | Suspicious constructs  | Built-in            |
| `golangci-lint` | Comprehensive linter   | `.golangci.yml`     |
| `go test`       | Test runner            | Built-in            |
| `go mod`        | Dependency management  | `go.mod` / `go.sum` |

### Formatting

- **Formatter:** `gofmt` (canonical). Run `goimports` for import ordering.
- **Indentation:** Tabs (Go standard).
- **Line length:** No hard limit, but keep lines readable (~100 characters as a guideline).
- All code **must** be formatted with `gofmt` before committing.

```go
// ✅ Good: gofmt-formatted, idiomatic
func GetUserByEmail(ctx context.Context, email string) (*User, error) {
	user, err := db.QueryUser(ctx, email)
	if err != nil {
		return nil, fmt.Errorf("get user by email %q: %w", email, err)
	}
	return user, nil
}

// ❌ Bad: Inconsistent formatting, no error wrapping
func GetUserByEmail(ctx context.Context, email string) (*User,error) {
  user,err := db.QueryUser(ctx,email)
  if err!=nil { return nil,err }
  return user,nil
}
```

### Naming Conventions

Follow Go naming idioms — names are a core part of Go's readability contract.

**Packages:**

- Short, lowercase, single-word names (e.g., `user`, `config`, `handler`).
- No underscores, no `mixedCaps`.
- Package name should describe what it provides, not what it contains.
- Avoid generic names like `util`, `common`, `helpers`.

```go
// ✅ Good
package user
package config

// ❌ Bad
package userService
package common_utils
```

**Variables and Functions:**

- `mixedCaps` / `MixedCaps` (exported) — no underscores.
- Short names for short scopes (`i`, `n`, `err`).
- Descriptive names for wider scopes (`userCount`, `maxRetries`).
- Acronyms are all caps: `HTTPClient`, `userID`, `xmlParser`.

```go
// ✅ Good
var userCount int
func ParseHTTPResponse(resp *http.Response) error { ... }

// ❌ Bad
var user_count int
func ParseHttpResponse(resp *http.Response) error { ... }
```

**Interfaces:**

- Single-method interfaces use `-er` suffix: `Reader`, `Writer`, `Handler`.
- Keep interfaces small (1–3 methods).
- Define interfaces where they are **used**, not where they are implemented.
- Accept interfaces, return concrete types.

```go
// ✅ Good: Small, focused interface defined at the call site
type UserFinder interface {
	FindByEmail(ctx context.Context, email string) (*User, error)
}

func NewHandler(finder UserFinder) *Handler {
	return &Handler{finder: finder}
}
```

**Constants:**

- `MixedCaps` (not `SCREAMING_SNAKE_CASE`).
- Group related constants with `iota` where applicable.

```go
// ✅ Good
const MaxRetries = 3
const defaultTimeout = 30 * time.Second

// ❌ Bad
const MAX_RETRIES = 3
const DEFAULT_TIMEOUT = 30
```

### Imports

Use `goimports` to automatically format and group imports:

1. Standard library
2. Third-party packages
3. Local project packages

Each group separated by a blank line.

```go
import (
	"context"
	"fmt"

	"github.com/gofiber/fiber/v2"
	"github.com/rs/zerolog"

	"github.com/tjmonsi/personal-website/internal/handler"
)
```

### Error Handling

Go's explicit error handling is a feature, not a burden. Follow these patterns:

- **Always check errors** — never discard with `_` unless explicitly justified.
- **Wrap errors with context** using `fmt.Errorf` and the `%w` verb.
- **Return errors early** — prefer guard clauses over deeply nested conditions.
- **Don't log and return** — choose one or the other.
- **Error messages are lowercase**, no trailing punctuation.

```go
// ✅ Good: Early return, wrapped error with context
func (s *Service) CreateUser(ctx context.Context, req CreateUserReq) (*User, error) {
	if req.Email == "" {
		return nil, errors.New("email is required")
	}

	user, err := s.repo.Insert(ctx, req)
	if err != nil {
		return nil, fmt.Errorf("create user: %w", err)
	}

	return user, nil
}

// ❌ Bad: Nested conditions, error discarded
func (s *Service) CreateUser(ctx context.Context, req CreateUserReq) (*User, error) {
	if req.Email != "" {
		user, _ := s.repo.Insert(ctx, req)
		return user, nil
	}
	return nil, errors.New("Failed")
}
```

**Error checking patterns:**

- Use `errors.Is` for sentinel errors (e.g., `errors.Is(err, sql.ErrNoRows)`).
- Use `errors.As` for typed errors (e.g., extracting a `*ValidationError`).
- Create custom error types for domain-specific errors.

### Structs and Methods

- Use pointer receivers for methods that modify state or for large structs.
- Use value receivers for small, immutable structs.
- Be consistent within a type's method set.
- Use struct tags for JSON, database mappings, and validation.

```go
// ✅ Good: Pointer receiver for mutation, struct tags for JSON
type User struct {
	ID    string `json:"id" firestore:"id"`
	Email string `json:"email" firestore:"email"`
	Name  string `json:"name" firestore:"name"`
}

func (u *User) SetEmail(email string) {
	u.Email = email
}
```

### Concurrency

- Always know **how** and **when** a goroutine will exit.
- Use `sync.WaitGroup` to coordinate goroutine lifetimes.
- Use `context.Context` for cancellation and timeouts.
- Prefer channels for communication; use `sync.Mutex` for protecting shared state.
- Never modify a `map` concurrently without synchronization.

```go
// ✅ Good: Context-aware, clean goroutine lifecycle
func (s *Service) ProcessBatch(ctx context.Context, items []Item) error {
	g, ctx := errgroup.WithContext(ctx)
	for _, item := range items {
		g.Go(func() error {
			return s.processItem(ctx, item)
		})
	}
	return g.Wait()
}
```

### HTTP Handlers and JSON APIs (Fiber)

- Use **[Fiber v2](https://gofiber.io/)** for routing, middleware, and request handling.
- Handlers return `error` — use Fiber's error handling for consistent error responses.
- Access path params via `c.Params("id")`, query params via `c.Query("key")`.
- Use `c.UserContext()` to pass Go's `context.Context` to service/repository layers.
- Use `c.JSON()` for JSON responses and `c.Status()` for status codes.
- Group related routes with `app.Group()`.
- Use Fiber's built-in middleware (CORS, Recover, Logger) where applicable.

```go
// ✅ Good: Clean Fiber handler with validation and proper error response
func (h *Handler) GetUser(c *fiber.Ctx) error {
	id := c.Params("id")
	if id == "" {
		return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
			"error": fiber.Map{"code": "MISSING_ID", "message": "missing user id"},
		})
	}

	user, err := h.service.FindByID(c.UserContext(), id)
	if err != nil {
		return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
			"error": fiber.Map{"code": "NOT_FOUND", "message": "user not found"},
		})
	}

	return c.JSON(user)
}
```

**Route registration:**

```go
// ✅ Good: Grouped routes with middleware
func SetupRoutes(app *fiber.App, h *Handler) {
	api := app.Group("/", middleware.Breadcrumb(), middleware.RequestID())

	api.Get("/technical", h.ListTechnical)
	api.Get("/technical/:slug", h.GetTechnical)
	api.Get("/blog", h.ListBlog)
	api.Get("/blog/:slug", h.GetBlog)
	api.Post("/t", h.Track)
}
```

### Project Structure

Follow Go project layout conventions:

```
cmd/
  api/              # main package — initializes Fiber app
    main.go
internal/
  handler/          # Fiber handlers (func(*fiber.Ctx) error)
  service/          # Business logic (accepts context.Context)
  repository/       # Data access (Firestore)
  model/            # Domain types
  middleware/       # Fiber middleware
  router/           # Route registration and grouping
go.mod
go.sum
```

- `cmd/` for `main` packages (entry points).
- `internal/` for packages private to this project.
- Group by domain/layer, not by file type.

### Testing

- Use **table-driven tests** with subtests (`t.Run`).
- Test files live alongside source: `user.go` → `user_test.go`.
- Name tests descriptively: `TestCreateUser_EmptyEmail_ReturnsError`.
- Mark helper functions with `t.Helper()`.
- Follow the **Arrange-Act-Assert** pattern.

```go
func TestCreateUser(t *testing.T) {
	tests := []struct {
		name    string
		email   string
		wantErr bool
	}{
		{name: "valid email", email: "user@example.com", wantErr: false},
		{name: "empty email", email: "", wantErr: true},
		{name: "invalid format", email: "not-an-email", wantErr: true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			svc := NewService(mockRepo)
			_, err := svc.CreateUser(context.Background(), CreateUserReq{Email: tt.email})

			if (err != nil) != tt.wantErr {
				t.Errorf("CreateUser() error = %v, wantErr %v", err, tt.wantErr)
			}
		})
	}
}
```

### Things to Avoid

- Discarding errors with `_` without justification.
- Global mutable state — pass dependencies explicitly.
- `init()` functions — use explicit initialization.
- `panic` for recoverable errors (use `error` return values).
- `interface{}` / `any` for everything — use specific types or generics.
- Modifying maps concurrently without a mutex.
- Creating goroutines without a known exit path.
- Magic numbers — use named constants.
- `SCREAMING_SNAKE_CASE` constants (use `MixedCaps`).
- Deep nesting — prefer early returns and guard clauses.

---

## Shared Conventions

These conventions apply across **all** codebases (Python, JavaScript/TypeScript, and Go).

### Git Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`.

```
feat(web-portal): add user profile page
fix(agent-fulfillment): handle missing email in user creation
docs: update CODE_STYLE_GUIDE with Vue conventions
```

### Comments

Follow the [self-explanatory code principle](https://google.github.io/styleguide/pyguide.html#385-block-and-inline-comments):

- Write code that speaks for itself.
- Comment **why**, not **what**.
- Use `TODO:`, `FIXME:`, `HACK:` annotations with context.
- No commented-out code — use version control.

### File Organization

- Keep files focused — one primary purpose per file.
- Use consistent directory structures across services.
- Group by feature/domain, not by file type, when possible.

### Security

- Never commit secrets, API keys, or credentials.
- Use environment variables or secret managers.
- Validate all user inputs.
- Use parameterized queries for database operations.

---

## Enforcement

| Check          | Python                         | JS / TS                              | Go                                   |
| -------------- | ------------------------------ | ------------------------------------ | ------------------------------------ |
| Formatting     | `black --check .`              | `npx eslint .`                       | `gofmt -l .`                         |
| Linting        | `ruff check .`                 | `npx eslint .`                       | `golangci-lint run`                  |
| Type checking  | `mypy .`                       | `npx nuxi typecheck`                 | `go vet ./...`                       |
| Tests          | `pytest`                       | `npx vitest`                         | `go test ./...`                      |

Run all checks before submitting a pull request. CI pipelines will enforce these automatically.

---

## References

- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- [Neostandard](https://github.com/neostandard/neostandard)
- [ESLint Stylistic](https://eslint.style/)
- [Nuxt ESLint Module](https://eslint.nuxt.com/)
- [Vue Style Guide](https://vuejs.org/style-guide/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Google Go Style Guide](https://google.github.io/styleguide/go/)
- [Standard Go Project Layout](https://github.com/golang-standards/project-layout)
- [Go Fiber](https://gofiber.io/)
- [Conventional Commits](https://www.conventionalcommits.org/)
