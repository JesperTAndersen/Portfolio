---
title: "Maintenance Log - Final Backend Week"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Backend", "Security", "Validation", "Integration"]
series: ["Maintenance Log"]
date: 2026-04-09
draft: false
---

# Devlog Week 8: Final Backend Hardening & Reflection

This is my last backend-focused post before I shift to a React frontend. This week wasn’t about new domain features — it was about hardening what I already built: tightening a security flaw, centralizing validation, and cleaning up the external API seeding integration.

## What Changed This Week

- **Server-side employee attribution**: Maintenance logs now derive `performedByEmployeeId` from JWT token (prevents impersonation)
- **Auth identity helper**: Removed repeated “get auth user → resolve employeeId” logic from controllers
- **Password change endpoint**: Implemented a secure `PATCH /employees/{id}/password` flow
- **Password util extraction**: Moved BCrypt hashing/verification into a dedicated `PasswordUtil`
- **Centralized validation utility**: All format/length validation in one place
- **Two-layer validation**: Controllers check required fields, services enforce format rules
- **Integration seeding cleanup**: Refactored the RandomUser seeding code and separated “demo benchmarking” from actual seeding

---

## Preventing Employee Impersonation

### The Security Flaw

When a technician creates a maintenance log, the system needs to record who performed the work. Originally, the client sent `performedByEmployeeId` in the request body.

**The problem:** Any authenticated user could claim the work was done by someone else.

**Example malicious request:**
```http
POST /api/v1/assets/1/logs
Authorization: Bearer <token>

{
  "performedDate": "2026-04-09T10:00:00",
  "status": "DONE",
  "taskType": "MAINTENANCE",
  "comment": "Routine check",
  "performedByEmployeeId": 5  ← Technician claims manager did the work
}
```

This breaks audit trail integrity. You can't trust who actually performed maintenance.

---

### The Fix: Server-Side Attribution

The employee who performed the work is now **derived from the JWT token**, not the request body.

**Updated request (client can't specify performer):**
```http
POST /api/v1/assets/1/logs
Authorization: Bearer <token>

{
  "performedDate": "2026-04-09T10:00:00",
  "status": "DONE",
  "taskType": "MAINTENANCE",
  "comment": "Routine check"
}
```

**Controller extracts performer from token:**
```java
public void createLogForAsset(Context ctx) {
    // Get authenticated user from JWT token
    UserDTO tokenUser = ctx.attribute("authUser");
    if (tokenUser == null) {
        throw new ApiException(401, "Missing authenticated employee");
    }
    
    // Resolve employee ID from email
    String authEmail = tokenUser.getUsername();
    Integer performedById = employeeIdentityService.getEmployeeIdByEmail(authEmail);
    
    if (performedById == null) {
        throw new ApiException(401, "Missing authenticated employee ID");
    }
    
    // Validate request body (no performedByEmployeeId field)
    CreateLogRequest body = ctx.bodyValidator(CreateLogRequest.class)
        .check(dto -> dto.performedDate() != null, "Performed date required")
        .check(dto -> dto.status() != null, "Status required")
        .check(dto -> dto.taskType() != null, "Task type required")
        .check(dto -> dto.comment() != null, "Comment required")
        .get();
    
    // Build request with server-controlled employee ID
    CreateLogRequest request = new CreateLogRequest(
        body.performedDate(),
        body.status(),
        body.taskType(),
        body.comment(),
        performedById  ← Server sets this
    );
    
    ctx.status(201).json(logService.create(assetId, request));
}
```

**Result:** Logged work always attributed to the authenticated user. No way to impersonate.

---

### ISP: EmployeeIdentityService

The controller needs to resolve an email to an employee ID. It shouldn't depend on the full `EmployeeService` or touch DAOs directly.

**New minimal interface:**
```java
public interface EmployeeIdentityService {
    Integer getEmployeeIdByEmail(String email);
}
```

**Implementation:**
```java
public class EmployeeIdentityServiceImpl implements EmployeeIdentityService {
    private final IEmployeeEmailQuery employeeDao;
    
    @Override
    public Integer getEmployeeIdByEmail(String email) {
        Employee employee = employeeDao.getByEmail(email);
        return employee == null ? null : employee.getEmployeeId();
    }
}
```

**Why this matters:** The controller depends only on what it needs (Interface Segregation Principle). Easy to test, easy to mock, no coupling to full employee service.

---

## Auth Helper + Password Changes

Once I started pulling identity from the token in multiple places (maintenance logs, employee operations, etc.), I noticed the same boilerplate creeping into controllers:

- Read authenticated user from `ctx.attribute("authUser")`
- Extract the username/email
- Resolve employee ID from the database
- Return 401 if anything is missing

To keep controllers focused on request/response logic, I extracted that pattern into a small helper (`EmployeeAuthUtil`). It returns a simple value object (authenticated library user + resolved employee ID), and controllers can just call a single “require auth” method instead of repeating the same guards.

I also kept ISP intact by leaning on a minimal identity contract (`EmployeeIdentityService`) so controllers don’t have to depend on an entire service API just to look up an ID. In practice, `EmployeeService` extends `EmployeeIdentityService`, so controllers can accept the minimal contract even when they’re passed the full service.

The controller call ends up as a single line:
```java
Integer employeeId = EmployeeAuthUtil.requireAuthenticatedEmployee(ctx, employeeService).id();
```

This is the same pattern I now use when deriving `performedByEmployeeId` for maintenance logs, instead of duplicating token extraction and email → employee ID lookups.

---

### Password Change Endpoint (Secured)

I added a password change route:

- `PATCH /employees/{id}/password` (role: AUTHENTICATED)

Key checks:

- Requires both `oldPassword` and `newPassword` in the JSON body (non-null / non-blank)
- Resolves the authenticated employee from the token via `EmployeeAuthUtil`
- Enforces that the path ID matches the authenticated employee ID (otherwise 403)
- Verifies the old password against the stored hash before updating
- Validates the new password using the same centralized rules (`ValidationUtil.validatePasswordNonNull(newPassword)`)

This makes it hard to accidentally expose a “change anyone’s password” endpoint, and it forces a re-auth style proof (old password) before persisting the new one.

---

### Password Crypto Refactor (PasswordUtil)

Previously, BCrypt hashing/verification lived in `SecurityServiceImpl`. That worked, but it created awkward coupling:

- Seeding, DAOs, and tests were calling security-layer helpers

So I extracted password hashing and verification into a dedicated utility (`PasswordUtil`) and removed the helper methods from `SecurityServiceImpl`. The side effect is that persistence-layer code can verify passwords without depending on the security service at all.

I also updated the remaining callers (seeding, DAO verification, and test populators) to use `PasswordUtil` so there’s a single place for the hashing rules.

**Sanity check:** After migrating the call sites, the Maven test suite still passed.

---

## Centralized Validation

Previously, validation logic was scattered: some in controllers (Javalin bodyValidator), some in services (ad-hoc checks), inconsistent error messages.

### The ValidationUtil

All format/length validation now lives in one utility class:

```java
public final class ValidationUtil {
    // Precompiled regex (performance optimization)
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$");
    private static final Pattern PHONE_PATTERN = 
        Pattern.compile("^[0-9+()\\s-]{6,20}$");
    
    public static void lengthBetween(String value, String fieldName, int min, int max) {
        if (value == null) return;
        int length = value.trim().length();
        if (length < min || length > max) {
            throw new ApiException(400, 
                String.format("%s must be between %d and %d characters", fieldName, min, max));
        }
    }
    
    public static void validateEmailNonNull(String email) {
        matches(email, EMAIL_PATTERN, "Invalid email format");
        if (email.trim().length() > 254) {  // RFC 5321 limit
            throw new ApiException(400, "Email is too long");
        }
    }
    
    public static void validatePhoneNonNull(String phone) {
        matches(phone, PHONE_PATTERN, "Invalid phone format");
    }
    
    public static void validatePasswordNonNull(String password) {
        lengthBetween(password, "Password", 4, 72);  // BCrypt limit
    }
}
```

**Validation rules:**
- **Email:** Regex + max 254 chars (RFC standard)
- **Phone:** 6-20 chars, allows `+`, `()`, spaces, `-`
- **Password:** 4-72 chars (BCrypt input limit)
- **Names:** 2-50 chars

All throw `ApiException(400, message)` for consistent error handling.

---

### Two-Layer Validation Pattern

Validation happens at two levels with different responsibilities:

**Layer 1: Controller - Required Fields**

Controllers reject requests missing required fields:

```java
public void register(Context ctx) {
    CreateEmployeeRequest request = ctx.bodyValidator(CreateEmployeeRequest.class)
        .check(dto -> dto.firstName() != null && !dto.firstName().trim().isEmpty(), 
               "First name is required")
        .check(dto -> dto.lastName() != null && !dto.lastName().trim().isEmpty(), 
               "Last name is required")
        .check(dto -> dto.email() != null && !dto.email().trim().isEmpty(), 
               "Email is required")
        .check(dto -> dto.password() != null && !dto.password().trim().isEmpty(), 
               "Password is required")
        .check(dto -> dto.phone() != null && !dto.phone().trim().isEmpty(), 
               "Phone is required")
        .check(dto -> dto.role() != null, "Role is required")
        .get();
    
    ctx.status(201).json(securityService.register(request));
}
```

**Why at controller level?** Fast rejection. If firstName is missing, don't waste time on service-layer validation.

---

**Layer 2: Service - Format & Business Rules**

Services enforce format constraints, length limits, and business rules:

```java
@Override
public EmployeeDTO register(CreateEmployeeRequest request) {
    // Format validation
    ValidationUtil.lengthBetween(request.firstName(), "First name", 2, 50);
    ValidationUtil.lengthBetween(request.lastName(), "Last name", 2, 50);
    ValidationUtil.validateEmailNonNull(request.email());
    ValidationUtil.validatePhoneNonNull(request.phone());
    ValidationUtil.validatePasswordNonNull(request.password());
    
    // Business rule: email uniqueness
    if (secDAO.getByEmail(request.email()) != null) {
        throw new ApiException(409, "Email already exists");
    }
    
    // Sanitize input (trim whitespace)
    Employee employee = Employee.builder()
        .firstName(request.firstName().trim())
        .lastName(request.lastName().trim())
        .email(request.email().trim())
        .phone(request.phone().trim())
        .role(request.role())
        .password(hashPassword(request.password()))
        .active(true)
        .build();
    
    Employee created = secDAO.create(employee);
    return EmployeeMapper.toDTO(created);
}
```

**Why at service level?** Consistent enforcement across all entry points (register, update, seeding). Reusable validation logic.

---

### Why Two Layers?

| Layer | Checks | Rationale |
|-------|--------|-----------|
| **Controller** | Required fields present | Early rejection, don't waste service time |
| **Service** | Format, length, business rules | Consistent enforcement, reusable logic |

Both throw `ApiException(400)`, so error responses are uniform.

---

## Integration Seeding Cleanup

I originally built the RandomUser seeding integration earlier in the project (week 3). This week I revisited it because it had grown into a mix of “real seeding” and “benchmark/demo” code.

What I changed:
- Separated benchmarking from actual seeding, so it doesn’t run every time by default
- Tightened up the multithreaded fetch to be safer (caps on threads, guaranteed shutdown)
- Made failures per-batch instead of failing the entire seed run
- Kept the fixed seed endpoint for deterministic demo/test data

The main win here wasn’t “more concurrency” — it was making the seeding tool predictable, maintainable, and less intrusive while I’m focusing on the core API.

---

## Reflection: What Went Well

**Architecture choices paid off:**
- Layered architecture made it easy to add security without rewriting controllers
- ISP in DAOs meant services only depend on what they need
- DTO mappers kept entities separate from API contracts

**Security was added cleanly:**
- JWT library isolated in security layer via conversion methods
- beforeMatched/afterMatched split makes sense once you understand Javalin's lifecycle
- Role hierarchy without multiple database roles keeps the data model simple

**Testing strategy worked:**
- RestAssured + Testcontainers caught bugs early
- Separate test ports prevented port conflicts
- Seeded data with known passwords made auth tests straightforward

---

## Reflection: What Was Hard

**JWT library integration:**
- Having two different `UserDTO` types was confusing until I renamed domain entities to Employee
- Understanding when to use beforeMatched vs afterMatched took some trial and error

**Validation duplication:**
- Initially had validation scattered across controllers and services
- Centralizing in ValidationUtil should have happened earlier

**Testing with authentication:**
- Every test breaking when I added security was tedious
- Should have built security earlier to avoid mass test updates

**Deployment setup:**
- Database table creation strategy (Hibernate update vs migrations) still feels hacky
- Manual admin seeding is functional but not elegant

---

## What I'd Do Differently

**1. Centralize validation sooner**  
Validation logic duplicated across services before I built ValidationUtil. Should have been a week 2 task.

**2. Add input sanitization earlier**  
Trimming strings before persistence should have been part of the initial validation strategy, not added later.

---

## What's Next: Frontend in React

The backend is functionally complete:
- Full CRUD for employees, assets, maintenance logs
- JWT authentication with role-based authorization
- Centralized validation
- ISP-compliant DAO layer
- Comprehensive test coverage
- External API integration for seeding

**Next up:** Building a React frontend that consumes this API.

**Planned features:**
- Login page with JWT token management
- Employee dashboard (role-dependent views)
- Asset list with search/filter
- Maintenance log creation form
- Manager analytics (logs by employee, status distribution)

**Technical stack:**
- React (JavaScript)
- React Router for navigation
- Fetch API for HTTP requests
- Context API for auth state

The backend is done for now. Time to make it usable.