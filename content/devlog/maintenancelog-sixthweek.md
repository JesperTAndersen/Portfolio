---
title: "Maintenance Log - Sixth Week"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Backend", "Security", "JWT", "exam-portfolio"]
series: ["Maintenance Log"]
date: 2026-03-23
draft: false
---

# Devlog Week 6: JWT Authentication & Role-Based Authorization

This week was entirely dedicated to building a deployment-ready authentication and authorization system. No new domain features—just securing everything that already exists. The focus was on JWT tokens, role hierarchies, and integrating security seamlessly into the existing architecture without breaking tests or existing functionality.

## What Changed This Week

- **Complete JWT authentication system**: Login endpoint, token generation, token verification
- **Role-based authorization**: ADMIN > MANAGER > TECHNICIAN > AUTHENTICATED hierarchy
- **Security integration**: `beforeMatched`/`afterMatched` hooks in Javalin request lifecycle
- **User → Employee refactoring**: Renamed all user-related code to "employee" for clarity with library DTO naming
- **DTO conversion layer**: Domain DTOs separated from library DTOs with explicit conversion
- **All routes protected**: Role requirements on every endpoint except login
- **Test infrastructure updated**: All tests now authenticate before making requests

---

## The Security Layer Problem

This week wasn't about learning JWT from scratch — our course uses a small helper library provided by our lecturer (`dk.bugelhartmann.TokenSecurity`) so issuing and validating tokens is pretty painless.

The real work was fitting authentication + authorization into a codebase that already had a nice separation of concerns, *without* turning the whole project into "security code everywhere".

**What I needed to add (without wrecking anything):** I already had a working REST API with controllers, services, DAOs, and a solid test suite. Now it needed to:
1. Add authentication (who are you?)
2. Add authorization (what can you do?)
3. Keep existing behavior intact
4. Update 100+ tests to include tokens
5. Keep the architecture clean (no library DTOs leaking into controllers/services)

**The constraint:** The helper library has its own DTO shape (`dk.bugelhartmann.UserDTO`) while my domain uses `EmployeeDTO`. Bridging the two cleanly required a conversion layer so the rest of the application doesn't take a dependency on the library.

---

## Authentication vs Authorization: The Javalin Lifecycle

**Request flow**

When a request hits the server, Javalin processes it in this order:
```
1. beforeMatched handlers  ← Run BEFORE finding which route to use
2. Route matching          ← Javalin finds the endpoint
3. afterMatched handlers   ← Run AFTER route found, BEFORE controller
4. Endpoint handler        ← Your controller method
5. after handlers          ← Cleanup
```

**Why does this matter?** Because authentication and authorization need different information, and Javalin only exposes some of it after routing.

**Authentication (`beforeMatched`)**

**Question:** "Is this token valid?"

**What it needs:**
- Token from `Authorization` header
- Secret key to verify signature
- Current timestamp to check expiration

**What it doesn't need:**
- Which endpoint is being called
- What roles are required
- What the request body contains

*The one practical exception:* you still keep a tiny allowlist of open endpoints (like `POST /auth/login`) that should skip token checks.
```java
@Override
public void authenticate(Context ctx) {
    // OPTIONS requests (CORS preflight) skip authentication
    if (ctx.method().toString().equals("OPTIONS")) {
        ctx.status(200);
        return;
    }

    // Open endpoints (e.g. login) skip authentication
    if (isOpenEndpoint(ctx)) {
        return;
    }

    // Extract and verify token
    UserDTO verifiedTokenUser = validateAndGetUserFromToken(ctx);
    ctx.attribute("user", verifiedTokenUser);  // Store for authorize()
}
```

**Why `beforeMatched`?**
- Token validation is the same regardless of which endpoint is being called
- If the token is invalid, reject immediately—don't waste time routing
- Fast fail principle: cheapest check first

**Authorization (`afterMatched`)**

**Question:** "Does this user have permission for THIS specific endpoint?"

**What it needs:**
- The user (from authentication)
- **The route's required roles** (`ctx.routeRoles()`)

**The catch:** `ctx.routeRoles()` is **only available after route matching**. Before matching, Javalin doesn't even know which endpoint you're hitting.
```java
@Override
public void authorize(Context ctx) {
    Set<String> allowedRoles = ctx.routeRoles()  // ← Only exists after matching
        .stream()
        .map(role -> role.toString().toUpperCase())
        .collect(Collectors.toSet());
    
    if (isOpenEndpoint(allowedRoles))
        return;
    
    UserDTO user = ctx.attribute("user");  // Get user from authenticate()
    if (user == null) {
        throw new ForbiddenResponse("No user was added from the token");
    }
    
    if (!userHasAllowedRole(user, allowedRoles)) {
        throw new ForbiddenResponse("Unauthorized");
    }
}
```

**Why `afterMatched`?**
- Needs `ctx.routeRoles()` which only exists post-matching
- Can make role-specific authorization decisions
- Still runs before the controller, so unauthorized requests never reach business logic

**The pattern**

**Authenticate** (beforeMatched): "Do I know who you are?"  
**Authorize** (afterMatched): "Are you allowed to do this specific thing?"

Splitting these concerns into separate lifecycle hooks keeps the code clean and testable.

---

## The DTO Conversion Challenge

**Two DTOs, same job**

The JWT library uses its own DTO:
```java
// dk.bugelhartmann.UserDTO (library)
public class UserDTO {
    private String username;
    private String password;
    private Set<String> roles;  // Multiple roles as strings
}
```

My domain has a different structure:
```java
// app.dtos.EmployeeDTO (domain)
public record EmployeeDTO(
    Integer id,
    String firstName,
    String lastName,
    String phone,
    String email,              // ← Not "username"
    EmployeeRole role,         // ← Single enum, not Set<String>
    boolean active
) {}
```

**The awkward bit:** the token library works with `dk.bugelhartmann.UserDTO`, while the rest of my application speaks in `EmployeeDTO`. I didn't want the library DTO creeping into controller/service code just because security needs it.

**Options I considered:**
1. Use the library DTO throughout the app (easy, but it leaks a course helper into the domain)
2. Fight the library and force everything to use my DTO (more work than it's worth)
3. Keep them separate and convert at the boundary

I went with option 3.

**Conversion layer:** created a single conversion method:
```java
private dk.bugelhartmann.UserDTO convertToLibraryDTO(EmployeeDTO employeeDTO) {
    return new dk.bugelhartmann.UserDTO(
        employeeDTO.email(),                // Maps to username
        Set.of(employeeDTO.role().name())   // Single role as Set<String>
    );
}
```

**Used only when creating tokens:**
```java
private String createToken(EmployeeDTO employeeDTO) {
    // Convert domain DTO to library DTO
    dk.bugelhartmann.UserDTO libraryDTO = convertToLibraryDTO(employeeDTO);
    
    // Library creates token
    return tokenSecurity.createToken(libraryDTO, ISSUER, TOKEN_EXPIRE_TIME, SECRET_KEY);
}
```

**For token verification**, the library DTO stays internal:
```java
private dk.bugelhartmann.UserDTO verifyToken(String token) {
    if (tokenSecurity.tokenIsValid(token, SECRET_KEY) && tokenSecurity.tokenNotExpired(token)) {
        return tokenSecurity.getUserWithRolesFromToken(token);  // Returns library DTO
    }
    throw new ApiException(403, "Token is not valid");
}
```

**Result:** Controllers and services never see the library DTO. It's purely an internal security layer concern.

---

## Role Hierarchy Without Multiple Roles

In this system, each employee has **exactly one role**: ADMIN, MANAGER, or TECHNICIAN. But access control follows a hierarchy:
- ADMIN can do everything MANAGER and TECHNICIAN can do
- MANAGER can do everything TECHNICIAN can do
- TECHNICIAN can only do TECHNICIAN things

**The constraint:** the library supports `Set<String> roles`, but my domain model stores a single `EmployeeRole`. I didn't want to change the data model just to make the token shape happy.

Instead of storing multiple roles anywhere, I just expand permissions at authorization time.

**Store only the actual role in the token:**
```java
// Token contains just the employee's real role
Set.of(employeeDTO.role().name())  // ["ADMIN"] or ["MANAGER"] or ["TECHNICIAN"]
```

**Expand the role during authorization:**
```java
private static final Map<String, Set<String>> ROLE_HIERARCHY = Map.of(
    "ADMIN", Set.of("ADMIN", "MANAGER", "TECHNICIAN", "AUTHENTICATED"),
    "MANAGER", Set.of("MANAGER", "TECHNICIAN", "AUTHENTICATED"),
    "TECHNICIAN", Set.of("TECHNICIAN", "AUTHENTICATED")
);

private static boolean userHasAllowedRole(UserDTO user, Set<String> allowedRoles) {
    // Get user's single role from token
    String userRole = user.getRoles().iterator().next();
    
    // Expand via hierarchy
    Set<String> effectiveRoles = ROLE_HIERARCHY.getOrDefault(userRole, Set.of(userRole));
    
    // Check if any effective role matches required roles
    return effectiveRoles.stream()
        .anyMatch(role -> allowedRoles.contains(role.toUpperCase()));
}
```

**Example flow:**

**Employee logs in as ADMIN:**
- Token contains: `{username: "admin@example.com", roles: ["ADMIN"]}`

**Request to `POST /assets` (requires MANAGER):**
1. `authenticate()` extracts user from token: roles = `["ADMIN"]`
2. `authorize()` gets required roles: `["MANAGER"]`
3. Expands ADMIN via hierarchy: `["ADMIN", "MANAGER", "TECHNICIAN", "AUTHENTICATED"]`
4. Checks if expanded roles contain "MANAGER": **YES**
5. Request proceeds

**Net effect:** one role in the database and token, but the permissions still behave like a hierarchy — and the logic lives in one place.

---

## The AUTHENTICATED Role Pattern

Some endpoints should be accessible to **any logged-in user**, regardless of role:
- `GET /employees` — view employee directory
- `GET /assets` — view asset list
- `GET /logs` — view maintenance history

**The problem:** how do you express "any logged-in user" without copy-pasting three roles onto every route?

**Bad approach:**
```java
get(employeeController::getAll, EmployeeRole.ADMIN, EmployeeRole.MANAGER, EmployeeRole.TECHNICIAN);
```

**This is verbose and brittle** — if you ever add a new role, you'd have to hunt down every endpoint and update it.

The fix was adding a special AUTHENTICATED role.

**Added to enum:**
```java
public enum EmployeeRole implements RouteRole {
    TECHNICIAN,
    MANAGER,
    ADMIN,
    AUTHENTICATED  // ← Special: "any logged-in user"
}
```

**Added to hierarchy:**
```java
private static final Map<String, Set<String>> ROLE_HIERARCHY = Map.of(
    "ADMIN", Set.of("ADMIN", "MANAGER", "TECHNICIAN", "AUTHENTICATED"),
    "MANAGER", Set.of("MANAGER", "TECHNICIAN", "AUTHENTICATED"),
    "TECHNICIAN", Set.of("TECHNICIAN", "AUTHENTICATED")
);
```

**Used in routes:**
```java
get(employeeController::getAll, EmployeeRole.AUTHENTICATED);  // ← Clean!
```

**How it works:**
- AUTHENTICATED is in every role's effective permissions
- Any logged-in user satisfies the requirement
- But unauthenticated requests still get rejected (no token = no roles = fail)

---

## Consolidating Employee Creation

Before this week, there were two ways to create employees:
1. `POST /users` via `UserController` → `UserService.create()`
2. (Planned) `POST /auth/register` via `SecurityController` → `SecurityService.register()`

**The problem:** duplicated logic. Both places would need to:
- Hash passwords
- Check for duplicate emails
- Set default values (`active = true`)
- Create the employee entity
- Return a DTO

So I consolidated it.

**Removed:**
- `EmployeeService.create()` method
- `POST /employees` endpoint
- `EmployeeController.create()` method

**Consolidated into:**
- `SecurityService.register()` — **only way** to create employees
```java
@Override
public EmployeeDTO register(CreateEmployeeRequest request) {
    // Check duplicate email
    if (secDAO.getByEmail(request.email()) != null) {
        throw new ApiException(409, "Email already exists");
    }
    
    // Create employee with hashed password
    Employee employee = Employee.builder()
        .firstName(request.firstName())
        .lastName(request.lastName())
        .email(request.email())
        .phone(request.phone())
        .role(request.role())  // Admin can specify any role
        .password(hashPassword(request.password()))
        .active(true)
        .build();
    
    Employee created = secDAO.create(employee);
    return EmployeeMapper.toDTO(created);
}
```

**Protected the endpoint:**
```java
post("/register", securityController::register, EmployeeRole.MANAGER);
```

**Result:**
- One source of truth for employee creation
- No duplicate password hashing logic
- Security service owns security-related operations (which is where it belongs)
- Managers/admins can create employees; technicians cannot

---

## Testing with Authentication

Once endpoints require tokens, any test that forgets the header gets an instant 403:
```java
// Before (worked last week)
given()
    .when()
    .get("/assets")
    .then()
    .statusCode(200);

// After (fails with 403)
given()
    .when()
    .get("/assets")  // ← No token!
    .then()
    .statusCode(200);  // ← Gets 403 instead
```

So the test setup now logs in first.

**Updated `TestPopulator` to include passwords:**
```java
public static Map<String, Employee> populateEmployees(EntityManagerFactory emf) {
    String hashedPassword = SecurityServiceImpl.hashPassword("password123");
    
    Employee employee1 = Employee.builder()
        .email("Johndoe@mail.dk")
        .password(hashedPassword)  // ← Added!
        .role(EmployeeRole.TECHNICIAN)
        .active(true)
        .build();
    // ... more employees
}
```

**Login in test setup:**
```java
private static String authenticatedToken;
private static String managerToken;
private static String adminToken;

@BeforeEach
void setUp() {
    employees = TestPopulator.populateEmployees(emf);
    assets = TestPopulator.populateAssets(emf);
    
    // Get tokens for each role
    authenticatedToken = loginAsEmployee("Johndoe@mail.dk", "password123");
    managerToken = loginAsEmployee("Janedoe@mail.dk", "password123");
    adminToken = loginAsEmployee("Jeffdoe@mail.dk", "password123");
}

private String loginAsEmployee(String email, String password) {
    return given()
        .contentType("application/json")
        .body(String.format("""
            {
                "email": "%s",
                "password": "%s"
            }
            """, email, password))
    .when()
        .post("/auth/login")
    .then()
        .statusCode(200)
    .extract()
        .path("token");
}
```

**Use in tests:**
```java
@Test
void testGetAllActiveAssets() {
    given()
        .header("Authorization", "Bearer " + authenticatedToken)  // ← Add token
        .when()
        .get("/assets?active=true")
        .then()
        .statusCode(200);
}
```

**Testing authorization**

Once I had tokens for each role, testing authorization became pretty straightforward:
```java
@Test
void testPostAssetAsManager() {
    given()
        .header("Authorization", "Bearer " + managerToken)  // ← Manager token
        .contentType("application/json")
        .body("""{"name": "New Machine", "active": true}""")
    .when()
        .post("/assets")  // Requires MANAGER
    .then()
        .statusCode(201);  // Success
}

@Test
void testPostAssetAsTechnicianFails() {
    given()
        .header("Authorization", "Bearer " + authenticatedToken)  // ← Technician token
        .contentType("application/json")
        .body("""{"name": "Should Fail", "active": true}""")
    .when()
        .post("/assets")  // Requires MANAGER
    .then()
        .statusCode(403);  // Forbidden
}
```

**Result:** all the existing tests were easy to update systematically, and it gave me confidence that the security rules were actually enforced end-to-end.

---

## The User → Employee Refactoring

With the library using `dk.bugelhartmann.UserDTO` and my domain talking about users too, it got confusing fast. In code reviews it was always: "wait — which `UserDTO` is this?"

**The confusion:**
```java
import dk.bugelhartmann.UserDTO;  // Library
import app.dtos.UserDTO;           // Domain - COMPILE ERROR!

// Now what?
UserDTO user = ...  // Which one?!
```

**The fix:** rename my domain entities to `Employee`.

**Renamed:**
- `User` → `Employee`
- `UserDTO` → `EmployeeDTO`
- `CreateUserRequest` → `CreateEmployeeRequest`
- `UserService` → `EmployeeService`
- `UserController` → `EmployeeController`
- `UserDAO` → `EmployeeDAO`
- All tests, all routes, all references

**Result:** much clearer intent. `UserDTO` is the library token DTO, `EmployeeDTO` is my domain DTO.

---

## Access Control Matrix

Here's the permission structure I ended up with:

| Endpoint | Role Required | Who Can Access |
|----------|---------------|----------------|
| **Authentication** |
| `POST /auth/login` | None | Anyone |
| `POST /auth/register` | `MANAGER` | Manager, Admin |
| **Employees** |
| `GET /employees` | `AUTHENTICATED` | All logged-in users |
| `GET /employees/{id}` | `AUTHENTICATED` | All logged-in users |
| `PUT /employees/{id}` | `MANAGER` | Manager, Admin |
| `DELETE /employees/{id}` | `ADMIN` | Admin only |
| `PATCH /employees/{id}` | `ADMIN` | Admin only |
| **Assets** |
| `GET /assets` | `AUTHENTICATED` | All logged-in users |
| `GET /assets/{id}` | `AUTHENTICATED` | All logged-in users |
| `POST /assets` | `MANAGER` | Manager, Admin |
| `PATCH /assets/{id}` | `MANAGER` | Manager, Admin |
| `DELETE /assets/{id}` | `ADMIN` | Admin only |
| **Logs (nested)** |
| `GET /assets/{id}/logs` | `AUTHENTICATED` | All logged-in users |
| `POST /assets/{id}/logs` | `TECHNICIAN` | Technician, Manager, Admin |
| **Logs (standalone)** |
| `GET /logs` | `AUTHENTICATED` | All logged-in users |
| `GET /logs/{id}` | `AUTHENTICATED` | All logged-in users |
| `GET /logs/employee/{id}` | `MANAGER` | Manager, Admin |

**The pattern:** Read access for authenticated users, write access for appropriate roles, destructive actions for admins only.

---

## Design Decisions This Week

**47. JWT Authentication via Lecturer-Provided Helper Library** — Used `dk.bugelhartmann.TokenSecurity` for token generation/verification; conversion layer isolates library DTO from domain

**48. Security Service Owns Employee Creation** — Consolidated all employee creation through `/auth/register`; removed `EmployeeService.create()` to avoid duplication

**49. beforeMatched for Authentication** — Token validation runs before route matching; doesn't need to know endpoint requirements

**50. afterMatched for Authorization** — Role checking runs after route matching; requires `ctx.routeRoles()` which is only available post-match

**51. Library DTO Internal to Security Layer** — `dk.bugelhartmann.UserDTO` used only for token operations; never exposed to controllers or returned in responses

**52. Domain DTO for All External Communication** — `EmployeeDTO` used in controller responses and service layer; clean separation from library implementation details

**53. Conversion Method Pattern** — `convertToLibraryDTO()` handles mapping between domain and library DTOs; single responsibility, easy to test

**54. Role Hierarchy via Map** — Static map defines role inheritance; expanded during authorization check rather than storing multiple roles in token

**55. Single Role per Employee** — Database stores one `EmployeeRole` enum; hierarchy expansion happens at authorization time, not in data model

**56. AUTHENTICATED Role for Any Logged-In User** — Special role that all roles inherit; enables "authenticated but any role" endpoints without listing all roles

**57. Open Endpoints Have No Roles** — Absence of roles = open endpoint; explicit rather than special marker role

**58. Soft Delete for Employees** — Inactive employees can't login (checked in `login()`); preserved in database for historical maintenance log references

**59. Login Returns Token + User** — Client gets both JWT token and user details in single response; avoids extra request to fetch user data

**60. Password Hashing with BCrypt** — Static `hashPassword()` method (salt factor 12); reused in service and test seeding

**61. Test Tokens Per Role** — Each test class maintains tokens for TECHNICIAN, MANAGER, ADMIN; enables comprehensive role-based testing

**62. Seeded Test Employees with Known Passwords** — `TestPopulator` creates employees with `password123`; enables login during test setup

---

## Thoughts on Security Implementation

I expected adding auth to be one of those "touch everything, break everything" weeks, but it honestly went smoother than I thought — mostly because the structure was already doing its job. Having clear layers (routes → controllers → services → DAOs) meant I could bolt security on at the edges instead of sprinkling checks throughout business logic.

The biggest "cost" wasn't getting it to work, it was making it feel *clean*: keeping authentication separate from authorization, keeping the lecturer-provided token types inside the security layer, and getting the role rules readable instead of turning every route into a list of three roles.

The `beforeMatched`/`afterMatched` split was the main mental hurdle. Once it clicked that route roles only exist after matching, the design stopped feeling like ceremony and started feeling like the request lifecycle doing me a favor.

And yes — updating the tests was a lot of mechanical work, but it also felt like a good sign: if adding tokens to tests is mostly systematic, the API surface is probably consistent.

## Next Week

Building out the deployment pipeline. The core functionality is solid; time to harden it and prepare for production.