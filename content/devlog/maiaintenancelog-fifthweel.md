---
title: "Maintenance Log - Fifth week"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Backend", "Architecture"]
series: ["Maintenance Log"]
date: 2026-02-06
draft: true
---

## User Stories

---

### **US-1: Create Maintenance Log Entry**

**As a** technician  
**I want to** create a maintenance log entry for an asset  
**So that** I can document work performed and maintain an audit trail

**Given** I am logged in as a technician  
**And** I have selected an asset from the system  
**When** I submit a maintenance log with performed date, task type, status, and comment  
**Then** the log is created and associated with the asset  
**And** the log displays my name as the performer (based on the authenticated user)  
**And** the asset's last log date is updated

**Definition of Done:**
- POST `/api/v1/assets/{id}/logs` endpoint functional
- All required fields validated (date, status, taskType, comment)
- Log is persisted with correct asset and user relationships
- Performer is set to the authenticated user (not supplied by client)
- Response includes asset name and performer name (hybrid DTO)
- Validation failures return clear error messages
- Unauthenticated requests return appropriate error

---

### **US-2: View Asset Maintenance History**

**As a** manager  
**I want to** view all maintenance logs for a specific asset  
**So that** I can review its maintenance history and identify patterns

**Given** I am viewing an asset's details  
**When** I request the maintenance logs for that asset  
**Then** all logs are displayed newest-first  
**And** I can filter by task type (production/maintenance/error)  
**And** I can filter by status (pending/in-progress/completed/cancelled)

**Definition of Done:**
- GET `/api/v1/assets/{id}/logs` endpoint functional
- Logs returned in descending order by performed date
- Query parameters `?taskType=X` and `?status=Y` work correctly
- Invalid enum values return error with clear message
- Response includes performer name and asset name
- Empty array returned if no logs exist

---

### **US-3: Create Asset**

**As a** manager  
**I want to** create a new asset in the system  
**So that** technicians can log maintenance work against it

**Given** I have asset details (name, description)  
**When** I create a new asset  
**Then** the asset is created and appears in the active assets list  
**And** the asset is ready to have maintenance logs attached to it

**Definition of Done:**
- POST `/api/v1/assets` endpoint functional
- All required fields validated (name, description)
- Asset can be created as active or inactive
- Response includes asset details with null lastLogDate
- Validation failures return clear error messages

---

### **US-4: View Asset Details**

**As a** technician  
**I want to** view a specific asset's details  
**So that** I can verify I'm working on the correct equipment

**Given** I have selected an asset  
**When** I request the asset details  
**Then** the asset information is displayed  
**And** the last maintenance date is shown if available

**Definition of Done:**
- GET `/api/v1/assets/{id}` endpoint functional
- Response includes all asset fields (id, name, description, active, lastLogDate)
- lastLogDate is calculated from most recent log (null if no logs)
- Asset not found returns appropriate error

---

### **US-5: View and Filter Asset List**

**As a** manager  
**I want to** view all assets and filter by active status  
**So that** I can manage equipment inventory effectively

**Given** the system contains multiple assets  
**When** I request the asset list  
**Then** active assets are displayed by default  
**And** I can filter to show only inactive assets  
**And** I can request to show all assets

**Definition of Done:**
- GET `/api/v1/assets` endpoint functional
- Query parameter `?active=false` returns only inactive assets
- No query parameter returns all assets
- Each asset includes lastLogDate field
- Empty array returned if no assets match filter

---

### **US-6: Deactivate Asset**

**As a** manager  
**I want to** deactivate an asset that is no longer in use  
**So that** it doesn't appear in active asset lists but history is preserved

**Given** I am viewing an active asset  
**When** I deactivate the asset  
**Then** the asset's active status is set to false  
**And** the asset no longer appears in the default asset list  
**And** all historical maintenance logs remain accessible  
**And** the asset can be reactivated later if needed

**Definition of Done:**
- DELETE `/api/v1/assets/{id}` endpoint functional
- Asset `active` field set to false (soft delete)
- Operation is idempotent (can call multiple times)
- Asset not found returns appropriate error
- GET `/api/v1/assets?active=false` shows deactivated assets
- Asset logs still accessible via `/api/v1/assets/{id}/logs`

---

### **US-7: Create User Account**

**As an** admin  
**I want to** create user accounts for technicians and managers  
**So that** they can log maintenance work under their own credentials

**Given** I have user details (name, email, phone, role)  
**When** I create a new user account  
**Then** the account is created with a secure password  
**And** the user appears in the active users list  
**And** the password is never exposed in API responses

**Definition of Done:**
- POST `/api/v1/users` endpoint functional
- Email uniqueness validated (duplicate returns error)
- Invalid email format rejected with clear error message
- Password stored hashed (BCrypt)
- UserDTO response excludes password field
- Role defaults to TECHNICIAN if not specified
- All required fields validated (firstName, lastName, email, phone, password)

---

### **US-8: Query Logs by Technician**

**As a** manager  
**I want to** view all maintenance logs performed by a specific technician  
**So that** I can review their work quality and productivity

**Given** I select a technician from the user list  
**When** I request their maintenance logs  
**Then** all logs they performed are displayed across all assets  
**And** logs include asset names and dates  
**And** logs are sorted newest-first

**Definition of Done:**
- GET `/api/v1/logs/user/{userId}` endpoint functional
- Returns logs across all assets for specified user
- Response includes asset names via hybrid DTO
- Logs sorted by performed date descending
- User not found returns appropriate error
- Empty array returned if user has no logs

---

### **US-9: Update User Information**

**As an** admin  
**I want to** update user details (name, phone, email, role)  
**So that** employee information stays current

**Given** I am viewing a user's profile  
**When** I update their information  
**Then** the changes are saved  
**And** their email can be changed if unique  
**And** their password is NOT changed via this endpoint  
**And** I can change a user's role between TECHNICIAN and MANAGER

**Definition of Done:**
- PUT `/api/v1/users/{id}` endpoint functional
- All updatable fields except password can be updated (password cannot be changed via this endpoint)
- Email uniqueness validated (duplicate returns error)
- ID in URL must match ID in body (or body ID omitted)
- User not found returns appropriate error
- Role can be updated to any valid UserRole enum value

---

### **US-10: Reactivate User**

**As an** admin  
**I want to** reactivate a previously deactivated user account  
**So that** former employees can regain system access if rehired

**Given** I am viewing a deactivated user's profile  
**When** I reactivate the user  
**Then** the user's active status is set to true  
**And** the user appears in the active users list  
**And** the user can log in and create maintenance logs

**Definition of Done:**
- PATCH `/api/v1/users/{id}` endpoint functional
- User `active` field set to true
- Operation is idempotent (can call multiple times)
- User not found returns appropriate error
- Reactivated user appears in GET `/api/v1/users` results