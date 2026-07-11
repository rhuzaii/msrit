# MSRIT Attendance Monitor

A full-stack web application for MSRIT faculty to monitor student attendance, identify low-attendance students, and send email alerts — built as a DevSecOps project.

**Live demo:** [msrit-seven.vercel.app](https://msrit-seven.vercel.app)

---

## Architecture

```
Browser
  ↓
Frontend — React + Vite (Vercel)
  ↓
Backend API — FastAPI (Render)
  ↓                    ↓
Neon PostgreSQL    Notification Service — FastAPI (Render)
                         ↓
                    Brevo Email API
```

| Service | Tech | Hosting |
|---|---|---|
| Frontend | React 18 + Vite | Vercel |
| Backend API | FastAPI + SQLAlchemy | Render (free) |
| Scraper | Selenium + Chrome | Local only (college LAN) |
| Notification | FastAPI + Brevo | Render (free) |
| Database | PostgreSQL | Neon (free) |

---

## Features

- Faculty login and registration with JWT authentication
- Attendance dashboard listing all students below 75% threshold
- Email alerts — HTML attendance report to teacher and/or students via Brevo
- Alert log history with filters
- Scraper trigger from the portal (college LAN only)
- Fernet-encrypted portal credential storage

---

## Project Structure

```
msrit/
├── frontend/                    # React + Vite SPA
│   ├── src/
│   │   ├── pages/               # Login, Register, Dashboard, Logs
│   │   ├── components/          # Navbar, TeacherCard, LogsTable
│   │   ├── context/             # AuthContext, ToastContext
│   │   └── api/index.js         # API client
│   └── public/msritlogo.png
│
├── msrit-scraper/               # Backend API + Selenium scraper
│   ├── backend/                 # FastAPI app
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── crud.py
│   │   ├── auth.py
│   │   ├── notify_client.py
│   │   └── routers/             # auth, me, alerts, scraper, students, teachers
│   ├── app/                     # Selenium scraper
│   │   ├── main.py
│   │   ├── scraper.py
│   │   ├── login.py
│   │   └── attendance.py
│   └── requirements.txt
│
├── notification-service/        # Email microservice
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── email_service.py     # Brevo HTTP API
│   │   ├── routes/notify.py
│   │   └── models.py
│   └── requirements.txt
│
└── vercel.json                  # SPA routing config
```

---

## Local Setup

### Prerequisites
- Python 3.11+
- Node.js 18+
- Chrome (for scraper)
- PostgreSQL or Neon account

### 1. Clone the repo

```bash
git clone https://github.com/rhuzaii/msrit.git
cd msrit
```

### 2. Frontend

```bash
cd frontend
npm install
# Create .env.local and set:
# VITE_API_BASE_URL=http://localhost:8000
npm run dev
```

### 3. Backend API

```bash
cd msrit-scraper
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# Copy .env.example to .env and fill in values
uvicorn backend.main:app --host 0.0.0.0 --port 8000
```

### 4. Notification Service

```bash
cd notification-service
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# Set BREVO_API_KEY and ALERT_SENDER in .env
uvicorn app.main:app --host 0.0.0.0 --port 8001
```

### 5. Run the Scraper (college LAN only)

```bash
cd msrit-scraper
source .venv/bin/activate
python -m app.main
```

---

## Environment Variables

### Backend (`msrit-scraper/.env`)

| Variable | Description |
|---|---|
| `DB_HOST` | PostgreSQL host |
| `DB_PORT` | PostgreSQL port (5432) |
| `DB_NAME` | Database name |
| `DB_USER` | Database user |
| `DB_PASSWORD` | Database password |
| `DB_SSL` | Set to `require` for Neon |
| `FERNET_KEY` | Encryption key for portal passwords — generate once, never change |
| `JWT_SECRET_KEY` | Secret for JWT tokens |
| `JWT_EXPIRE_MINUTES` | Token expiry (default: 1440 = 24h) |
| `NOTIFICATION_SERVICE_URL` | URL of the notification service |
| `ATTENDANCE_THRESHOLD` | Low attendance cutoff (default: 75.0) |
| `EMAILS_CSV_PATH` | Path to USN→Email CSV file |

### Notification Service

| Variable | Description |
|---|---|
| `BREVO_API_KEY` | Brevo transactional email API key |
| `ALERT_SENDER` | Verified sender email on Brevo |
| `DB_*` | Same Neon DB credentials as backend |

### Frontend

| Variable | Description |
|---|---|
| `VITE_API_BASE_URL` | Backend URL (e.g. `https://your-app.onrender.com`) |

---

## Deployment

| Service | Platform | Root Dir | Start Command |
|---|---|---|---|
| Frontend | Vercel | `/` | `cd frontend && npm run build` |
| Backend | Render | `msrit-scraper` | `uvicorn backend.main:app --host 0.0.0.0 --port $PORT` |
| Notification | Render | `notification-service` | `uvicorn app.main:app --host 0.0.0.0 --port $PORT` |

> **Note:** The Selenium scraper cannot run on Render — no Chrome installed and `staff.msrit.edu` is college-LAN-only. Run it locally while connected to the college network.

---

## Email Setup (Brevo)

The notification service uses [Brevo](https://brevo.com) (free — 300 emails/day, no domain verification needed):

1. Sign up at brevo.com
2. Go to **Senders & IP → Senders → Add a Sender** — verify your email address
3. Go to **SMTP & API → API Keys** — generate an API key
4. Add `BREVO_API_KEY` and `ALERT_SENDER` to the notification service environment on Render

---

## Generating the Fernet Key

Run once — never regenerate after teachers are added or all stored portal passwords break:

```bash
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

---

## Scraper Note

The scraper uses Selenium to log into `staff.msrit.edu` and pull attendance data. It only works on the college LAN. Run it manually whenever you want fresh data:

```bash
cd msrit-scraper
source .venv/bin/activate
python -m app.main
```

Data is saved to Neon → the Render backend serves it to everyone via the Vercel frontend.

---

## Tech Stack

- **Frontend:** React 18, React Router v6, Vite 5, CSS custom properties
- **Backend:** FastAPI, SQLAlchemy 2, Pydantic v2, python-jose, bcrypt, Fernet
- **Scraper:** Selenium 4, webdriver-manager, Chrome
- **Database:** PostgreSQL via Neon serverless
- **Email:** Brevo transactional API (HTTPS — works on Render free tier)
- **Auth:** JWT Bearer tokens + Fernet-encrypted portal credentials

---

## Security

| What | Protection |
|---|---|
| Portal passwords | Fernet AES-128-CBC encrypted at rest, decrypted only in memory during scrape |
| App passwords | bcrypt hashed |
| API tokens | JWT with expiry |
| DB credentials | Environment variables only, never in code |
| Student emails | gitignored CSV (PII) |
| API responses | `portal_password_encrypted` structurally absent from all schemas |
