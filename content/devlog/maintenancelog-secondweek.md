---
title: "Maintenance Log - Second Week"
tags: ["Devlog", "Project", "3Sem","MaintenanceLog"]
series: ["Maintenance Log"]
date: 2026-02-13
draft: false
---

So, welcome to this second entry of my Devlog. Without further ado, let's continue into this week's additions.

### Relations, DAOs and exception handling
This week's primary goal was to get the necessary relations between my entities up and running, begin to finalize the DAOs for each, and integrate interfaces both for simple CRUD and for specific queries across the board. After that, the project looked a bit like this:

{{< figure
    src="img/MLCDV3.png"
    alt="classDiagram"
    >}}

After that I began to add exception handling across my different CRUD operations. I made the design choice of adding my own custom `DatabaseException` class with some error types I can relate to common HTTP error codes, both to keep low coupling between layers and to have the correlation I need when I start to use `APIException` in the service layer to interpret `DatabaseException`s. For reference, I have thrown in my current custom exception, the error type enum and an example of how it's used in the code:

```java
public enum DatabaseErrorType
{
    CONSTRAINT_VIOLATION , // 409
    NOT_FOUND, // 404
    CONNECTION_FAILURE, // 503
    TRANSACTION_FAILURE, // 500
    QUERY_FAILURE, // 500
    UNKNOWN
}
```

```java
public class DatabaseException extends RuntimeException {
    private final DatabaseErrorType errorType;

    public DatabaseException(String message, DatabaseErrorType errorType) {
        super(message);
        this.errorType = errorType;
    }

    public DatabaseException(String message, DatabaseErrorType errorType, Throwable cause) {
        super(message, cause);
        this.errorType = errorType;
    }

    public DatabaseErrorType getErrorType() {
        return errorType;
    }
}
```

```java
@Override
public User update(User u)
{
    if (u == null || u.getUserId() == null)
    {
        throw new IllegalArgumentException("User and user id are required");
    }

    try (EntityManager em = emf.createEntityManager())
    {
        em.getTransaction().begin();

        try
        {
            User merged = em.merge(u);
            em.getTransaction().commit();
            return merged;
        }
        catch (IllegalArgumentException e)
        {
            if (em.getTransaction().isActive())
            {
                em.getTransaction().rollback();
            }
            throw new DatabaseException("User not found or invalid", DatabaseErrorType.NOT_FOUND, e);
        }
        catch (PersistenceException e)
        {
            if (em.getTransaction().isActive())
            {
                em.getTransaction().rollback();
            }
            throw new DatabaseException("Update User failed", DatabaseErrorType.TRANSACTION_FAILURE, e);
        }
        catch (RuntimeException e)
        {
            if (em.getTransaction().isActive())
            {
                em.getTransaction().rollback();
            }
            throw new DatabaseException("Update User failed", DatabaseErrorType.UNKNOWN, e);
        }
    }
}
```

### Who needs to know about whom?

One of the more interesting design decisions I had while working on the project this week was about relationships — specifically, which entities in my domain model need to *know about* each other, and which ones don't.

It sounds obvious when you say it out loud, but it's one of those things that's easy to just... not think about, and then end up with a bloated model where everything points to everything else for no good reason.

**Asset ↔ MaintenanceLog (Bidirectional)**

This one was a clear yes for bidirectional. An asset *needs* to know about its logs because the whole point of the system is reviewing an asset's maintenance history. You land on an asset, and you expect to see its logs right there.

```java
// Asset side
@OneToMany(fetch = FetchType.LAZY, mappedBy = "asset")
@OrderBy("performedDate DESC")
private List<MaintenanceLog> logs = new ArrayList<>();

// MaintenanceLog side (owning side)
@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "asset_id", nullable = false)
Asset asset;
```

The `MaintenanceLog` owns the relationship (it holds the foreign key), but `Asset` can still navigate to its logs. The `@OrderBy` annotation means they always come back newest first, without me having to sort anything manually.

**MaintenanceLog → User (Unidirectional)**

This one was a deliberate choice to *not* go bidirectional. A log needs to know who performed it — that's just part of the audit trail. But does a `User` need a list of all the logs they've ever written?

In practice, no — at least not in the way this system works. If you want logs by a specific user, you go through the `MaintenanceLogDAO` and query directly. You don't navigate there through the `User` entity. Keeping it unidirectional keeps `User` clean and focused.

```java
// MaintenanceLog side only — User has no collection of logs
@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "performed_by_user_id", nullable = false)
User performedBy;
```

The question I found most useful to ask myself throughout all of this was: *"Will I ever need to navigate this relationship from the other direction in a realistic use case?"*

Not "could I imagine a scenario where...", but actually, in the flow of this application, does it make sense? For assets and logs: yes, absolutely. For users and logs: no, you query for that directly.

It's a small thing, but keeping relationships unidirectional where possible means less to maintain, less risk of accidentally triggering lazy loading where you don't want it, and a domain model that actually reflects how the system gets used — rather than just how it *could* be used in some hypothetical future.

Anyway, small win for thinking before just annotating everything with `@OneToMany` and calling it a day.

And finally a quick rundown of my current design decisions:

### Maintenance Log - Design Decisions (Updated)

#### Updates Since Last Summary

**Current state:** DAO layer mostly complete will probably need to make more specific queries later when the need arises, integration tests planned next.

##### Major Design Changes

**1. Asset Fetch Strategy Changed Back to LAZY**
- **Previous:** `FetchType.EAGER` on `Asset.logs`
- **Current:** `FetchType.LAZY` on `Asset.logs`
- **Why changed back:** EAGER would load all logs for all assets in list queries (performance concern)
- **Solution:** Use explicit queries when logs are needed, rely on lazy loading otherwise

**2. Added Helper Method to Asset**
- **Implementation:** `Asset.addLog(MaintenanceLog log)` helper method added
- **Previous decision:** No helper methods needed for stateless REST API
- **Why changed:** Provides convenient way to maintain bidirectional relationship consistency
- **Usage:** Optional convenience method, not required for persistence

**3. Asset.logs Collection Initialized**
- **Implementation:** `private List<MaintenanceLog> logs = new ArrayList<>()`
- **Rationale:** Prevents `NullPointerException` when using `addLog()` helper method

**4. Added @OrderBy to Asset.logs**
- **Implementation:** `@OrderBy("performedDate DESC")`
- **Rationale:** Logs always returned in chronological order (newest first) without manual sorting

**5. Asset Entity Now Has Selective Mutability**
- **Implementation:** Only `active` field has `@Setter`, other fields immutable after creation
- **Rationale:** 
  - `name` and `description` shouldn't change (audit trail)
  - `active` status needs to change (deactivation workflow)
  - ID never changes (automatically generated)

  **6. Comprehensive Exception Handling Added to DAO Layer**
- **Implementation:** All DAO methods now wrap JPA exceptions in custom `DatabaseException`
- **Pattern:**
  - Input validation throws `IllegalArgumentException` (e.g., null checks)
  - Read failures throw `DatabaseException` with `QUERY_FAILURE` error type
  - Write failures throw `DatabaseException` with `TRANSACTION_FAILURE` error type
  - "Not found" scenarios throw `DatabaseException` with `NOT_FOUND` error type
- **Transaction safety:** All write operations include proper rollback on any exception
- **Rationale:** 
  - Decouples persistence layer from JPA implementation details
  - Provides consistent exception interface for service layer
  - Maps cleanly to HTTP status codes without exposing persistence concerns
  - Distinguishes between different failure types for better error handling

### DAO Method Additions:
- **MaintenanceLogDAO:**
  - `getByPerformedUser(Integer userId)` — cross-user log queries
  - `getLogsOnActiveAssets(int limit)` — filtered + paginated queries
  
- **UserDAO:**
  - `getActiveUsers(int limit)` — paginated active users query

- **AssetDAO:**
  - `setActive(Integer id, boolean active)` — only allowed mutation
  - `getInactiveAssets()` — query deactivated assets

### Structural Changes:
- Removed Task entity entirely
- Added TaskType enum
- Reorganized project structure into separate packages
- Added DAO interfaces layer
- Standardized exception handling across all DAOs
- Removed unnecessary `new ArrayList<>()` wrapping in DAO return statements

---

#### 1. Immutability of Maintenance Logs
- **Decision:** `MaintenanceLog` entries are **never updated or deleted**
- **Implementation:** `MaintenanceLogDAO` has NO `update()` method — throws `UnsupportedOperationException`
- **Rationale:** Data integrity and traceability (GDPR compliance, audit trail)
- **Future consideration:** Log corrections will reference previous entries (handled in service/GUI layer)

---

#### 2. Soft Delete for Users
- **Decision:** Users are deactivated, not deleted (`active` boolean field)
- **Implementation:** 
  - `User.active` field added
  - No `delete()` method in `UserDAO`
  - `update()` method used to set `active = false`
- **Rationale:** Preserve historical data — maintenance logs need to show who performed them, even after users leave

---

#### 3. Immutability of Assets (with Exception)
- **Decision:** Assets are immutable **except for `active` status**
- **Implementation:**
  - Only `active` field has `@Setter` on entity
  - `name` and `description` have no setters (immutable)
  - `AssetDAO.update()` throws `UnsupportedOperationException`
  - `AssetDAO.setActive(Integer id, boolean active)` allows only status changes
  - Uses `find()` + setter pattern, not `merge()`
- **Rationale:** 
  - Asset details (name, description) should not change for audit trail
  - Active status needs to change for operational/deactivation workflow
  - Maintains data integrity while allowing necessary state management

---

#### 4. Task System Redesigned as Enum
- **Decision:** Removed `Task` entity, replaced with `TaskType` enum
- **Implementation:**
```java
  public enum TaskType {
      PRODUCTION,
      MAINTENANCE,
      ERROR
  }
```
- **Previous design:** Task entity with title and description
- **Why changed:** 
  - Task descriptions are unique per log (technician writes what they did)
  - Only the category (title) is predefined
  - Enum is simpler, type-safe, and matches actual use case
  - Specific work details go in `MaintenanceLog.comment` field (now `nullable = false`)
- **Rationale:** Eliminates unnecessary entity and relationship, clearer domain model

---

#### 5. Entity Relationships

##### MaintenanceLog relationships:
- `@ManyToOne` to `Asset` (owning side, `LAZY` loading)
- `@ManyToOne` to `User` (owning side, `LAZY` loading)
- All relationships: `nullable = false`
- **Changed:** Removed `@ManyToOne` to Task (now uses `TaskType` enum)

##### Asset relationship:
- `@OneToMany` to `MaintenanceLog` (non-owning side)
- `mappedBy = "asset"`
- `LAZY` loading
- `@OrderBy("performedDate DESC")` — automatic chronological sorting
- Initialized to empty `ArrayList<>()` to prevent NullPointerException
- Optional helper method: `addLog(MaintenanceLog log)` for bidirectional consistency
- No setter on collection (relationship managed via `MaintenanceLog` creation or helper method)

##### User relationship:
- **Unidirectional** from `MaintenanceLog` to `User`
- User entity has no collection of logs
- **Rationale:** Logs are accessed via Asset or direct queries, not via User

---

#### 6. Helper Methods on Entities
- **Decision:** `Asset.addLog(MaintenanceLog log)` helper method added
- **Previous decision:** No helper methods needed
- **Why changed:** Provides convenient way to maintain bidirectional consistency if needed
- **Usage:** Optional — not required for persistence, can still manage via MaintenanceLog side only
- **Implementation:** Sets both sides of relationship (`logs.add(log)` and `log.setAsset(this)`)

---

#### 7. DAO Layer Responsibilities and Architecture

##### DAOs are "dumb persistence" — no business logic

**Responsibilities:**
- CRUD operations
- Database queries via JPQL
- Exception wrapping (convert JPA exceptions to `DatabaseException`)
- Input validation (`IllegalArgumentException` for null/invalid inputs)

**Not responsible for:**
- Validation (service layer)
- Business rules (service layer)
- HTTP concerns (controller layer)

##### DAO Interfaces
- **Implementation:** Separate interfaces (`IDAO<T>`, `IUserDAO`, etc.) for each DAO
- **Rationale:** 
  - Contract separation for testing
  - Interface Segregation Principle
  - Allows mocking in future tests

---

#### 8. Exception Handling Strategy

##### Custom Exception Hierarchy
```java
public class DatabaseException extends RuntimeException {
    private final DatabaseErrorType errorType;
}

public enum DatabaseErrorType {
    CONSTRAINT_VIOLATION,  // 409
    NOT_FOUND,            // 404
    CONNECTION_FAILURE,   // 503
    TRANSACTION_FAILURE,  // 500
    QUERY_FAILURE,        // 500
    UNKNOWN
}
```

##### Exception Rules
- **"Not found" throws exceptions** (not returns null)
  - **Rationale:** In this system, looking up by ID is expected to succeed (IDs from database)
  - Failed lookup indicates something went wrong (deleted, corrupted)
  - Service layer catches and converts to appropriate HTTP responses
- **Read operations** (`SELECT`) → `DatabaseErrorType.QUERY_FAILURE`
- **Write operations** (`INSERT`, `UPDATE`) → `DatabaseErrorType.TRANSACTION_FAILURE`
- **Rationale:** Clear distinction helps service layer understand failure type
- **Input validation** → `IllegalArgumentException` (not `DatabaseException`)

##### Transaction Management
- **Write operations:** Explicit transaction with proper rollback on all exception types
- **Read operations:** No transaction needed
- **Pattern:**
```java
  em.getTransaction().begin();
  try {
      // operation
      em.getTransaction().commit();
  } catch (PersistenceException e) {
      if (em.getTransaction().isActive()) {
          em.getTransaction().rollback();
      }
      throw new DatabaseException(..., TRANSACTION_FAILURE, e);
  }
```

---

#### 9. Query Strategy
- **Primary key lookups:** Use `em.find()` (simpler, cached, returns null)
  - Then check null and throw `DatabaseException` with `NOT_FOUND`
- **Other queries:** Use JPQL with `TypedQuery` and named parameters
- **getSingleResult() handling:** Catch `NoResultException`, rethrow as `DatabaseException`
- **No unnecessary wrapping:** Return `query.getResultList()` directly (removed `new ArrayList<>()` wrapper)

---

#### 10. User Authentication Considerations
- **Decision:** `UserDAO.getByEmail()` filters by `active = true`
- **Rationale:** Only active users can log in
- **But:** `UserDAO.get(userId)` returns all users (active or not)
- **Rationale:** Historical lookups need to show inactive users (who performed a log)

---

#### 11. Field Protection
- **Decision:** Manual `@Setter` annotations on individual fields
- **Primary keys:** NO setter (immutable after creation)
- **Immutable entities (Asset):** Only `active` field has setter
- **Mutable entities (User):** All fields except ID have setters
- **Rationale:** Prevents accidental ID modification and enforces immutability where needed

---

#### 12. Email Uniqueness
- **Decision:** Database-level constraint + service-layer validation
- **Implementation:** `@Column(unique = true)` on `User.email`
- **Rationale:** Defense in depth — database prevents duplicates, service layer provides better error messages

---

#### 13. Merge vs. Find Pattern for Updates
- **Decision:** Use `find()` + setter, not `merge()` with partial entities
- **Implementation:**
```java
  Asset asset = em.find(Asset.class, id);
  asset.setActive(active);
  // commit auto-flushes
```
- **Rationale:** 
  - Prevents accidentally clearing other fields
  - More explicit about what's being changed
  - No need for defensive find-before-merge
  - Clearer intent

---

#### 14. Fetch Strategy
- **Asset.logs:** `LAZY` loading with `@OrderBy("performedDate DESC")`
  - **Rationale:** Prevents loading all logs when listing assets
  - **Ordering:** Always returns logs newest-first without manual sorting
  - **Access:** Explicit queries or navigation through relationship when needed
- **All MaintenanceLog relationships:** `LAZY` loading

---

#### 15. GetAll() Methods
- **Decision:** Included in all DAOs despite potential performance issues
- **Rationale:** School project requirement to demonstrate functionality
- **Implementation:** 
  - `AssetDAO.getAll()` only returns active assets
  - Some methods have `limit` parameter for pagination (e.g., `getActiveUsers(int limit)`)
- **Note:** Should be paginated for production use

---

#### Architecture Layers (Planned)
```
┌─────────────────────────────────┐
│   REST API (Javalin)            │  ← HTTP status codes
├─────────────────────────────────┤
│   Service Layer                 │  ← Business rules, validation
├─────────────────────────────────┤
│   DAO Layer (Current)           │  ← Database operations
├─────────────────────────────────┤
│   JPA/Hibernate                 │
├─────────────────────────────────┤
│   PostgreSQL                    │
└─────────────────────────────────┘
```