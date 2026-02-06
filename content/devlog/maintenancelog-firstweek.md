---
title: "Maintenance Log - First Week"
tags: ["Devlog", "Project", "3Sem","MaintenanceLog"]
date: 2026-02-06
draft: false
---

So first of all welcome to this first week post of the development of my little maintenance log, just for posterity lets start with what is in my readme for the project, since that actually breaks down what it its:

{{< github repo="JesperTAndersen/MaintenanceLog" showThumbnail=true >}}
----

# Maintenance Log Backend

This project is a backend API for managing maintenance history of assets such as machines, vehicles, or equipment.

The system focuses on traceability and data integrity by storing maintenance activities as immutable logs. Each log represents a concrete maintenance action performed on an asset at a specific point in time.

The application is developed incrementally as a school project on the third semester at EK. Lyngby, where new backend technologies and architectural concepts are gradually introduced and integrated into the same system.

### Core concepts
- Assets that require maintenance
- Maintenance logs representing performed work
- Users with different roles (e.g. technician, manager, admin)
- Historical data that should not be modified or deleted

### Initial scope
The initial version of the system focuses on a simple domain model with assets and maintenance logs, exposing basic CRUD functionality through a REST API.

Authentication, authorization, validation, testing, and deployment concerns will be added progressively as the project evolves.

### Goal
The goal of this project is to build a production-ready backend system that demonstrates clean structure, realistic business rules, and continuous technical progression.


----

So after that piece of business pitch, lets go into what actually happened this week:

### The Project & Hibernate
So this weeks focus was to get hibernate integrated into the project and understand how the different annotations works and what they do. I started by defining what entities my project should have and how my structure should looks like going forward. I started by sketching a quick class diagram in a plantUML.

{{< figure
    src="img/MLCDV1.png"
    alt="classDiagram"
    >}}

after that i got to work on actually creating relations (or trying since i dont actually know if theses work yet), and creating a some DAO classes for MaintenanceLog and User. The final result after todays work looked like this:

{{< figure
    src="img/MLCDv2.png"
    alt="classDiagram2"
    >}}
    

and finally a quick rundown of my current design decisions:

### Maintenance Log - Design Decisions

#### 1. Immutability of Maintenance Logs
- **Decision:** `MaintenanceLog` entries are **never updated or deleted**
- **Implementation:** `MaintenanceLogDAO` has NO `update()` or `delete()` methods
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

#### 3. Entity Relationships

##### MaintenanceLog relationships:
- `@ManyToOne` to `Asset` (owning side)
- `@ManyToOne` to `Task` (owning side)
- `@ManyToOne` to `User` (owning side)
- All relationships: `LAZY` loading, `nullable = false`

##### Asset relationship:
- `@OneToMany` to `MaintenanceLog` (non-owning side)
- `mappedBy = "asset"`
- `LAZY` loading
- No setter (only managed via `MaintenanceLog` creation)

---

#### 4. No Helper Methods on Entities
- **Decision:** No `addLog()` helper method in `Asset`
- **Rationale:** Stateless REST API — entities reloaded fresh each request, so in-memory bidirectional sync not needed
- **Implementation:** All relationships managed via builder pattern on owning side (`MaintenanceLog`)

---

#### 5. DAO Layer Responsibilities

##### DAOs are "dumb persistence" — no business logic

**Responsibilities:**
- CRUD operations
- Database queries
- Return `null` for not-found (consistent with `em.find()`)

**Not responsible for:**
- Validation (service layer)
- Business rules (service layer)
- HTTP concerns (controller layer)

---

#### 6. Transaction Management
- **Decision:** Transactions only on write operations
- **Read operations** (`SELECT` queries): NO transaction
- **Write operations** (`INSERT`, `UPDATE`): Transaction required

---

#### 7. Query Strategy
- **Primary key lookups:** Use `em.find()` (simpler, cached)
- **Other queries:** Use JPQL with named parameters
- **Consistency:** Methods return `null` when not found (not exceptions)
  - Exception: `getByEmail()` catches `NoResultException` and returns `null`

---

#### 8. User Authentication Considerations
- **Decision:** `UserDAO.getByEmail()` filters by `active = true`
- **Rationale:** Only active users can log in
- **But:** `UserDAO.get(userId)` returns all users (active or not)
- **Rationale:** Historical lookups need to show inactive users (who performed a log)

---

#### 9. Field Protection
- **Decision:** Manual `@Setter` annotations on individual fields
- **Primary keys:** NO setter (immutable after creation)
- **Other fields:** Setters allowed for updates
- **Rationale:** Prevents accidental ID modification

---

#### 10. Email Uniqueness
- **Decision:** Database-level constraint + service-layer validation
- **Implementation:** `@Column(unique = true)` on `User.email`
- **Rationale:** Defense in depth — database prevents duplicates, service layer provides better error messages


---

### Portfolio Site
Finally to end this "little" entry for the week, i think actually talking a little about this hugo/blowfish template thing is in its place. Is has been a fun little side thing to get up and running and i looks forward to tinkering with the different settings and find out how everything actually operates as i go further along. Im doubtful if i will use this for other things than a Dev Log atm. but maybe i should start my new life as a Bread Blogger aswell. I just think there is too much sourdough in the world already, and i think i get only a short runway on throwing up pictures of my normal yeast bread.