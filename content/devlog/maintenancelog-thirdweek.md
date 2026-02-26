---
title: "Maintenance Log - Third Week"
tags: ["Devlog", "Project", "3Sem","MaintenanceLog","API"]
series: ["Maintenance Log"]
date: 2026-02-26
draft: false
---
# Devlog Week 3: External API Integration & Concurrent Processing

Alright, welcome back! After a little bit of a break due to unforeseen circumstances, I've now continued my work on this little project of mine. The goal this week was to integrate and fetch some data from an external API and use it somehow — so, without further ado:

---

## Choosing the Right API for Testing

Choosing an API was actually the first hurdle I had to overcome. The core issue was that from a design standpoint (and for the main use case), I didn't actually *need* any external information — I just wanted realistic seed data.

So I looked around a bit and found this nifty little API that can generate random user information. The goal was to have data I could enter and use in my database instead of generic "User1", "User2", etc., but also to not have to hand-write all these guys myself. The added bonus is that I can make the output deterministic by using a fixed seed, which is perfect for testing.

And yes: it also gives me users I can log in as later by digging out the passwords *before* they get hashed (obviously only for testing — not real production).

**Key decision:** 
- API: RandomUser.me
- Purpose: Generate realistic test users for development
- Benefit: Known passwords (via fixed seed) for testing authentication later

---

## The Fixed Seed Trade-off

After settling on RandomUser.me, I started playing around with the endpoint configuration. The API is actually pretty flexible — you can specify which fields you want, which nationalities, and even use a seed for deterministic results.

I ended up with this:
```java
"https://randomuser.me/api/?results=%d&nat=gb,dk&inc=name,login,email,phone&seed=myfixedseed123"
```

Breaking this down:
- `nat=gb,dk` — Only British and Danish users (seemed more realistic for a Denmark-based system than getting users from everywhere)
- `inc=name,login,email,phone` — Only return the fields I actually need, so I don't have to ignore a ton of JSON
- `seed=myfixedseed123` — The big one: this makes the API return the same users every time

The seed parameter was great for predictability, but it created an interesting problem I didn't anticipate. When I tried running my multi-threaded fetch (more on that later), I got duplicate key violations in the database. Took me a minute to realize what was happening: all my threads were calling the API with the same seed, so they were all getting back the same 5 users!

**The solution:** I ended up creating two endpoints — one with the fixed seed for actual seeding (predictable, known data), and one without for demonstrating the multi-threading speedup (random data, no duplicates).

---

## Building a Generic API Reader

One thing I wanted to avoid was writing API-specific HTTP and JSON handling code every time I needed to call an external service. So I built a generic `APIReader` class that could handle any JSON API, not just RandomUser.

The idea was: generic HTTP fetching + JSON parsing in one place, then specific client classes (like `RandomUserClient`) that know about the API's structure and endpoints.

This also meant I had to think about exception handling. My first pass was pretty lazy:
```java
catch (IOException | URISyntaxException e) {
    throw new IllegalArgumentException("Could not retrieve data from the provided URL. Try again later");
}
```

This loses all context — you don't know what URL failed, why it failed, or whether retrying even makes sense.

After some back-and-forth, I landed on this approach:
```java
catch (URISyntaxException e) {
    String safeUrl = url.split("\\?")[0];  // Redact query params (API keys, etc.)
    throw new IllegalArgumentException("Invalid URL: " + safeUrl + " error: " + e.getMessage(), e);
}
catch (IOException e) {
    String safeUrl = url.split("\\?")[0];
    throw new RuntimeException("API call failed for " + safeUrl + ": " + e.getMessage(), e);
}
```

**Why separate the exceptions?**
- `URISyntaxException` = programming error (malformed URL string)
- `IOException` = runtime error (network failure, bad JSON, etc.)

Different problems, different handling. And by chaining the original exception (`e`), I preserve the full stack trace for debugging.

---

## DTO Pattern & Nested JSON Mapping

The RandomUser API returns nested JSON, which meant I needed to map it properly. The structure looks like this:
```json
{
  "name": {
    "first": "John",
    "last": "Doe"
  },
  "login": {
    "password": "secret"
  },
  "email": "john@example.com",
  "phone": "123-456-7890"
}
```

My first instinct was to flatten this in the DTO using `@JsonProperty`:
```java
@JsonProperty("name.first")
private String firstName;
```

Turns out Jackson doesn't support dot notation like that. It just looks for a field literally called `"name.first"` at the root level.

**The actual solution:** Mirror the JSON structure with nested records:
```java
@JsonProperty("name")
private Name name;

@JsonProperty("login")
private Login login;

public record Name(String first, String last) {}
public record Login(String password) {}
```

Then when I need the data in the service layer:
```java
dto.getName().first()
dto.getLogin().password()
```

**One gotcha I hit:** I initially made the records `private`, which meant I couldn't access them from outside the DTO class. Had to make them `public` for the service layer to use them.

---

## ExecutorService: When Concurrency Actually Matters

This was probably the most interesting part of the week — demonstrating actual, measurable speedup from multi-threading.

The setup was simple:
- **Sequential:** Make 5 API calls one after another, each fetching 10 users
- **Concurrent:** Make 5 API calls in parallel using ExecutorService, each fetching 10 users

I timed both approaches:
```
Sequential (5x10): 1447ms
Concurrent (5 threads, 50 total): 216ms
Speedup: 6.7x
```

The concurrent version was almost 7 times faster. The reason is that network I/O is the bottleneck here. When you make a request, most of the time is spent waiting for the server to respond. With multi-threading, all 5 threads can be waiting simultaneously instead of one at a time.

**Exception handling in concurrent code:**

One thing I had to think through was: what happens if one of the API calls fails? Should the whole seeding operation fail, or should I just skip that batch and continue with the others?

I went with the latter:
```java
for (Future<List<RandomUserDTO>> f : futures) {
    try {
        List<RandomUserDTO> userDTOList = f.get();
        users.addAll(userDTOList);
    } catch (ExecutionException e) {
        System.out.println("Batch failed: " + e.getCause().getMessage());
        // Log and continue — don't stop the whole process
    }
}
```
small note: for the moment this is just a print statement, when i set up some logging that will change, but since this is a once only in non-production operation, this works for the moment.

This way, if one batch fails (network hiccup, bad JSON, whatever), I still get the other 4 batches worth of users. For a one-time seeding operation, partial success is better than complete failure.

---

## Service Layer: Where Business Logic Lives

With the API client done, I needed a service layer to handle the actual business logic: converting DTOs to entities, assigning roles, hashing passwords, and persisting everything.

I kept the DAO layer "dumb" — it just handles database operations. All the logic lives in the service:

**Responsibilities:**
- DTO → Entity conversion
- Role assignment (every 5th user = MANAGER, rest = TECHNICIAN)
- Password hashing with BCrypt
- Orchestrating the whole seeding process

**One design decision I revisited: when to hash passwords**

Initially, I had the flow like this:
1. Convert DTOs to User entities (with plain text passwords)
2. Loop through and hash all passwords
3. Persist

But this meant plain text passwords were sitting in memory between steps 1 and 2, which felt wrong. I refactored to hash immediately during conversion:
```java
User.builder()
    .firstName(dto.getName().first())
    .lastName(dto.getName().last())
    .email(dto.getEmail())
    .password(hashPassword(dto.getLogin().password()))  // Hash right away
    .active(true)
    .build()
```

This way, plain text passwords never even make it into a User entity. They're hashed the moment they're pulled from the DTO.

---

## Dependency Injection & Interface Usage

I hit a small compilation error that turned into a good learning moment about dependency injection.

I had declared my DAO as an interface in `Main`:
```java
IDAO<User> userDao = new UserDAO(emf);
```

But my service was expecting the concrete class:
```java
public class ApiUserServiceImpl {
    private final UserDAO userDao;  // Concrete class
}
```
This failed to compile when I tried to pass the interface-typed variable to the service.

The fix was simple: make the service depend on the abstraction, not the implementation:
```java
public class ApiUserServiceImpl {
    private final IDAO<User> userDao;  // Interface
}
```

**Why this matters:**
- I can now inject a mock DAO for testing without changing the service
- The service doesn't care about UserDAO's specific implementation details
- This is basically the Dependency Inversion Principle in action

Small change, but it makes the code more flexible and testable.

---

## Security: BCrypt Password Hashing

One thing I definitely didn't want was plain text passwords in the database. Even for test data, it's bad practice to get into the habit of storing passwords unhashed.

I used BCrypt, which handles salting automatically:
```java
private String hashPassword(String password) {
    return BCrypt.hashpw(password, BCrypt.gensalt());
}
```

Each password gets a unique salt generated by `BCrypt.gensalt()`, which means even if two users have the same password, the hashes will be different.

**The benefit of using a fixed seed for the API:**
- I know the plain text passwords (they're in the API response)
- They're hashed in the database
- When I implement authentication later, I can test it with known credentials

So I get realistic password security while still having predictable test data. Win-win.

---

## Lessons Learned

**Technical skills:**
- ExecutorService and concurrent programming (and seeing real, measurable speedup)
- Jackson JSON mapping, especially with nested structures
- Exception handling strategies: preserve context, chain causes, handle different error types differently
- BCrypt password hashing

**Design patterns:**
- DTO pattern for API integration (separating external data structure from internal entities)
- Dependency injection with interfaces (program to abstractions, not implementations)
- Separation of concerns across layers (DAO = persistence, Service = business logic)

**Debugging insights:**

The duplicate user issue was a good lesson in understanding how tools work. I assumed "concurrent API calls" + "fixed seed" would work fine, but I didn't think through that the seed makes the API deterministic — same seed + same parameters = same results. All my threads were fetching identical users.

Once I understood that, the fix was obvious: either use different seeds/pages per thread, or just don't use multi-threading with a fixed seed. I went with the latter for actual seeding (single-threaded, predictable) and kept multi-threading for the performance demo (random data).

---

## What's Next

The persistence layer is solid, I've got test data, and I've demonstrated some concurrency skills. Next up is probably the REST API layer with Javalin — time to actually expose this thing over HTTP and start thinking about authentication and authorization.

---

## Appendix: Current folder structure

Just for quick context (and so this post has *somewhere* to point when I mention packages), here's the *trimmed* version of the project structure. I'm only including the stuff that's relevant to this week's work (API integration + services + persistence).

```
src/
└─ main/
   ├─ java/
   │  └─ app/
   │     ├─ Main.java
   │     ├─ entities/
   │     │  └─ model/ (Asset, MaintenanceLog, User)
   │     ├─ exceptions/ (ApiException, DatabaseException, DatabaseErrorType)
   │     ├─ integration/
   │     │  ├─ client/ (RandomUserClient)
   │     │  ├─ dto/ (RandomUserDTO)
   │     │  └─ util/ (APIReader)
   │     ├─ persistence/ (Hibernate config + DAOs)
   │     └─ services/ (ApiUserService + implementation)
   └─ resources/ (config.properties, logback.xml)
```


## Maintenance Log - Design Decisions (Updated)

#### Updates Since Last Summary

**Current state:** Persistence layer is solid, DAOs/services are in place, and I've now got a repeatable way to seed realistic test users (including a concurrency demo).

#### Carried forward (still true)

Before the new API stuff (skip to #16 for those), these are still the rules of the world:

- Maintenance logs are immutable (audit trail)
- Users are soft-deleted (`active` boolean)
- Assets are immutable except for `active`
- Relationships are intentional (only bidirectional where navigation is actually used)
- Default to `LAZY` loading; pull related data explicitly when needed
- DAOs are “dumb persistence”; services contain business logic
- Persistence errors are wrapped in custom exceptions (`DatabaseException` + error type)
- `TaskType` is an enum (not an entity)
- Logs come back newest-first (`@OrderBy("performedDate DESC")`)

#### Major Design Changes (This Week)

##### 16. API Integration Architecture
- **Decision:** Generic `APIReader` + specific client classes
- **Implementation:** `APIReader` handles HTTP/JSON, `RandomUserClient` handles RandomUser-specific logic
- **Rationale:** Reusability — can add other API integrations (weather, stock prices, etc.) without duplicating HTTP/JSON code
- **Exception handling:** Separate `URISyntaxException` (programming error) from `IOException` (runtime error), preserve original exception with cause chaining

##### 17. Fixed Seed for Test Data
- **Decision:** Use fixed seed for seeding, random for demonstrations
- **Implementation:** Two endpoint configurations — fixed (`seed=myfixedseed123`) and random
- **Rationale:** Predictable test data with known passwords for authentication testing
- **Trade-off discovered:** Fixed seed + multi-threading = duplicate users (all threads get same results)
- **Solution:** Use single-threaded for actual seeding, multi-threaded for performance demo with random data

##### 18. Password Security
- **Decision:** Hash passwords immediately during entity creation (not after)
- **Implementation:** BCrypt with auto-generated salt, hashing happens in DTO → Entity conversion
- **Rationale:** Never store plain text passwords in memory or database, even temporarily
- **Password field:** Added `password` field to User entity for future authentication features
- **Testing benefit:** Fixed seed means known plain text passwords for testing authentication flow

##### 19. Concurrent API Fetching
- **Decision:** Support both single and multi-threaded fetching via configuration
- **Implementation:** ExecutorService with configurable thread count, `invokeAll()` for batch submission
- **Performance:** ~6.7x speedup (1447ms sequential vs 216ms concurrent for 50 users)
- **Exception handling:** Individual batch failures logged and skipped, partial results returned
- **Rationale:** Demonstrates concurrency benefits, graceful degradation on partial failure

##### 20. Service Layer Responsibilities
- **Decision:** Service layer handles all business logic, DAO remains "dumb persistence"
- **Implementation:**
  - DTO → Entity conversion
  - Role assignment logic (every 5th user = MANAGER, rest = TECHNICIAN)
  - Password hashing (BCrypt)
  - Orchestration of seeding process
  - Check for existing users before seeding
- **DAO layer:** Only CRUD operations, no business rules
- **Rationale:** Clear separation of concerns, testable business logic, DAO can be swapped without affecting logic

##### 21. DTO Structure for Nested JSON
- **Decision:** Use nested records that mirror API response structure
- **Implementation:** Inner `record Name(String first, String last)` and `record Login(String password)`
- **Attempted approach that failed:** `@JsonProperty("name.first")` — Jackson doesn't support dot notation
- **Correct approach:** Nested structures with proper Jackson mapping
- **Visibility:** Records must be `public` (not `private`) for access from service layer

---

That's it for this week's progress. Solid chunk of work, and I'm actually pretty happy with how clean the integration turned out. Next time: making this thing actually respond to HTTP requests.