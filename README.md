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
│  • JWT Authentication   │ JWT  │  • Face Recognition (FaceNet)│
│  • User Management      │─────▶│  • Medicine Detection (YOLO) │
│  • AI Chatbot (Gemini)  │Token │  • Object Detection (YOLO)   │
│  • GPS Tracking         │Verify│  • Model Hot-Reload from S3  │
│  • Prescriptions & Todos│      │  • Real-time Frame Analysis  │
│  • Game Scores          │      │                              │
│  • MRI Scan Analysis    │      │                              │
│  • Admin Dashboard      │      │                              │
├─────────────────────────┤      ├──────────────────────────────┤
│  Port: 5005             │      │  Port: 5000                  │
│  DB: SQL Server (SSMS)  │      │  Storage: AWS S3             │
│  Framework: Flask       │      │  Framework: Flask            │
└─────────────────────────┘      └──────────────────────────────┘
           │                                  │
           ▼                                  ▼
    ┌──────────────┐                ┌──────────────────┐
    │  SQL Server   │                │    AWS S3         │
    │  (Alzaware DB)│                │  (Model Storage)  │
    └──────────────┘                └──────────────────┘
```

---

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
```

### Face-Recognition-Service `.env`

```env
# Must match the SECRET_KEY in Auth-ChatBot-Service
SECRET_KEY=your-secret-key-change-in-production
```

---

## 🛠️ Tech Stack

| Layer | Auth-ChatBot-Service | Face-Recognition-Service |
|:---|:---|:---|
| **Framework** | Flask 3.0 | Flask 3.0 |
| **Database** | SQL Server (SSMS) | AWS S3 (model storage) |
| **ORM** | Flask-SQLAlchemy + Alembic | — |
| **Auth** | PyJWT (HS256) | PyJWT (HS256) — token verification |
| **AI/ML** | Google Gemini (LangChain) | FaceNet, SVM, MTCNN, YOLO |
| **Security** | Flask-Talisman, Flask-Limiter, Pydantic | — |
| **Voice** | SpeechRecognition, gTTS, pydub | — |
| **Vector DB** | ChromaDB + sentence-transformers | — |
| **Cloud** | — | AWS S3 (boto3) |

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

- **Password Hashing**: Passwords are hashed using `passlib` (bcrypt) — never stored in plain text
- **JWT Tokens**: HS256-signed tokens with configurable expiry and password-change invalidation
- **Token Blacklist**: Logout revokes tokens via in-memory blacklist
- **Rate Limiting**: Per-IP rate limiting on all auth endpoints (default: 100/min)
- **Security Headers**: Flask-Talisman enforces CSP, HSTS, and secure cookie settings
- **Input Validation**: Pydantic validates all JSON payloads (returns 422 on schema errors)
- **Cross-Service Auth**: Face-Recognition-Service verifies JWT tokens from Auth-ChatBot-Service

> ⚠️ **Production Recommendations**:
> - Use a strong, unique `SECRET_KEY` (at least 32 random characters)
> - Persist token blacklist in Redis or database
> - Enable HTTPS via reverse proxy (Nginx)
> - Use a WSGI server like Gunicorn instead of Flask dev server

---

## 👥 Authors

Built and maintained by **Mohamed Ashraf** and team.

---

Good luck! 🚀
