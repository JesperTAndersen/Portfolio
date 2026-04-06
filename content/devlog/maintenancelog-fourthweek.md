---
title: "Maintenance Log - Fourth Week: Building a Production-Style REST API"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "REST", "Javalin", "Logging", "Validation"]
series: ["Maintenance Log"]
date: 2026-03-09
draft: false
---

# Devlog Week 4: Building a Production-Style REST API

Welcome back! This week was all about taking the solid persistence layer and API integration work from previous weeks and exposing it all through a proper REST API. The goal: build a complete, well-architected HTTP interface using Javalin 7.0.1 that follows REST principles and industry best practices.

Spoiler: it also turned into a significant refactor.

**TL;DR (what changed this week):**
- REST API layer added on top of the persistence/integration work
- Routes reorganized + versioned under `/api/v1` (Javalin 7-style config)
- DTO strategy tightened (request vs response where shapes differ)
- Validation split into controller (input) vs service (business rules)
- Centralized exception handling + proper logging (Logback)
- `Main` cleaned up via `AppConfig` + `DependencyContainer`

---

## Javalin 7.0.1: New Version, New Rules

First hurdle: Javalin 7.0.1 introduced some breaking changes from earlier versions that I had to work through.

**The big one:** In Javalin 7, routes need to be registered inside the `config.routes` block during app creation. Adding routes after calling `.start()` no longer works.
```java
Javalin app = Javalin.create(config -> {
    config.routes.apiBuilder(routes.getRoutes());  // ← Must be here
}).start(7070);

// app.get("/test", handler);  ← This no longer works!
```

This actually forced better architecture — all routing configuration happens upfront in one place, which makes the setup more explicit and easier to reason about.

**API Versioning from Day One**

I decided to version the API right from the start with an `/api/v1/` prefix on all endpoints:
```
/api/v1/users
/api/v1/assets
/api/v1/logs
```

**Rationale:** Even if this is a school project, building the habit of versioning APIs is important. If I ever need to make breaking changes later (different response structure, new validation rules, etc.), I can introduce `/api/v2/` without breaking existing clients.

---

## Route Organization: Keeping Things Modular

With three entities (User, Asset, MaintenanceLog), I needed a clean way to organize routes without having one giant file with 15+ endpoint definitions.

**The pattern I landed on:**
```
controllers/routes/
├── Routes.java                  # Aggregator
├── UserRoutes.java              # /api/v1/users
├── AssetRoutes.java             # /api/v1/assets + /{id}/logs
└── MaintenanceLogRoutes.java    # /api/v1/logs
```

Each route class returns an `EndpointGroup`:
```java
public class UserRoutes {
    public EndpointGroup getRoutes() {
        return () -> {
            path("api/v1/users", () -> {
                get(userController::getAll);
                get("/{id}", userController::get);
                post(userController::create);
                put("/{id}", userController::update);
                delete("/{id}", userController::deactivate);
                patch("/{id}", userController::activate);
            });
        };
    }
}
```

Then `Routes.java` aggregates them all:
```java
public EndpointGroup getRoutes() {
    return () -> {
        userRoutes.getRoutes().addEndpoints();
        assetRoutes.getRoutes().addEndpoints();
        logRoutes.getRoutes().addEndpoints();
    };
}
```

**Why this matters:** Each entity's routes are self-contained. If I need to change how assets work, I only touch `AssetRoutes.java`. No risk of accidentally breaking user endpoints.

---

## DTO Strategy: Security and Separation of Concerns

One of the more interesting design decisions this week: **when do you use the same DTO for requests and responses, and when do you split them?**

**Rule of thumb:** I split DTOs whenever the *shape* of the request and response should differ — either because of **sensitive fields** (only allowed inbound) or **server-owned fields** (IDs, derived/calculated fields, etc.).

**User API: Split DTOs**

For users, I needed separate request and response DTOs because of passwords:
```java
// Request DTO (used for POST /users)
public class CreateUserRequest {
    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private UserRole role;
    private String password;  // ← Only in requests!
}

// Response DTO (used for all GET endpoints)
public class UserDTO {
    private Integer id;
    private String firstName;
    private String lastName;
    private String phone;
    private String email;
    private UserRole role;
    private boolean active;
    // NO password field — never exposed
}
```

**The password flow:**
1. Client sends `CreateUserRequest` with plain text password
2. Service layer hashes it immediately with BCrypt
3. `UserDTO` returned in response (no password)
4. Password never appears in any response

**Assets/Logs: Shared read DTOs + create-specific request DTOs**

For assets and logs, there's no sensitive data, so I can generally reuse the same DTO for *read* endpoints (and for any operations where the request/response fields match). That keeps things simple and avoids needless duplication.

For *create* endpoints, I still prefer a dedicated request DTO (like `CreateLogRequest`) because the client shouldn't be sending fields like IDs, and sometimes the API returns extra data that isn't part of the input.

---

## Nested Resources: When REST Gets Interesting

A key architectural decision this week was how to structure log endpoints.

**The use case:** In the UI, you select an asset and then view its logs. The primary access pattern is **asset-centric**, not log-centric.

**Two options:**

**Option A: Only nested routes**
```
GET /api/v1/assets/{id}/logs
```

**Option B: Both nested AND standalone routes**
```
# Asset-scoped
GET /api/v1/assets/{id}/logs

# Cross-asset queries
GET /api/v1/logs?userId=42
GET /api/v1/logs?status=COMPLETED
```

I went with **Option B** — both structures.

**Rationale:**
- Most of the time, you're looking at logs for a specific asset (nested routes)
- But sometimes you need cross-cutting queries: "show me all logs by this technician" or "show me all incomplete maintenance tasks"
- Having both gives maximum flexibility without forcing awkward query parameters

**Implementation:**
```java
// AssetRoutes.java
path("api/v1/assets", () -> {
    // ... asset endpoints ...
    path("/{id}/logs", () -> {
        get(logController::getLogsByAsset);
        post(logController::createLogForAsset);
    });
});

// MaintenanceLogRoutes.java
path("api/v1/logs", () -> {
    get(logController::getAll);
    get("/{id}", logController::get);
    get("/user/{userId}", logController::getByUser);
});
```

**Design constraint:** Logs can ONLY be created via `/assets/{id}/logs`. This enforces that every log is attached to an asset from the start — you can't accidentally create orphaned logs.

---

## Validation: Two Layers, Two Responsibilities

I split validation into two distinct layers with different responsibilities:

**Controller Layer: Input Validation**

This is where I check that the request is structurally valid:
```java
CreateUserRequest request = ctx.bodyValidator(CreateUserRequest.class)
    .check(dto -> dto.getFirstName() != null, "First name is required")
    .check(dto -> dto.getEmail() != null, "Email is required")
    .check(dto -> dto.getEmail().matches("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"), 
           "Invalid email format")
    .get();
```

**Service Layer: Business Validation**

This is where I check business rules:
```java
public UserDTO create(CreateUserRequest request) {
    // Business rule: email must be unique
    if (userDaoExpanded.getByEmail(request.getEmail()) != null) {
        throw new ApiException(409, "Email already exists");
    }
    // ...
}
```

**Why separate them?**
- Input validation catches malformed requests before they hit business logic
- Business validation enforces domain rules (uniqueness, referential integrity, etc.)
- Service layer can be tested independently of HTTP concerns
- Clear separation of concerns

---

## The Lazy Loading Problem (And How I Solved It)

Hit an interesting issue when implementing the Asset DTO.

**The requirement:** When you GET an asset, include the date of its most recent maintenance log.

**Note:** This assumes the logs collection is ordered newest-first (for example via `@OrderBy("performedDate DESC")`, or by fetching the latest log with a query).

**First attempt:**
```java
public class AssetDTO {
    private LocalDateTime lastLogDate;
    
    public AssetDTO(Asset asset) {
        // ...
        this.lastLogDate = asset.getLogs().isEmpty() ? null 
            : asset.getLogs().get(0).getPerformedDate();
    }
}
```

**The problem:** `asset.getLogs()` triggers lazy loading. If the persistence context/session is already closed (for example, after the DAO call returns), this throws `LazyInitializationException`.

**The solution:** Calculate it in the service layer while you're still inside an active persistence context:
```java
public AssetDTO get(Integer id) {
    Asset asset = assetDao.get(id);
    
    LocalDateTime lastLogDate = null;
    if (!asset.getLogs().isEmpty()) {  // ← Persistence context still active here
        lastLogDate = asset.getLogs().get(0).getPerformedDate();
    }
    
    AssetDTO dto = new AssetDTO(asset);
    dto.setLastLogDate(lastLogDate);  // ← Set after construction
    return dto;
}
```

**Lesson learned:** DTOs should be "dumb" data containers. Any logic that requires database access belongs in the service layer, not the DTO constructor.

---

## Query Parameters: UI-Friendly Filtering

For the asset list endpoint, I wanted the UI to have a dropdown: "Show all / Show active / Show inactive".

**Implementation:**
```java
// Controller
public void getAll(Context ctx) {
    String activeParam = ctx.queryParam("active");
    Boolean active = activeParam != null ? Boolean.parseBoolean(activeParam) : null;
    ctx.json(assetService.getAll(active));
}

// Service
public List<AssetDTO> getAll(Boolean active) {
    List<Asset> assets;
    if (active == null) {
        assets = assetDao.getAll();  // All assets
    } else {
        assets = assetDaoExpanded.getAllByStatus(active);  // Filtered
    }
    return assets.stream().map(AssetDTO::new).toList();
}
```

**This gives three API calls:**
```
GET /api/v1/assets              → All assets
GET /api/v1/assets?active=true  → Active only
GET /api/v1/assets?active=false → Inactive only
```

---

## Exception Handling: Centralized and Logged

Rather than handling exceptions in every controller method, I configured them once in the Javalin setup:
```java
config.routes.exception(DatabaseException.class, (e, ctx) -> {
    int statusCode = switch (e.getErrorType()) {
        case NOT_FOUND -> 404;
        case CONSTRAINT_VIOLATION -> 409;
        case CONNECTION_FAILURE -> 503;
        case TRANSACTION_FAILURE, QUERY_FAILURE, UNKNOWN -> 500;
    };

    if (statusCode >= 500) {
        log.error("Database error [{}]: {}", e.getErrorType(), e.getMessage(), e);
    } else {
        log.warn("Database error [{}]: {}", e.getErrorType(), e.getMessage());
    }

    ctx.status(statusCode).json(Map.of("status", statusCode, "msg", e.getMessage()));
});
```

**Benefits:**
- DRY — exception mapping in one place
- Consistent error responses across all endpoints
- Automatic logging with appropriate levels (ERROR for 500s, WARN for 4xxs)
- SLF4J logs full cause chain when exception is passed as last parameter

**Enum validation:**

One thing I had to add: try-catch blocks for enum parsing:
```java
try {
    LogStatus status = LogStatus.valueOf(statusParam.toUpperCase());
    ctx.json(logService.getByStatus(status));
} catch (IllegalArgumentException e) {
    throw new ApiException(400, "Invalid status value");
}
```

Without this, sending `?status=INVALID` would crash with an unhandled exception. Now it returns a clean 400 error.

---

## Logging: SLF4J + Logback

Set up proper logging this week with Logback:

**Configuration highlights:**
- Console appender for development (immediate feedback)
- Rolling file appender for production (persistent logs)
- Daily rolling: `javalin-app.log` → `javalin-app-YYYY-MM-DD.log`
- 30 days retention (automatic cleanup)
- App package at DEBUG, frameworks at WARN (reduce noise)

---

## The Great Refactoring: Cleaning Up Main

At this point, `Main.java` had grown to ~80 lines of dependency wiring and configuration. It looked like this:
```java
public static void main(String[] args) {
    IDAO<User> userDao = new UserDAO(emf);
    IUserDAO userDaoExpanded = new UserDAO(emf);
    UserService userService = new UserServiceImpl(userDao, userDaoExpanded);
    UserController userController = new UserController(userService);
    
    // ... repeat for Asset and Log ...
    
    Javalin app = Javalin.create(config -> {
        // 50+ lines of exception handlers and configuration
    }).start(7070);
}
```

**The problem:** Main was doing too much. It was responsible for:
1. Creating all DAOs, services, and controllers
2. Configuring Javalin (plugins, routes, exception handlers)
3. Starting the server

**The refactoring:**

Created two new classes to separate concerns:

**DependencyContainer.java** — handles all object creation:
```java
public class DependencyContainer {
    private static final EntityManagerFactory emf = HibernateConfig.getEntityManagerFactory();
    
    private final UserDAO userDao;
    private final AssetDAO assetDao;
    private final MaintenanceLogDAO logDao;
    
    private final UserService userService;
    private final AssetService assetService;
    private final MaintenanceLogService logService;
    
    private final UserController userController;
    private final AssetController assetController;
    private final MaintenanceLogController logController;
    
    public DependencyContainer() {
        // Create all dependencies in correct order
        this.userDao = new UserDAO(emf);
        // ...
    }
    
    public Routes getRoutes() {
        return new Routes(userController, assetController, logController);
    }
}
```

**AppConfig.java** — handles Javalin configuration:
```java
public class AppConfig {
    public static void start(int port) {
        DependencyContainer container = new DependencyContainer();
        Routes routes = container.getRoutes();
        
        Javalin app = Javalin.create(config -> {
            configurePlugins(config);
            configureRoutes(config, routes);
            configureExceptionHandlers(config);
        });
        
        app.start(port);
    }
    
    private static void configurePlugins(JavalinConfig config) { /* ... */ }
    private static void configureRoutes(JavalinConfig config, Routes routes) { /* ... */ }
    private static void configureExceptionHandlers(JavalinConfig config) { /* ... */ }
}
```

**Main.java after refactoring:**
```java
public static void main(String[] args) {
    AppConfig.start(7070);
    log.info("Server started on port 7070");
}
```

**Two lines.** That's it.

**Benefits:**
- Single Responsibility Principle: each class has one job
- Testable: can inject test dependencies into DependencyContainer
- Maintainable: clear separation makes changes easier
- Clean: Main is now just the entry point, nothing more

---


## Project Structure: The Final Cleanup

After all the refactoring, I reorganized the entire package structure for clarity:
```
app/
├── Main.java
├── config/
│   ├── AppConfig.java
│   ├── DependencyContainer.java
│   └── hibernate/                    # Moved from persistence.config
│       ├── HibernateConfig.java
│       ├── EntityRegistry.java
│       └── ...
├── controllers/
│   ├── UserController.java
│   ├── AssetController.java
│   ├── MaintenanceLogController.java
│   └── routes/
│       ├── Routes.java               # Aggregator
│       ├── UserRoutes.java
│       ├── AssetRoutes.java
│       └── MaintenanceLogRoutes.java
├── dtos/
│   ├── UserDTO.java
│   ├── CreateUserRequest.java
│   ├── AssetDTO.java
│   ├── MaintenanceLogDTO.java
│   └── CreateLogRequest.java
├── entities/
│   ├── User.java
│   ├── Asset.java
│   ├── MaintenanceLog.java
│   └── enums/
│       ├── UserRole.java
│       ├── LogStatus.java
│       └── TaskType.java
├── exceptions/
│   ├── ApiException.java
│   ├── DatabaseException.java
│   └── enums/
│       └── DatabaseErrorType.java
├── integration/
│   ├── RandomUserClient.java
│   ├── RandomUserDTO.java
│   └── seeding/                      # Organized seeding logic
│       ├── ApiUserService.java
│       ├── ApiUserServiceImpl.java
│       └── UserSeeder.java
├── persistence/
│   ├── daos/
│   │   ├── UserDAO.java
│   │   ├── AssetDAO.java
│   │   └── MaintenanceLogDAO.java
│   └── interfaces/
│       ├── IDAO.java
│       ├── IUserDAO.java
│       ├── IAssetDAO.java
│       └── IMaintenanceLogDAO.java
├── services/
│   ├── UserService.java
│   ├── UserServiceImpl.java
│   ├── AssetService.java
│   ├── AssetServiceImpl.java
│   ├── MaintenanceLogService.java
│   └── MaintenanceLogServiceImpl.java
└── utils/
    ├── APIReader.java
    ├── CredentialsHandler.java
    └── PropertyReader.java           # Renamed from Utils.java
```

**Key changes:**
- Hibernate config moved to `app.config.hibernate` (it's application config, not persistence logic)
- Seeding logic organized under `integration.seeding`
- `Utils.java` renamed to `PropertyReader.java` (specific name, not a junk drawer)
- Consistent naming throughout (no more generic "Utils" or "Helper" classes)

---

## API Documentation

### Base URL
```
http://localhost:7070/api/v1
```

---

### Authentication
*Not yet implemented. All endpoints are currently open.*

---

### Quick Reference
- Users: `/users`
- Assets: `/assets`
- Asset logs (create + list): `/assets/{id}/logs`
- Cross-asset logs (queries): `/logs`

---

## User Endpoints

| HTTP Method | Endpoint | Notes | Success | Common Errors |
|-------------|----------|-------|---------|--------------|
| POST | /users | Create user | 201 | 400, 409 |
| GET | /users | List users | 200 | |
| GET | /users/{id} | Get user by ID | 200 | 404 |
| PUT | /users/{id} | Update non-password fields | 200 | 400, 404, 409 |
| DELETE | /users/{id} | Deactivate user (soft delete) | 204 | 404 |
| PATCH | /users/{id} | Reactivate user | 204 | 404 |

**Example: Create user (request)**
```json
{
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "phone": "12345678",
    "role": "TECHNICIAN",
    "password": "securePassword123"
}
```

**Example: User (response)**
```json
{
    "id": 1,
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "phone": "12345678",
    "role": "TECHNICIAN",
    "active": true
}
```

**Validation Rules:**
- Required on create: `firstName`, `lastName`, `email`, `phone`, `role`, `password`
- `email` must match `^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$`
- `role` must be `TECHNICIAN` or `MANAGER`
- Password is hashed with BCrypt and never returned

---

## Asset Endpoints

| HTTP Method | Endpoint | Notes | Success | Common Errors |
|-------------|----------|-------|---------|--------------|
| POST | /assets | Create asset (`active` optional) | 201 | 400 |
| GET | /assets | Optional filter: `?active=true/false` | 200 | |
| GET | /assets/{id} | Get asset by ID | 200 | 404 |
| PATCH | /assets/{id} | Activate asset | 204 | 404 |
| DELETE | /assets/{id} | Deactivate asset | 204 | 404 |

**Example: Create asset (request)**
```json
{
    "name": "Hydraulic Press #3",
    "description": "Main production line hydraulic press",
    "active": true
}
```

**Example: Asset (response)**
```json
{
    "id": 1,
    "name": "Hydraulic Press #3",
    "description": "Main production line hydraulic press",
    "active": true,
    "lastLogDate": null
}
```

**Query Parameters:**
- `active` (optional): `true` = active assets only, `false` = inactive only, omitted = all assets

---

## Maintenance Log Endpoints (Asset-Scoped)

| HTTP Method | Endpoint | Notes | Success | Common Errors |
|-------------|----------|-------|---------|--------------|
| POST | /assets/{id}/logs | Create log for a specific asset | 201 | 400, 404 |
| GET | /assets/{id}/logs | Optional filters: `?taskType=...`, `?status=...` | 200 | 400, 404 |

**Example: Create log (request)**
```json
{
    "performedDate": "2026-03-06T14:30:00",
    "status": "COMPLETED",
    "taskType": "MAINTENANCE",
    "comment": "Replaced hydraulic fluid",
    "performedByUserId": 1
}
```

**Example: Log (response)**
```json
{
    "id": 1,
    "performedDate": "2026-03-06T14:30:00",
    "status": "COMPLETED",
    "taskType": "MAINTENANCE",
    "comment": "Replaced hydraulic fluid",
    "assetId": 1,
    "assetName": "Hydraulic Press #3",
    "performedByUserId": 1,
    "performedByName": "John Doe"
}
```

**Query Parameters:**
- `taskType` (optional): `PRODUCTION`, `MAINTENANCE`, or `ERROR`
- `status` (optional): `PENDING`, `IN_PROGRESS`, `COMPLETED`, or `CANCELLED`

---

## Maintenance Log Endpoints (Standalone)

| HTTP Method | Endpoint | Notes | Success | Common Errors |
|-------------|----------|-------|---------|--------------|
| GET | /logs | Optional filter: `?status=...` | 200 | 400 |
| GET | /logs/{id} | Get log by ID | 200 | 404 |
| GET | /logs/user/{userId} | Logs performed by a user | 200 | 404 |
| GET | /logs/active-assets | Optional: `?limit=10` | 200 | |

**Query Parameters:**
- `status` (optional): Filter by `PENDING`, `IN_PROGRESS`, `COMPLETED`, or `CANCELLED`
- `limit` (optional): Maximum number of results (default: 10)

**Notes:**
- Logs are immutable (no update/delete operations)
- Logs can only be created via `/assets/{id}/logs`

---

## Data Models

### Enums

**UserRole:**
```
TECHNICIAN
MANAGER
```

**LogStatus:**
```
PENDING
IN_PROGRESS
COMPLETED
CANCELLED
```

**TaskType:**
```
PRODUCTION
MAINTENANCE
ERROR
```

---

## Error Response Format

All errors follow this format:
```json
{
  "status": 404,
  "msg": "User not found"
}
```

**HTTP Status Codes:**
- `400` - Bad Request (validation failure, invalid input)
- `404` - Not Found (resource doesn't exist)
- `409` - Conflict (constraint violation, e.g., duplicate email)
- `500` - Internal Server Error (database transaction/query failure)
- `503` - Service Unavailable (database connection failure)

---

## API Notes

- `DELETE` deactivates (`active=false`), `PATCH` reactivates (`active=true`) and both return `204 No Content`
- Passwords are only accepted inbound and never returned (hashed with BCrypt)
- Logs are immutable and always belong to an asset (created via `/assets/{id}/logs`)
- Assets have no general update endpoint (only `active` can change)
- Filtering is done with query parameters, not separate routes

---

## Lessons Learned

**Technical skills:**
- Javalin 7.0.1 configuration patterns and breaking changes
- RESTful API design (nested resources, query parameters, proper HTTP verbs)
- DTO strategy for security and separation of concerns
- Multi-layer validation (input vs. business rules)
- Centralized exception handling with logging
- Lazy loading pitfalls and solutions

**Design patterns:**
- Controller → Service → DAO layering
- Dependency injection via constructor
- Configuration separation (AppConfig, DependencyContainer)
- Route organization with EndpointGroups
- Hybrid DTOs (minimal embedded data for relationships)

**Architecture principles:**
- Single Responsibility Principle (each class has one job)
- Dependency Inversion (depend on interfaces, not implementations)
- Separation of concerns (each layer has distinct responsibilities)
- DRY (exception handling in one place, not scattered)

**Debugging insights:**

The lazy loading issue was a good reminder that **layer boundaries matter**. The DTO constructor runs after the DAO returns, which often means the persistence context/session has already been closed. Any lazy collection access at that point fails.

The solution: keep entity relationship navigation in the service layer while you're still in an active persistence context, and keep DTOs as simple data containers.

---

## What's Next

The REST API layer is complete for a minimal viable product and has the parts I want for now. The entire application now has:
- ✅ Solid persistence layer with exception handling
- ✅ External API integration with concurrent processing
- ✅ Complete REST API with proper layering
- ✅ Input and (limited) business validation
- ✅ Centralized exception handling and logging
- ✅ Clean, maintainable architecture

Next steps:
- API endpoint testing (Rest Assured + Hamcrest)
- Authentication and authorization (JWT tokens?)
- Swagger/OpenAPI documentation generation
- Deployment considerations

---

## Maintenance Log - Weekly Summary (Updated)

#### Updates Since Last Summary

**Current state:** REST API layer is complete on top of the existing persistence + integration work. Routing, DTOs, validation, exception handling, and logging are now in place with a cleaner app entrypoint/config.

**New / changed this week:**
- Javalin 7.0.1 routing moved fully into `config.routes` using `EndpointGroup`
- API versioning baked in from day one (`/api/v1/...`)
- Routes split per entity and aggregated (User / Asset / MaintenanceLog)
- DTOs split for Users (request vs response) to keep passwords out of responses
- Logs exposed both asset-scoped and standalone (nested resources + cross-asset queries)
- Validation split into controller input checks + service business rules
- Centralized exception mapping + Logback logging (consistent API errors + better diagnostics)
- Main refactored into `Main` (entry), `AppConfig` (Javalin), and `DependencyContainer` (wiring)

**Still true (carried forward):**
- Maintenance logs are immutable (audit trail)
- Users/assets are soft-deleted via `active`
- Default to `LAZY` loading; service layer handles any relationship navigation
- DAOs are persistence-focused; services own business rules

---

That's it for this week — next up is testing the API endpoints with Rest Assured + Hamcrest.