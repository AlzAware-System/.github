# 🧠 AlzAware — Smart Glasses Backend Platform

> **A comprehensive backend system for smart glasses designed to assist Alzheimer's patients with real-time face recognition, medicine detection, AI chatbot assistance, GPS tracking, and caregiver management.**

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [System Architecture](#-system-architecture)
- [Services Overview](#-services-overview)
- [Cross-Service Authentication](#-cross-service-authentication)
- [Quick Start](#-quick-start)
- [API Endpoints Reference](#-api-endpoints-reference)
- [Environment Configuration](#-environment-configuration)
- [Tech Stack](#-tech-stack)
- [Data Validation & Transformation](#-data-validation--transformation)
- [Project Structure](#-project-structure)
- [Security](#-security)
- [Authors](#-authors)

---

## 🌟 Project Overview

**AlzAware** is a smart glasses system that helps Alzheimer's patients in their daily life through:

| Feature | Description |
|:---|:---|
| 👤 **Face Recognition** | Identifies family members and caregivers in real-time |
| 💊 **Medicine Detection** | Recognizes medicines using YOLO object detection |
| 🤖 **AI Chatbot** | Conversational assistant powered by Google Gemini (text & voice) |
| 📍 **GPS Tracking** | Real-time location tracking with history for caregivers |
| 🔐 **Multi-Role Auth** | JWT authentication for patients, doctors, caregivers, and admins |
| 🧩 **Cognitive Games** | Score tracking for brain-training games |
| 📋 **Prescriptions & Todos** | Medication management and task reminders |
| 🧠 **MRI Scan Analysis** | Brain MRI scan analysis for Alzheimer's detection |

---

## 🏗️ System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Smart Glasses / Mobile App                │
│                     (Flutter / React Native / IoT)               │
└──────────┬──────────────────────────────────┬────────────────────┘
           │                                  │
           │  REST API                        │  REST API
           │  (Auth, Chat, GPS, User Mgmt)    │  (Image Upload, Recognition)
           ▼                                  ▼

┌─────────────────────────┐      ┌──────────────────────────────┐
│  Auth-ChatBot-Service   │      │  Face-Recognition-Service    │
│  ─────────────────────  │      │  ──────────────────────────  │
│  • JWT Authentication   │ JWT  │  • Face Recognition Module   │
│  • User Management      │────▶│ • Medicine Detection Module  │
│  • ChatBot Module       │Token │  • Object Detection Module   │
│  • GPS Tracking         │Verify│  • Model Hot-Reload from S3  │
│  • Prescriptions & Todos│      │  • Real-time Frame Analysis  │
│  • Game Scores          │      │                              │
│  • MRI Scan Management  │      │                              │
│  • Admin Dashboard      │      │                              │
├─────────────────────────┤      ├──────────────────────────────┤
│ Port: 5005              │      │ Port: 5000                   │
│ Framework: Flask        │      │ Framework: Flask             │
└──────────┬──────────────┘      └──────────────┬───────────────┘
           │                                    │
           │                                    │
     ┌─────▼─────┐                       ┌──────▼───────┐
     │ SQL Server│                       │    AWS S3    │
     │AlzAware DB│                       │ Model Storage│
     └───────────┘                       └──────────────┘
           ▲
           │
     ┌─────┴─────┐
     │   Redis   │
     │ Cache &   │
     │    JWT    │
     │ Blacklist │
     └───────────┘
```

> **Note:** Redis is integrated as an in-memory caching layer for GPS location retrieval and JWT token blacklisting, reducing database load and improving response time.

---
## ⚡ Redis Integration

To improve performance and enhance security, Redis was introduced as an in-memory data store within the Auth-ChatBot-Service.

### GPS Location Caching

Originally, every GPS update and retrieval operation interacted directly with the SQL Server database.

After integrating Redis, the latest location for each patient is cached using a dedicated key:

```text
gps:<patient_id>
```

When a new GPS coordinate is received:

```text
GPS Device
    ↓
receive_gps
    ↓
SQL Server
    ↓
Redis Cache
```

When a caregiver requests the patient's latest location:

```text
get_last_location
    ↓
Redis Cache
```

If the location exists in Redis, it is returned immediately without accessing the database.

If the cache entry is missing:

```text
SQL Server
    ↓
Redis Cache
    ↓
Response
```

A TTL (Time-To-Live) of 24 hours is applied to automatically remove outdated location data and prevent cache growth.

### JWT Token Blacklisting

Redis is also used to implement token revocation after logout.

When a user logs out:

```text
JWT Token
    ↓
Logout Endpoint
    ↓
Redis Blacklist
```

The token is stored using:

```text
blacklist:<jwt_token>
```

The remaining token lifetime is used as the Redis TTL value. Once the token naturally expires, Redis automatically removes the blacklist entry.

For every authenticated request:

```text
JWT Validation
    ↓
Redis Blacklist Check
```

If the token exists in the blacklist:

```http
401 Unauthorized
```

is returned immediately.

### Centralized Redis Client

A dedicated Redis client module was implemented:

```text
app/utils/redis_client.py
```

This centralizes connection management and allows all modules to reuse the same Redis configuration.

### Fault Tolerance

Redis operations are wrapped in exception handling blocks.

If Redis becomes unavailable:

* GPS requests automatically fall back to SQL Server.
* Authentication continues using the existing validation workflow.
* Core application functionality remains operational.

This graceful degradation strategy ensures high availability while still benefiting from caching and token blacklisting when Redis is available.

### Benefits

* Faster GPS retrieval operations.
* Reduced database workload.
* Secure JWT revocation mechanism.
* Automatic cleanup using TTL.
* Improved system scalability.
* No changes required on the mobile application side.


## 📦 Services Overview

### 1. Auth-ChatBot-Service (Port 5005)

The core backend that handles all user-facing features:

- **Authentication**: Multi-role JWT auth (Patient, Doctor, Caregiver, Admin)
- **Chatbot**: AI assistant powered by Google Gemini with text & voice support
- **User Management**: Profile CRUD, prescriptions, todos, game scores
- **GPS Tracking**: Real-time location with history
- **Admin Panel**: User management, system logs, analytics overview
- **MRI Scan Analysis**: Brain scan analysis for Alzheimer's detection

📖 **[Full Documentation →](./Auth-ChatBot-Service/README.md)**

### 2. Face-Recognition-Service (Port 5000)

The AI inference engine for real-time visual recognition:

- **Face Recognition**: FaceNet + SVM pipeline with MTCNN face detection
- **Medicine Detection**: YOLO expert model for identifying medicines
- **General Object Detection**: YOLO general model for everyday objects
- **Hot-Reload**: Updates models from AWS S3 without downtime

📖 **[Full Documentation →](./Face-Recognition-Service/README.md)**

---

## 🔐 Cross-Service Authentication

The two services share a **JWT secret key** (`SECRET_KEY`) so that tokens issued by **Auth-ChatBot-Service** can be verified by **Face-Recognition-Service**.

### Authentication Flow

```
1. User logs in via Auth-ChatBot-Service → receives JWT token
2. User sends image to Face-Recognition-Service with the JWT token
3. Face-Recognition-Service verifies the token using the shared SECRET_KEY
4. If valid → processes the image; If invalid → returns 401
```

### Protected Endpoints

| Endpoint | Service | Auth Required |
|:---|:---|:---|
| `POST /api/upload_frame` | Face-Recognition-Service | ✅ Bearer Token |
| `POST /chat/ask` | Auth-ChatBot-Service | ✅ Bearer Token |
| `POST /chat/voice` | Auth-ChatBot-Service | ✅ Bearer Token |
| All `/user/*` endpoints | Auth-ChatBot-Service | ✅ Bearer Token |
| All `/admin/*` endpoints | Auth-ChatBot-Service | ✅ Bearer Token |

### Example Usage

```bash
# Step 1: Login to get a token
curl -X POST http://localhost:5005/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "ahmed@example.com", "password": "secret"}'

# Response: { "token": "eyJhbGciOiJIUzI1NiIs..." }

# Step 2: Use the token to upload a frame for recognition
curl -X POST http://localhost:5000/api/upload_frame \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -F "image=@photo.jpg"
```

> ⚠️ **Important**: The `SECRET_KEY` in both `.env` files **MUST** be identical for cross-service authentication to work.

---

## 🚀 Quick Start

### Prerequisites

- Python 3.11 or later
- ODBC Driver 17 (or 18) for SQL Server
- SQL Server instance (database `Alzaware` created in SSMS)
- AWS credentials (for S3 model storage)

### 1. Setup Auth-ChatBot-Service

```powershell
cd Auth-ChatBot-Service
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt

# Configure .env (see Environment Configuration section)
python run.py
# Server runs on http://localhost:5005
```

### 2. Setup Face-Recognition-Service

```powershell
cd Face-Recognition-Service
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt

# Configure .env with the SAME SECRET_KEY
python app.py
# Server runs on http://localhost:5000
```

### 3. Database Migrations (Auth-ChatBot-Service)

```powershell
cd Auth-ChatBot-Service
flask --app run.py db init
flask --app run.py db migrate -m "Initial tables"
flask --app run.py db upgrade
```

---

## 📡 API Endpoints Reference

### Auth-ChatBot-Service (`http://localhost:5005`)

#### 🔓 Authentication (`/auth`)

| Method | Endpoint | Description |
|:---|:---|:---|
| `POST` | `/auth/register` | Register a new user |
| `POST` | `/auth/register/patient` | Register a patient |
| `POST` | `/auth/register/doctor` | Register a doctor |
| `POST` | `/auth/register/caregiver` | Register a caregiver |
| `POST` | `/auth/login` | Login and get JWT token |
| `POST` | `/auth/logout` | Logout (revoke token) |
| `POST` | `/auth/forgetpassword` | Request password reset email |
| `POST/GET` | `/auth/resetpassword` | Reset password with token |
| `PATCH/POST` | `/auth/updatemypassword` | Update current password |

#### 👤 User Management (`/user`)

| Method | Endpoint | Description |
|:---|:---|:---|
| `GET` | `/user/me` | Get current user profile |
| `PATCH/POST` | `/user/updateme` | Update profile |
| `DELETE/POST` | `/user/deleteme` | Delete account |
| `POST` | `/user/prescriptions` | Add a prescription |
| `GET` | `/user/my-prescriptions` | Get my prescriptions |
| `GET` | `/user/prescriptions/:patient_id` | Get patient's prescriptions |
| `GET` | `/user/my-patients` | Get my patients (doctor/caregiver) |
| `POST` | `/user/games/scores` | Submit game score |
| `GET` | `/user/games/scores/patient/:patient_id` | Get patient game scores |
| `POST` | `/user/device-token` | Register push notification token |
| `POST` | `/user/todos` | Add a to-do item |
| `GET` | `/user/todos/patient/:patient_id` | Get patient to-dos |
| `PATCH` | `/user/todos/:todo_id` | Update a to-do |
| `DELETE` | `/user/todos/:todo_id` | Delete a to-do |

#### 🤖 Chatbot (`/chat`)

| Method | Endpoint | Auth | Description |
|:---|:---|:---|:---|
| `POST` | `/chat/ask` | 🔒 JWT | Send text message to AI chatbot |
| `POST` | `/chat/voice` | 🔒 JWT | Send voice message to AI chatbot |

#### 📍 GPS Tracking (`/api`)

| Method | Endpoint | Description |
|:---|:---|:---|
| `POST` | `/api/gps` | Send GPS coordinates |
| `GET` | `/api/gps/last` | Get last known location |
| `GET` | `/api/gps/history` | Get location history |

#### 🛡️ Admin (`/admin`)

| Method | Endpoint | Description |
|:---|:---|:---|
| `GET` | `/admin/overview` | Dashboard overview |
| `GET` | `/admin/users` | List all users |
| `GET` | `/admin/users/:role` | List users by role |
| `POST` | `/admin/users/:role` | Create a user |
| `PATCH` | `/admin/users/:role/:user_id/email` | Update user email |
| `PATCH` | `/admin/users/:role/:user_id/account-action` | Manage user account |
| `GET` | `/admin/logs` | View system logs |
| `GET` | `/admin/logs/patient-logins` | View patient login logs |
| `GET` | `/admin/logs/new-patients` | View new patient logs |

#### 🧠 MRI Scan (`/scan`)

| Method | Endpoint | Description |
|:---|:---|:---|
| `POST` | `/scan/mri` | Analyze MRI brain scan |

---

### Face-Recognition-Service (`http://localhost:5000`)

| Method | Endpoint | Auth | Description |
|:---|:---|:---|:---|
| `POST` | `/api/upload_frame` | 🔒 JWT | Upload image for recognition |
| `POST` | `/api/set_active_mode` | 🔓 Public | Switch mode (`face` / `object`) |
| `GET` | `/api/get_latest_results` | 🔓 Public | Get latest recognition results |
| `GET` | `/health` | 🔓 Public | Health check (models status) |
| `POST` | `/reload_eng_mo` | 🔓 Public | Hot-reload model from S3 |
| `POST` | `/api/start-retrain` | 🔓 Public | Trigger model retraining |

---

## ⚙️ Environment Configuration

### Auth-ChatBot-Service `.env`

```env
# SQL Server Database
MSSQL_SERVER=localhost\SQLEXPRESS
MSSQL_DB=Alzaware
MSSQL_DRIVER=ODBC Driver 17 for SQL Server
MSSQL_TRUSTED=true

# Flask & JWT
SECRET_KEY=your-secret-key-change-in-production
FLASK_ENV=development
JWT_EXP_MINUTES=60

# Rate Limiting
RATE_LIMIT_PER_HOUR=100 per minute
RATELIMIT_STORAGE_URI=memory://

# Gmail SMTP (for password reset emails)
SMTP_USER=your_gmail@gmail.com
SMTP_PASSWORD=your_gmail_app_password
EMAIL_FROM=your_gmail@gmail.com

# AWS / SNS configuration
AWS_REGION=eu-north-1
AWS_ACCESS_KEY_ID=your_aws_access_key_id
AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key
SNS_PLATFORM_APPLICATION_ARN=your_sns_platform_application_arn

# Redis Configuration
REDIS_HOST=your_redis_host_here
REDIS_PORT=10371
REDIS_USERNAME=default
REDIS_PASSWORD=your_redis_password_here
```

### Face-Recognition-Service `.env`

```env
API_KEY=My-Super-Secret-Key-For-Training
JWT_SECRET=JwtSecretForAuth
SECRET_KEY=another_secure_key_for_flask_internal
```

---

## 🛠️ Tech Stack

| Layer                 | Auth-ChatBot-Service                    | Face-Recognition-Service           |
| :-------------------- | :-------------------------------------- | :--------------------------------- |
| **Framework**         | Flask 3.0                               | Flask 3.0                          |
| **Database**          | SQL Server (SSMS)                       | AWS S3 (Model Storage)             |
| **ORM**               | Flask-SQLAlchemy + Alembic              | —                                  |
| **Authentication**    | PyJWT (HS256)                           | PyJWT (HS256) – Token Verification |
| **Caching**           | Redis (GPS Cache & JWT Blacklist)       | —                                  |
| **AI/ML**             | Google Gemini (LangChain)               | FaceNet, SVM, MTCNN, YOLO          |
| **Security**          | Flask-Talisman, Flask-Limiter, Pydantic | JWT Middleware                     |
| **Voice Processing**  | SpeechRecognition, gTTS, pydub          | —                                  |
| **Vector Database**   | ChromaDB + Sentence Transformers        | —                                  |
| **Cloud Services**    | Redis Cloud                             | AWS S3 (boto3)                     |
| **Deployment**        | Gunicorn, Python 3.11                   | Gunicorn, Python 3.11              |
| **API Communication** | REST APIs                               | REST APIs                          |

---

## 📝 Data Validation & Transformation

This project employs rigorous data validation and transformation pipelines to ensure data integrity, prevent injection attacks, and prepare data for AI models across services.

### Validation (`Pydantic`)
- **Strict Payloads:** All incoming API requests to the Auth-ChatBot-Service are validated against robust `Pydantic v2` schemas (e.g., `RegisterPatientPayload`, `LoginPayload`).
- **Data Integrity:** We use `ConfigDict(extra='forbid')` to reject any undocumented fields, mitigating mass-assignment vulnerabilities.
- **Custom Business Logic:** Complex validation rules (e.g., matching passwords, verifying contact info formats) are enforced using Pydantic's `@model_validator(mode='after')`.

### Serialization & Transformation
- **Object-to-JSON Serialization:** Dedicated mapping functions cleanly transform complex `SQLAlchemy` ORM entities into secure, formatted JSON dictionaries for API responses.
- **JSON-to-Object Deserialization:** Incoming raw JSON payloads are transformed and sanitized into safe Python data types using Pydantic.
- **AI/ML Data Pipelines:**
  - **NLP:** Text queries are transformed into dense vector embeddings using `SentenceTransformer` for RAG and ChromaDB querying.
  - **Computer Vision:** `LabelEncoder` transforms raw string labels (names) into numeric formats for SVM training (`fit_transform`) and inverse-transforms back to names during inference (`inverse_transform`).

---

## 📁 Project Structure

```
project-grad-code/
│
├── README.md                          ← You are here
│
├── Auth-ChatBot-Service/              ← Authentication, Chatbot & User Management
│   ├── app/
│   │   ├── __init__.py                ← Flask app factory & configuration
│   │   ├── controllers/               ← Business logic
│   │   │   ├── auth_controller.py     ← Registration, login, password mgmt
│   │   │   ├── user_controller.py     ← Profile, prescriptions, todos, games
│   │   │   ├── admin_controller.py    ← Admin dashboard & user management
│   │   │   ├── chat_controller.py     ← AI chatbot (text & voice)
│   │   │   ├── gps_controller.py      ← GPS location tracking
│   │   │   └── scan_controller.py     ← MRI scan analysis
│   │   ├── models/                    ← SQLAlchemy data models
│   │   │   ├── patient.py             ← Patient model
│   │   │   ├── doctor.py              ← Doctor model
│   │   │   ├── caregiver.py           ← Caregiver model
│   │   │   ├── admin.py               ← Admin model
│   │   │   ├── prescription.py        ← Prescription model
│   │   │   ├── todo.py                ← To-do model
│   │   │   ├── game_score.py          ← Game score model
│   │   │   ├── location.py            ← GPS location model
│   │   │   └── ...
│   │   ├── routes/                    ← Flask Blueprints (URL → Controller)
│   │   └── utils/                     ← JWT, validation, email, etc.
│   ├── .env                           ← Environment configuration
│   ├── run.py                         ← Entry point
│   └── requirements.txt               ← Python dependencies
│
└── Face-Recognition-Service/          ← AI Inference Engine
    ├── app.py                         ← Entry point & model loading
    ├── controllers/                   ← HTTP request handlers
    │   ├── inference_controller.py    ← Frame processing & prediction
    │   ├── mode_controller.py         ← Face/Object mode switching
    │   └── system_controller.py       ← Health check, reload, retrain
    ├── middleware/                     ← Authentication middleware
    │   └── auth.py                    ← JWT token verification
    ├── routes/                        ← URL routing
    │   ├── api_routes.py              ← REST API endpoints
    │   └── page_routes.py             ← Web UI routes
    ├── services/                      ← AI/ML business logic
    │   ├── model_loader.py            ← S3 model download & loading
    │   ├── prediction_service.py      ← Face & object prediction
    │   ├── model_state.py             ← Global model state
    │   ├── state_store.py             ← Results cache
    │   └── worker_service.py          ← Background analysis worker
    ├── .env                           ← Shared SECRET_KEY
    ├── upload.py                      ← Camera capture → S3 upload script
    └── requirements.txt               ← Python dependencies
```

---

## 🔒 Security

### Baseline Defenses

| Layer | Technology | Scope |
|:---|:---|:---|
| **Password Hashing** | `bcrypt` | Auth-ChatBot-Service |
| **JWT Authentication** | `HS256` + password-bound `pwd_sig` claim | All services |
| **Security Headers** | Flask-Talisman (CSP, HSTS, X-Frame-Options) | Auth-ChatBot-Service |
| **Input Validation** | Pydantic v2 strict schemas | Auth-ChatBot-Service |
| **Rate Limiting** | Flask-Limiter (per-IP, 100 req/min) | Auth-ChatBot-Service |
| **SQL Injection** | SQLAlchemy ORM (parameterized queries) | Auth-ChatBot-Service |
| **API Key Auth** | Static `X-Auth-Key` header | Face-Recognition-Service |
| **Token Revocation** | Redis-based JWT Blacklist | Auth-ChatBot-Service |

---

### 🛡️ Security Audit & Vulnerability Remediation

A comprehensive security audit was conducted across the entire AlzAware backend platform. **7 vulnerabilities** were identified and remediated:

| # | Vulnerability | Severity | OWASP Category | Service(s) Affected | Status |
|:--|:---|:---|:---|:---|:---|
| 1 | Registration Enumeration | 🔴 High | A01 — Broken Access Control | Auth-ChatBot | ✅ Fixed |
| 2 | Werkzeug Debugger / Info Disclosure | 🔴 High | A05 — Security Misconfiguration | Auth-ChatBot | ✅ Fixed |
| 3 | Weak JWT Secret (Token Forgery) | 🔴 Critical | A02 — Cryptographic Failures | All Services | ✅ Fixed |
| 4 | Token Invalidation Failure | 🔴 High | A07 — Auth Failures | Auth-ChatBot | ✅ Fixed |
| 5 | AI Chatbot Prompt Injection | 🟠 Medium | A03 — Injection | Auth-ChatBot | ✅ Fixed |
| 6 | Token Fixation | 🟠 Medium | A07 — Auth Failures | Auth-ChatBot | ✅ Fixed |
| 7 | Account Lockout via Email Hijacking | 🔴 High | A01 — Broken Access Control | Auth-ChatBot | ✅ Fixed |

**Key highlights:**

- **Registration Enumeration:** Registration endpoints now return identical responses (status code, body, timing) regardless of whether an email exists, preventing attackers from harvesting registered accounts.
- **Weak JWT Secret & Safe Rotation:** The weak JWT secret was replaced with a cryptographically secure 64-character hex key across all three services. A `JWT_SECRET_OLD` fallback mechanism ensures existing mobile sessions are not disrupted during rotation.
- **Token Invalidation After Password Change:** Fixed a Python timezone bug that prevented `password_changed_at` timestamps from correctly invalidating old tokens. Password changes now instantly revoke all previously issued JWTs.
- **AI Prompt Injection Defense:** User input is now strictly separated from system instructions in the Gemini API integration using structured multi-turn message arrays, preventing prompt override attacks.
- **Werkzeug Debugger Disabled:** Production environments no longer expose the interactive debugger, preventing stack trace leakage and remote code execution.
- **Token Fixation Protection:** Authentication operations always issue fresh tokens bound to the current password hash, preventing session reuse across authentication boundaries.
- **Email Hijacking Prevention:** Email updates now require ownership verification before applying changes, preventing account identity theft.

📖 **Detailed patch documentation:** See [Auth-ChatBot-Service Security](./Auth-ChatBot-Service/README.md#-security) and [Face-Recognition-Service Security](./Face-Recognition-Service/README.md#-security).

> ⚠️ **Production Checklist**:
> - ✅ Cryptographically secure `JWT_SECRET` (64+ hex characters) across all services
> - ✅ `FLASK_ENV=production` enforced
> - ☐ Enable HTTPS via Nginx reverse proxy
> - ☐ Deploy with Gunicorn / uWSGI (not Flask dev server)
> - ✅ Persist token blacklist in Redis or database
> - ✅ Rotate `JWT_SECRET_OLD` out after transition period

---

## 👥 Authors

Built and maintained by **Mohamed Ashraf** and team.

---

Good luck! 🚀
