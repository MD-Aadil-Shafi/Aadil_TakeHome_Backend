# FastAPI Users Extended - Complete User Management System

A comprehensive user management backend built on FastAPI with advanced features for role-based access control, session tracking, usage analytics, rate limiting, account lockout, and AI-generated user profiles.

## Features

### 1. Role-Based Access Control (RBAC)

- Three user roles: viewer, moderator, admin
- Role hierarchy enforcement
- Protected endpoints based on user role
- Automatic 403 Forbidden response for insufficient permissions

### 2. Session Activity Tracking

- last_login_at — Timestamp of last successful login
- last_logout_at — Timestamp of last logout
- ISO 8601 UTC format for all timestamps
- Useful for audit trails and "last seen" displays

### 3. App-Usage Time Tracking

- total_time_spent_seconds — Cumulative seconds spent in app
- Auto-calculated on logout
- total_time_spent_human — Human-readable format (e.g., "2h 15m 30s")
- Per-user usage statistics via admin endpoint

### 4. Rate Limiting

- Protects auth endpoints from brute-force attacks
- Sliding window algorithm (60-second window)
- Configurable limits per endpoint
- HTTP 429 response with Retry-After header
- Tracks by IP address

### 5. Account Lockout

- Automatic lockout after 3 consecutive failed login attempts
- 10-minute lockout duration (configurable)
- Prevents targeted credential attacks
- Admin unlock capability
- Auto-unlock after duration expires

### 6. AI-Generated User Profiles

- Integrates with Google Gemini API
- Generates professional bios based on profession and experience
- User-editable generated content
- Graceful failure handling

---

## Installation & Setup

### Step 1: Navigate to Project

```
cd Aadil_Solution/examples/sqlalchemy
```

### Step 2: Create Virtual Environment

```
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate  # Windows
```

### Step 3: Install Dependencies

```
pip install -r requirements.txt
```

---

## Environment Variables

```
DATABASE_URL=sqlite+aiosqlite:///./test.db
JWT_SECRET=your-secret-key-here
GEMINI_API_KEY=your-google-gemini-api-key
RATE_LIMIT_LOGIN_RPM=10
RATE_LIMIT_REGISTER_RPM=5
LOCKOUT_THRESHOLD=3
LOCKOUT_DURATION_MINUTES=10
```

---

## Running the Application

### Development Mode

```
cd Aadil_Solution/examples/sqlalchemy => python main.py
OR
uvicorn app.app:app --reload
```

### Access API

- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

---

## API Routes Quick Reference

### Authentication

```
POST   /auth/register              - Register new user
POST   /auth/jwt/login             - Login (with lockout protection)
POST   /auth/jwt/logout            - Logout (with session tracking)
```

### User Profile

```
GET    /users/me                   - Get current user profile
PATCH  /users/me                   - Update own profile
POST   /users/me/generate-about    - Generate AI bio
GET    /admin/users/{id}/stats     - Get user usage stats (admin only)
```

### Role-Based Routes

```
GET    /me/dashboard               - Viewer dashboard (all authenticated users)
GET    /mod/reports                - Moderator reports (moderator+ only)
GET    /admin/stats                - Admin statistics (admin only)
POST   /admin/users/{id}/unlock    - Unlock account (admin only)
```

---

## Testing API Endpoints

### Test Rate Limiting (11+ attempts in 60 seconds)

```
for i in {1..12}; do
  curl -X POST http://localhost:8000/auth/jwt/login \
    -H "Content-Type: application/json" \
    -d '{"username": "test@example.com", "password": "Wrong"}' 2>/dev/null
  sleep 1
done
```

Expected: First 10 return 401, 11th returns 429 Too Many Requests

### Test Account Lockout (3 failed attempts)

```
# Make 3 failed attempts
for i in {1..3}; do
  curl -X POST http://localhost:8000/auth/jwt/login \
    -H "Content-Type: application/json" \
    -d '{"username": "user@example.com", "password": "Wrong"}' 2>/dev/null
done

# 4th attempt should return 403 locked
curl -X POST http://localhost:8000/auth/jwt/login \
  -H "Content-Type: application/json" \
  -d '{"username": "user@example.com", "password": "Wrong"}'
```

### Test Session Tracking (Login, Wait, Logout)

```
# Register
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "session@example.com", "password": "Pass123!"}'

# Login and get token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/jwt/login \
  -H "Content-Type: application/json" \
  -d '{"username": "session@example.com", "password": "Pass123!"}' | jq -r '.access_token')

# Wait 5 seconds
sleep 5

# Logout
curl -X POST http://localhost:8000/auth/jwt/logout \
  -H "Authorization: Bearer $TOKEN"

# Check profile - should show approximately 5 seconds spent
curl -X GET http://localhost:8000/users/me \
  -H "Authorization: Bearer $TOKEN" | jq '.total_time_spent_human'
```

### Test AI Bio Generation

```
# Get token first
TOKEN=$(curl -s -X POST http://localhost:8000/auth/jwt/login \
  -H "Content-Type: application/json" \
  -d '{"username": "user@example.com", "password": "Pass123!"}' | jq -r '.access_token')

# Generate AI bio
curl -X POST http://localhost:8000/users/me/generate-about \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"profession": "Data Scientist", "years_of_experience": 6}'

# Edit bio (manual edit)
curl -X PATCH http://localhost:8000/users/me \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"about": "My custom bio text..."}'
```

---

## Troubleshooting

### GEMINI_API_KEY not found

```
export GEMINI_API_KEY="your-actual-api-key"
```

### Database is locked

```
rm test.db
# Restart server - it will auto-create fresh database
```

### Token expired

Get a new token by logging in again

---

## Database Schema

The User table includes:

- RBAC: role (viewer/moderator/admin)
- Session: last_login_at, last_logout_at
- Usage: total_time_spent_seconds
- Lockout: failed_login_attempts, locked_until
- Profile: profession, years_of_experience, about

---

## Notes

- JWT token lifetime: 3600 seconds (1 hour)
- Rate limit window: 60 seconds (sliding window)
- Account lockout: 10 minutes (configurable)
- Lockout threshold: 3 failed attempts (configurable)
- All timestamps in UTC ISO 8601 format

---
