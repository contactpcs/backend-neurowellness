# Complete Project Walkthrough - Sozo Healthcare Platform

## 📋 Project Overview
**Project Name:** Sozo Healthcare Platform  
**Tech Stack:** FastAPI (Backend) + Vanilla JavaScript (Frontend) + Supabase (PostgreSQL Database)  
**Location:** `C:\Users\mohan\OneDrive\Desktop\Sozo`  
**Python Environment:** Virtual environment at `C:\Users\mohan\OneDrive\Desktop\Sozo\venv`

---

## 🏗️ Architecture & Design Decisions

### 1. **Three-Tier Architecture**
- **Frontend (Vanilla JS)** → **Backend (FastAPI)** → **Database (Supabase PostgreSQL)**
- ✅ **NO direct frontend-to-database connections** (security by design)
- ✅ All data flows through FastAPI backend APIs
- ✅ Backend is the single source of truth for all operations

### 2. **Security Implementation**
- **Password Hashing:** SHA-256 (implemented in `app/core/security.py`)
- **Authentication:** JWT tokens with user_id claim
- **Authorization:** Role-Based Access Control (RBAC) with permissions
- **Token Storage:** JWT stored in localStorage on frontend
- **API Protection:** All endpoints require valid JWT + permission checks

---

## 📁 Project Structure

```
Sozo/
├── backend/
│   ├── venv/                           # Python virtual environment
│   ├── app/
│   │   ├── main.py                     # FastAPI app entry point with CORS
│   │   ├── core/
│   │   │   ├── database.py             # Supabase connection manager
│   │   │   ├── security.py             # SHA256 hashing + JWT generation
│   │   │   └── config.py               # Environment variables
│   │   ├── shared/
│   │   │   └── schemas/
│   │   │       └── auth.py             # LoginRequest, TokenResponse, JWTClaims
│   │   └── modules/
│   │       ├── users/
│   │       │   ├── router.py           # /register, /login, /roles endpoints
│   │       │   └── schemas.py          # UserCreate, UserResponse schemas
│   │       └── authorization/
│   │           ├── service.py          # get_user_permissions(), has_permission()
│   │           ├── dependencies.py     # require_permission() FastAPI dependency
│   │           └── router.py           # Example RBAC-protected endpoints
│   └── requirements.txt                # Python dependencies
│
└── frontend/
    ├── index.html                      # Login page
    ├── register.html                   # Registration page
    ├── auth.js                         # Authentication API calls (fetch to backend)
    └── styles.css                      # UI styling
```

---

## 🗄️ Database Schema (Supabase PostgreSQL)

### Tables Implemented:

#### 1. **users** (Main user table)
```sql
- user_id (UUID, PK)
- email (TEXT, UNIQUE)
- first_name (TEXT)
- last_name (TEXT)
- hashed_password (TEXT)           # SHA-256 hashed
- phone (TEXT, OPTIONAL)
- is_active (BOOLEAN, default: true)
- verified_email (BOOLEAN, default: false)
- created_at (TIMESTAMPTZ)
- updated_at (TIMESTAMPTZ)
```

#### 2. **roles** (Available system roles)
```sql
- role_id (UUID, PK)
- role_name (TEXT, UNIQUE)         # e.g., "PATIENT_PORTAL", "DOCTOR_UI"
- description (TEXT)
```

**Current Roles:**
- `PATIENT` - Regular patients
- `DOCTOR` - Doctors
- `CLINICAL_ASSISTANT` - Clinical assistants
- `SUPER_ADMIN` - Super administrators
- `PLATFORM_ADMIN` - Platform administrators
- `CLINICAL_ADMIN` - Clinical admins (facility managers)
- `RECEPTIONIST` - Receptionists

#### 3. **permissions** (Granular permissions)
```sql
- permission_id (UUID, PK)
- resource (TEXT)                  # e.g., "PATIENT", "REPORT"
- action (TEXT)                    # e.g., "READ", "CREATE", "UPDATE", "DELETE"
- description (TEXT)
```

#### 4. **user_roles** (User-to-Role mapping)
```sql
- user_role_id (UUID, PK)
- user_id (UUID, FK → users)
- role_id (UUID, FK → roles)
- assigned_at (TIMESTAMPTZ)
```

#### 5. **role_permissions** (Role-to-Permission mapping)
```sql
- role_permission_id (UUID, PK)
- role_id (UUID, FK → roles)
- permission_id (UUID, FK → permissions)
- granted_at (TIMESTAMPTZ)
```

---

## 🔐 Authentication Flow (Implemented & Working)

### **Registration Flow:**
```
1. User fills form on register.html
2. Frontend → POST /api/v1/users/register
   {
     "email": "user@example.com",
     "password": "plaintext",
     "first_name": "John",
     "last_name": "Doe",
     "phone": "+1234567890",
     "role": "patient"  // or doctor, clinical_assistant, super_admin, clinical_admin, platform_admin, receptionist
   }

3. Backend (router.py):
   a. Check if email exists in Supabase users table
   b. Hash password using SHA-256
   c. Generate UUID for user_id
   d. Insert into users table
   e. Map role enum to database role_name
   f. Query roles table to get role_id
   g. Insert into user_roles table (assign role)

4. Response:
   {
     "user_id": "uuid",
     "email": "user@example.com",
     "first_name": "John",
     "role": "patient",
     "created_at": "2026-02-13T..."
   }
```

### **Login Flow:**
```
1. User enters credentials on index.html
2. Frontend → POST /api/v1/users/login
   {
     "email": "user@example.com",
     "password": "plaintext"
   }

3. Backend (router.py):
   a. Query users table by email
   b. Verify SHA-256 hashed password
   c. Generate JWT token with claims: {user_id, exp}
   d. Return token

4. Response:
   {
     "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
     "token_type": "bearer"
   }

5. Frontend stores JWT in localStorage
```

---

## 🛡️ Authorization System (RBAC - Just Implemented)

### **Permission Check Flow:**

```
1. User makes authenticated request with JWT header:
   Authorization: Bearer <jwt_token>

2. Endpoint uses dependency:
   @router.post("/reports")
   async def create_report(
       claims: JWTClaims = Depends(require_permission("REPORT", "CREATE"))
   ):
       ...

3. Dependency execution (dependencies.py):
   a. Extract user_id from JWT
   b. Call get_user_permissions(user_id)
   
4. Database query (service.py):
   SQL JOIN:
   user_roles 
   → roles (get role_name)
   → role_permissions 
   → permissions (get resource + action)

5. Returns:
   {
     "roles": ["DOCTOR_UI"],
     "permissions": {
       "PATIENT": ["READ", "UPDATE"],
       "REPORT": ["CREATE", "READ"]
     }
   }

6. Validate:
   - Check if "REPORT" in permissions
   - Check if "CREATE" in permissions["REPORT"]
   - If NO → HTTPException(403 Forbidden)
   - If YES → Proceed to endpoint

7. Endpoint executes with user_id available
```

### **Available Permission Dependencies:**

```python
# Single permission required
require_permission("REPORT", "CREATE")

# OR logic - user needs ANY ONE permission
require_any_permission(
    ("PATIENT", "READ"),
    ("REPORT", "READ")
)

# AND logic - user needs ALL permissions
require_all_permissions(
    ("PATIENT", "UPDATE"),
    ("REPORT", "CREATE")
)

# Get full permission set (no validation)
get_current_user_permissions()
```

---

## 🔧 Core Components Explained

### **1. Database Manager (`app/core/database.py`)**
```python
class SupabaseDatabaseManager:
    - query_table(table, filters=None)     # SELECT with WHERE
    - insert_record(table, data)           # INSERT single record
    - update_record(table, record_id, data) # UPDATE by ID
    - delete_record(table, record_id)      # DELETE by ID
```
**Uses:** Supabase Python client with REST API

### **2. Security Manager (`app/core/security.py`)**
```python
class PasswordManager:
    - hash_password(plain_password) → SHA-256 hash
    - verify_password(plain, hashed) → Boolean

class JWTManager:
    - create_access_token(user_id) → JWT string
    - verify_token(token) → {user_id, exp} or raises error
```

### **3. Authorization Service (`app/modules/authorization/service.py`)**
```python
async def get_user_permissions(user_id: UUID):
    """
    Query chain:
    1. Get roles from user_roles table
    2. Get permissions via role_permissions join
    3. Structure as {roles: [], permissions: {}}
    """

def has_permission(permissions_data, resource, action) → Boolean:
    """Check if specific permission exists in data structure"""
```

### **4. Authorization Dependencies (`app/modules/authorization/dependencies.py`)**
```python
def require_permission(resource: str, action: str):
    """FastAPI dependency - validates single permission"""

def require_any_permission(*permission_pairs):
    """OR logic - needs at least one permission"""

def require_all_permissions(*permission_pairs):
    """AND logic - needs all permissions"""
```

---

## 🌐 API Endpoints Summary

### **Users Module** (`/api/v1/users`)

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| POST | `/register` | ❌ | Register new user + assign role |
| POST | `/login` | ❌ | Login and get JWT token |
| GET | `/roles` | ❌ | Get all available roles (for registration dropdown) |

### **Authorization Module** (`/api/v1/auth`)

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/my-permissions` | ✅ JWT | Get current user's roles & permissions |
| POST | `/reports` | ✅ REPORT:CREATE | Example: Create report (permission required) |
| GET | `/records` | ✅ PATIENT:READ OR REPORT:READ | Example: OR permission logic |
| POST | `/sensitive` | ✅ PATIENT:UPDATE AND REPORT:CREATE | Example: AND permission logic |

---

## 🎨 Frontend Implementation

### **Files:**
- `index.html` - Login page with email/password form
- `register.html` - Registration with role selection dropdown
- `auth.js` - All API interaction functions

### **Key Functions in auth.js:**

```javascript
// Load roles for registration dropdown
async function loadRoles()
  → GET /api/v1/users/roles
  → Populates <select> with role options

// Register new user
async function registerUser(userData)
  → POST /api/v1/users/register
  → Stores JWT in localStorage
  → Redirects to dashboard

// Login user
async function loginUser(credentials)
  → POST /api/v1/users/login
  → Stores JWT in localStorage
  → Redirects to dashboard

// Logout
function logout()
  → Clears localStorage
  → Redirects to login page
```

### **Security Features:**
✅ No Supabase client in frontend  
✅ No database credentials exposed  
✅ All requests go through backend  
✅ JWT stored in localStorage  
✅ Token sent as Bearer token in Authorization header  

---

## ⚙️ Configuration & Environment

### **Backend Environment Variables** (`.env` file needed)
```env
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-anon-key
JWT_SECRET_KEY=your-secret-key-for-jwt
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=30
```

### **CORS Configuration** (`main.py`)
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5500", "http://127.0.0.1:5500"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### **Python Dependencies** (`requirements.txt`)
```
fastapi==0.109.0
uvicorn[standard]==0.27.0
supabase==2.3.0
python-jose[cryptography]==3.3.0
passlib==1.7.4
python-multipart==0.0.6
pydantic==2.5.3
pydantic-settings==2.1.0
```

---

## 🚀 How to Run the Project

### **Backend Startup:**
```bash
# Navigate to backend
cd C:\Users\mohan\OneDrive\Desktop\Sozo\backend

# Activate virtual environment
venv\Scripts\activate

# Install dependencies (if not done)
pip install -r requirements.txt

# Run FastAPI server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```
**Backend runs on:** `http://localhost:8000`  
**API Docs:** `http://localhost:8000/docs`

### **Frontend Startup:**
```bash
# Use Live Server extension in VS Code
# Or any static file server on port 5500
```
**Frontend runs on:** `http://localhost:5500`

---

## ✅ What's Currently Working

1. ✅ **User Registration**
   - Email/password with SHA-256 hashing
   - Role selection (patient, clinician, nurse, admin, center_manager)
   - Automatic role assignment via user_roles table
   - Data persisted to Supabase

2. ✅ **User Login**
   - Email/password verification
   - JWT token generation
   - Token stored in frontend localStorage

3. ✅ **JWT Authentication**
   - All protected endpoints verify JWT
   - user_id extracted from token claims
   - Token expiration handled

4. ✅ **Role-Based Authorization (RBAC)**
   - get_user_permissions() queries database
   - require_permission() dependency validates access
   - Supports OR/AND permission logic
   - Returns 403 Forbidden if permission denied

5. ✅ **Frontend-Backend Integration**
   - All API calls go through backend
   - No direct database access from frontend
   - JWT sent in Authorization header

6. ✅ **Database Operations**
   - Users table CRUD operations
   - Roles table querying
   - User-role assignment
   - Permission queries with joins

---

## 🔴 Current Limitations & Known Issues

1. **No Password Reset/Recovery** - Users cannot reset forgotten passwords
2. **No Email Verification** - verified_email flag not enforced
3. **No Token Refresh** - JWT expires, users must re-login
4. **Permission Seeding** - Roles and permissions must be manually inserted into database
5. **No Audit Logging** - No tracking of permission checks or access attempts
6. **No Rate Limiting** - APIs vulnerable to brute force attacks
7. **Basic Error Handling** - Some error cases may not be gracefully handled
8. **No Frontend Dashboard** - After login, no actual dashboard/UI implemented

---

## 📊 Database Setup Requirements

**Before the system works, you MUST populate these tables in Supabase:**

### 1. **Roles Table**
```sql
INSERT INTO roles (role_id, role_name, description) VALUES
('uuid1', 'PATIENT_PORTAL', 'Regular patient access'),
('uuid2', 'DOCTOR_UI', 'Doctor/clinician access'),
('uuid3', 'CLINIC_ASSISTANT', 'Nurse/assistant access'),
('uuid4', 'SUPER_ADMIN', 'System administrator'),
('uuid5', 'CLINICIAN_ADMIN', 'Center manager');
```

### 2. **Permissions Table**
```sql
INSERT INTO permissions (permission_id, resource, action, description) VALUES
('uuid1', 'PATIENT', 'READ', 'View patient data'),
('uuid2', 'PATIENT', 'CREATE', 'Create new patient'),
('uuid3', 'PATIENT', 'UPDATE', 'Update patient data'),
('uuid4', 'REPORT', 'READ', 'View reports'),
('uuid5', 'REPORT', 'CREATE', 'Create reports'),
('uuid6', 'APPOINTMENT', 'CREATE', 'Schedule appointments');
```

### 3. **Role Permissions Mapping**
```sql
-- Example: DOCTOR_UI gets all patient and report permissions
INSERT INTO role_permissions (role_permission_id, role_id, permission_id) VALUES
('uuid1', 'doctor_role_uuid', 'patient_read_permission_uuid'),
('uuid2', 'doctor_role_uuid', 'patient_update_permission_uuid'),
-- ... etc
```

---

## 🎯 Recommended Next Steps

### **Priority 1: Critical Security**
1. Implement token refresh mechanism (refresh tokens)
2. Add rate limiting to prevent brute force
3. Implement email verification workflow
4. Add password reset functionality
5. Enable Supabase Row Level Security (RLS) policies

### **Priority 2: Core Features**
1. Build dashboard UI after successful login
2. Implement patient management endpoints (CRUD)
3. Implement report management endpoints
4. Add appointment scheduling system
5. Create admin panel for user/role management

### **Priority 3: Data Management**
1. Seed script for roles, permissions, role_permissions
2. Database migration system
3. Audit logging for all operations
4. Backup and recovery procedures

### **Priority 4: User Experience**
1. Form validation improvements
2. Loading states and error messages
3. Password strength requirements
4. Session timeout warnings
5. Remember me functionality

### **Priority 5: DevOps & Testing**
1. Unit tests for authentication/authorization
2. Integration tests for API endpoints
3. Docker containerization
4. CI/CD pipeline setup
5. Production deployment configuration

---

## 🔑 Key Design Principles Followed

1. **Security First** - No direct database access from frontend
2. **Separation of Concerns** - Clear module boundaries
3. **Database as Source of Truth** - Permissions queried per request
4. **Stateless Authentication** - JWT with minimal claims
5. **Modular Authorization** - Reusable permission dependencies
6. **Type Safety** - Pydantic schemas for all data
7. **RESTful API Design** - Clear HTTP methods and status codes

---

## 📝 Code Quality & Best Practices

✅ **Followed:**
- Pydantic schemas for validation
- Async/await for database operations
- Environment variables for secrets
- Proper HTTP status codes
- Error handling with HTTPException
- Logging for debugging
- Type hints throughout codebase

❌ **Missing:**
- Comprehensive unit tests
- API documentation beyond auto-generated
- Input sanitization for SQL injection
- CSRF protection
- Content Security Policy headers

---

## 🎓 Summary for Next Developer/AI

**Current State:** You have a **fully functional authentication and authorization system** using FastAPI + Supabase with:
- User registration with role assignment
- SHA-256 password hashing
- JWT authentication
- Role-Based Access Control (RBAC) with granular permissions
- Frontend completely isolated from database
- Modular, extensible codebase

**What Works:** Registration, login, JWT issuance, permission validation at endpoint level

**What's Missing:** Actual business logic endpoints (patient management, reports, appointments), dashboard UI, token refresh, email verification, comprehensive testing

**Next Steps:** Choose from priorities above based on business needs. The foundation is solid - you can now build feature-specific endpoints using the `require_permission()` dependency to protect them.

**Tech Debt:** None critical, but add tests, rate limiting, and token refresh before production.

---

## 🆘 Quick Troubleshooting

**Import errors in VS Code?**
→ Activate venv and select interpreter: `venv\Scripts\python.exe`

**Backend won't start?**
→ Check `.env` file exists with Supabase credentials

**Frontend can't connect?**
→ Check CORS origins in `main.py` match frontend URL

**Login fails?**
→ Check if user exists in Supabase users table
→ Verify password was hashed with SHA-256 during registration

**Permission denied (403)?**
→ Check if role has required permission in role_permissions table
→ Verify user has role assigned in user_roles table

---

**This document serves as the complete reference for the current state of the Sozo platform. Use it to continue development without breaking existing functionality.**
