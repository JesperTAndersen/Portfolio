---
title: "Maintenance Log - Fifth Week"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Backend", "Testing", "Architecture", "exam-portfolio"]
series: ["Maintenance Log"]
date: 2026-03-16
draft: false
---

# Devlog Week 5: Interface Refinement, REST API Testing & The Mapper Pattern

This week marked a significant shift from building features to refining architecture and establishing a robust testing foundation. The focus was on three major areas: granular interface design following ISP principles, comprehensive REST API testing with RestAssured, and implementing the DTO mapper pattern.

## What Changed This Week

- **Atomic interface design**: Split `IDAO<T>` into `ICreateDAO`, `IReadDAO`, `IUpdateDAO` — services now depend only on what they use
- **REST API integration tests**: UserRoutes, AssetRoutes, MaintenanceLogRoutes tested with RestAssured + Testcontainers
- **DTO Mapper pattern**: Pure record DTOs with dedicated mapper classes for entity ↔ DTO conversion
- **Constructor chaining**: DependencyContainer supports both production and test configurations via `this(emf)` delegation
- **JOIN FETCH strategy**: Lazy loading issues resolved with query-specific eager fetching

---

## Interface Segregation: From Monolithic to Atomic

Last session ended with a discussion about the Interface Segregation Principle (ISP), and this week I took it to its logical conclusion. The original `IDAO<T>` interface was doing too much — forcing services to depend on methods they'd never call.

### The Problem

My `MaintenanceLogService` needed to fetch `Asset` and `User` entities, but only for reading. Yet it depended on the full `IDAO<T>` interface with `create()`, `update()`, and other unused operations. ISP says: "Clients should not be forced to depend on methods they do not use."

### The Solution: Atomic Interfaces

**Options considered:**
1. Keep `IDAO<T>` and accept overexposure (easy, but violates ISP)
2. Split into atomic interfaces and compose as needed (more upfront work, enforces intent)

I chose option 2. Break down into single-responsibility building blocks:
```java
public interface ICreateDAO<T> {
    T create(T t);
}

public interface IReadDAO<T> {
    T get(Integer id);
    List<T> getAll();
}

public interface IUpdateDAO<T> {
    T update(T t);
}
```

These compose where full create/read/update is needed:
```java
public interface ICrudDAO<T> extends ICreateDAO<T>, IReadDAO<T>, IUpdateDAO<T> {}
```

**Note on deletion:** There's no `IDeleteDAO`. This system uses soft deletes (activation/deactivation) as domain commands, not CRUD operations. Users and Assets have `active` boolean fields; MaintenanceLogs are immutable (never deleted). Hard deletes don't exist in the API — deactivation is a state change handled via `update()` or dedicated methods like `setActive()`.

Entity-specific interfaces combine base operations with domain queries:
```java
public interface IUserDAO extends ICrudDAO<User>, IUserQueries {}
public interface IAssetDAO extends ICreateDAO<Asset>, IReadDAO<Asset>, IAssetQueries {}
public interface IMaintenanceLogDAO extends ICreateDAO<MaintenanceLog>, IReadDAO<MaintenanceLog>, IMaintenanceLogQueries {}
```

Notice `Asset` doesn't extend `IUpdateDAO` — assets are immutable after creation (only activation status changes).

### Services Declare Minimal Dependencies
```java
public class MaintenanceLogServiceImpl {
    private final IMaintenanceLogDAO logDao;  // Full operations
    private final IReadDAO<Asset> assetDao;   // Read-only!
    private final IReadDAO<User> userDao;     // Read-only!
}
```

**Why not use `IAssetDAO` here?** The service only needs `get()` and `getAll()`. Depending on `IReadDAO<Asset>` instead of `IAssetDAO`:
- Makes intent explicit (this service doesn't create/update assets)
- Reduces coupling (service doesn't depend on asset-specific queries it doesn't use)
- Enables compiler enforcement (can't accidentally call `assetDao.create()`)

The compiler now prevents calling `assetDao.create()` or `userDao.update()` in the log service — those methods don't exist on `IReadDAO<T>`.

---

## REST API Testing with RestAssured

With the REST layer complete from last week, this week was all about verification. I set up comprehensive integration tests using RestAssured, Testcontainers, and JUnit 5.

### Test Infrastructure

Each route group gets its own test class with isolated test ports to avoid Javalin binding conflicts when tests run in parallel:
```java
@BeforeAll
public static void init() {
    emf = HibernateTestConfig.getEntityManagerFactory();  // Testcontainers PostgreSQL
    container = new DependencyContainer(emf);
    app = AppConfig.start(container, 7071);  // UserRoutesTest: 7071, AssetRoutesTest: 7072, etc.

    RestAssured.baseURI = "http://localhost:7071";
    RestAssured.basePath = "/" + Routes.getApiVersion();
}

@BeforeEach
void setUp() {
    seeded = TestPopulator.populateUsers(emf);  // TRUNCATE + fresh data each test
}
```

**Testcontainers lifecycle:** One PostgreSQL container per test class (`@BeforeAll` static setup). Container starts once, database is wiped before each test via `TRUNCATE TABLE ... RESTART IDENTITY CASCADE`. This prevents test pollution (leftover data from previous tests causing failures) while keeping startup overhead minimal.

**Why separate ports?** If multiple test classes run in parallel (Maven Surefire default), they'd conflict trying to bind to the same port. Each class gets its own port (7071, 7072, 7073) to enable parallel execution.

The `TestPopulator` ensures predictable state — auto-incrementing IDs reset, foreign key constraints maintained, no orphaned data.

### Constructor Chaining for Testability

To support both production and test configurations, I refactored `DependencyContainer`:
```java
public DependencyContainer() {
    this(HibernateConfig.getEntityManagerFactory());  // Production
}

public DependencyContainer(EntityManagerFactory emf) {
    // Actual wiring logic (used by both constructors)
}
```

Production calls `new DependencyContainer()`. Tests inject a Testcontainers-backed EMF via `new DependencyContainer(testEmf)`. Single source of truth for wiring.

### Test Coverage

Each endpoint tested for:
- Happy paths (200/201/204 with correct data)
- Query parameters (filtering, limits)
- Edge cases (404, 409, 400)
- Idempotency

**Happy path example:**
```java
@Test
void testGetAllActiveUsers() {
    given()
        .when()
        .get("/users?active=true")
        .then()
        .statusCode(200)
        .body("email", containsInAnyOrder(
            seeded.values().stream()
                .filter(User::isActive)
                .map(User::getEmail)
                .toArray()
        ));
}
```

**Edge case example:**
```java
@Test
void postExistingEmailReturns409() {
    User user1 = seeded.get("user1");
    
    given()
        .contentType("application/json")
        .body(String.format("""
            {
                "firstName": "Test",
                "lastName": "User",
                "email": "%s",
                "phone": "12345678",
                "role": "TECHNICIAN",
                "password": "password123"
            }
            """, user1.getEmail()))
        .when()
        .post("/users")
        .then()
        .statusCode(409);
}
```

### The Lazy Loading Gotcha (Again)

Hit a classic Hibernate issue during testing: accessing `asset.getLogs()` after the `EntityManager` closed threw `LazyInitializationException`. 

**Week 4 solution:** Calculate `lastLogDate` in the service layer while the persistence context is active.

**Week 5 solution:** Use `JOIN FETCH` in the DAO for queries that need relationship data:
```java
@Override
public Asset get(Integer id) {
    TypedQuery<Asset> query = em.createQuery(
        "SELECT a FROM Asset a LEFT JOIN FETCH a.logs WHERE a.assetId = :id", 
        Asset.class
    );
    // ...
}
```

**Rule of thumb:**
- **Use `JOIN FETCH`** when the API contract requires relationship data and you want the DAO to guarantee it's loaded
- **Use service-layer calculation** when you only need a derived scalar (like `lastLogDate`) and don't want to widen fetch graphs for all callers

This isn't "defeating" lazy loading — it's consciously choosing the right fetch strategy per query. Other queries that don't need logs still use lazy loading by default.

---

## The Mapper Pattern: Pure DTOs & Separation of Concerns

My lecturer suggested extracting DTO construction logic into dedicated mapper classes. The goal: make DTOs pure data carriers while keeping conversion concerns separate from both entities and services.

### Alternatives Considered

1. **Constructors in DTO** (Week 1–4 approach): Couples DTO to entity, mixes data structure with conversion logic
2. **Mapping in service layer**: Repetitive, spreads mapping logic across services
3. **Dedicated mapper class**: Single responsibility, reusable, testable

I chose option 3.

### Before: Constructor-Based Conversion
```java
public record UserDTO(/* fields */) {
    public UserDTO(User user) {  // Constructor does the mapping
        this(user.getId(), user.getFirstName(), ...);
    }
}
```

### After: Mapper Classes

Pure record:
```java
public record UserDTO(
    Integer id,
    String firstName,
    String lastName,
    String phone,
    String email,
    UserRole role,
    boolean active
) {}
```

Dedicated mapper:
```java
public class UserMapper {
    public static UserDTO toDTO(User user) {
        return new UserDTO(
            user.getUserId(),
            user.getFirstName(),
            user.getLastName(),
            user.getPhone(),
            user.getEmail(),
            user.getRole(),
            user.isActive()
        );
    }
}
```

### When to Map DTO → Entity (or Not)

Most mappers only need `toDTO()` methods. For `User`, the `create()` service method handles entity construction manually because it needs to hash the password and set defaults — business logic that doesn't belong in a mapper.

The exception is `AssetMapper`, which has `toEntity()` since asset creation is straightforward mapping with no special logic.

### Handling Calculated Fields

`AssetDTO` includes `lastLogDate`, which isn't stored — it's calculated from logs. The mapper takes it as an optional parameter:
```java
public class AssetMapper {
    public static AssetDTO toDTO(Asset asset, LocalDateTime lastLogDate) {
        return new AssetDTO(
            asset.getAssetId(),
            asset.getName(),
            asset.getDescription(),
            asset.isActive(),
            lastLogDate
        );
    }
    
    public static AssetDTO toDTO(Asset asset) {
        return toDTO(asset, null);  // For list views
    }
}
```

Service calculates, mapper structures:
```java
public AssetDTO get(Integer id) {
    Asset asset = assetDao.get(id);
    
    LocalDateTime lastLogDate = null;
    if (!asset.getLogs().isEmpty()) {
        lastLogDate = asset.getLogs().get(0).getPerformedDate();
    }
    
    return AssetMapper.toDTO(asset, lastLogDate);
}
```

### Mapper Pattern Tradeoffs

**Why static methods?**

**Pros:**
- Simple (no DI needed)
- Easy to call in streams (`map(AssetMapper::toDTO)`)
- No lifecycle management

**Cons:**
- Harder to mock in tests
- Can grow into "god mapper" if entity relationships get complex
- No polymorphism (can't swap implementations)

For this project, the simplicity wins. If mapping logic gets complex (multiple DTO variants per entity, conditional mapping), I'd reconsider instance-based mappers with DI.

---

## Design Decisions This Week

**35. Atomic Interface Design** — Split `IDAO<T>` into `ICreateDAO`, `IReadDAO`, `IUpdateDAO`; services depend on minimal interfaces (e.g., `IReadDAO<Asset>` instead of full DAO)

**36. Interface Composition** — `ICrudDAO<T>` composes atomic interfaces; entity-specific DAOs extend base + queries (e.g., `IUserDAO extends ICrudDAO<User>, IUserQueries`)

**37. No Hard Deletes** — System uses soft deletes (activation/deactivation) as domain commands; no `IDeleteDAO` or hard delete operations in API

**38. Constructor Chaining for Tests** — Production constructor delegates to test constructor: `this(emf)` pattern enables dependency injection for testing

**39. RestAssured Test Structure** — One test class per route group, fresh database state via `TRUNCATE` in `@BeforeEach`, separate test ports per class (7071, 7072, 7073) to enable parallel execution

**40. Testcontainers Lifecycle** — One PostgreSQL container per test class, started in `@BeforeAll`; database wiped before each test to prevent state leakage

**41. Query-Specific Fetch Strategy** — Use `JOIN FETCH` when API contract requires relationship data; use service-layer calculation for derived scalars; keep lazy loading as default

**42. Mapper Pattern for DTOs** — Pure record DTOs with no constructor logic; mapper classes handle entity ↔ DTO conversion with static methods

**43. Bidirectional Mapping When Needed** — Most mappers only need `toDTO()`; `toEntity()` only when conversion is straightforward (e.g., `AssetMapper` has it, `UserMapper` doesn't)

**44. Calculated Fields as Parameters** — Mapper methods accept calculated fields (e.g., `lastLogDate`) as parameters; service layer handles calculation, mapper handles structure

**45. Method Overloading for Optional Fields** — Overloaded mapper methods for with/without optional parameters; enables method references (`map(AssetMapper::toDTO)`)

**46. Static Mappers for Simplicity** — Static mapper methods chosen over instance-based for ease of use in streams; trade-off accepted: harder to mock, risk of "god mapper" growth

---

## Pending Items

- JWT authentication (token issuing + validation)
- Protect selected routes with `Authorization: Bearer <token>`

---

## Thoughts on Testing

Writing tests highlighted how valuable the layered architecture is. Each layer has clear responsibilities: DAOs worry about persistence, services handle business logic, controllers manage HTTP concerns, mappers structure data. When a test fails, it's immediately obvious which layer has the problem. This clarity is the payoff for all the interface design work.

## Next Week

Implementing JWT-based authentication: issuing tokens on login and validating `Authorization: Bearer <token>` on protected endpoints.