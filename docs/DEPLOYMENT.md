# TalkTails Cloud — Deployment & Operations Guide

**Project:** TalkTails (TalkTails-Cloud)  
**Target Environments:** Render Web Services (Backend Compute), Supabase (Database & Storage), Android (Client Binary)  
**Repository:** [https://github.com/Kanani-Shubham/TalkTails-Cloud](https://github.com/Kanani-Shubham/TalkTails-Cloud)

---

## 1. Cloud Infrastructure Overview

TalkTails utilizes a hybrid multi-cloud infrastructure designed for zero-maintenance serverless scalability, low-latency regional database access, and cost efficiency.

```
                              [ GitHub Repository ]
                                        │
                                        ▼ (Auto Deploy on Push)
                             ┌───────────────────────┐
                             │  Render Web Services  │
                             │  (FastAPI Backend)    │
                             └──────────┬────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
          ┌───────────────────┐ ┌───────────────┐ ┌──────────────────┐
          │ Supabase Postgres │ │  Groq Cloud   │ │ Gnani.ai VoiceOS │
          │  & Cloud Storage  │ │ (LLM Gateway) │ │   (STT & TTS)    │
          └───────────────────┘ └───────────────┘ └──────────────────┘
```

---

## 2. Cloud Compute Deployment (Render)

### 2.1 Web Service Configuration
1. Log in to [Render Dashboard](https://dashboard.render.com).
2. Click **New +** → **Web Service**.
3. Connect your GitHub repository: `Kanani-Shubham/TalkTails-Cloud`.
4. Configure the build and runtime parameters:

| Configuration Field | Production Value |
|---|---|
| **Name** | `lingoboop-ai` (or `talktails-backend`) |
| **Region** | Oregon (US West) or Singapore |
| **Branch** | `main` |
| **Root Directory** | `backend` |
| **Runtime** | Python 3 |
| **Build Command** | `pip install -r requirements.txt` |
| **Start Command** | `uvicorn app.main:app --host 0.0.0.0 --port $PORT` |
| **Instance Type** | Free ($0/month) |

### 2.2 Environment Variables on Render
Add the following key-value pairs in the **Environment** tab:

```env
APP_ENV=production
APP_NAME=TalkTails-Backend
DEBUG=false

# Groq Cloud AI Intelligence
GROQ_API_KEY=gsk_your_actual_groq_key_here
GROQ_MODEL=openai/gpt-oss-120b

# Gnani.ai VoiceOS
GNANI_BASE_URL=https://api.vachana.ai
GNANI_WS_URL=wss://api.vachana.ai
GNANI_API_KEY_ID=your_gnani_api_key_id_here
GNANI_STT_MODEL=prisma-v3.0
GNANI_TTS_MODEL=timbre-v3.0
GNANI_TTS_DEFAULT_VOICE=Shivani

# Supabase Integration
SUPABASE_URL=https://kmkvyhtwxhsweznzubrr.supabase.co
SUPABASE_ANON_KEY=your_supabase_anon_public_key
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_private_key
DATABASE_URL=postgresql://postgres:your_password@db.kmkvyhtwxhsweznzubrr.supabase.co:5432/postgres
```

> [!NOTE]
> **Render Free Instance Behavior:** Free tier instances automatically spin down after 15 minutes of inactivity. When a new incoming HTTP request arrives, the instance spins back up automatically, which can introduce a cold-start latency of approximately 50 seconds on the initial request.

---

## 3. Cloud Database & Storage Setup (Supabase)

### 3.1 Project Provisioning
1. Sign in to [Supabase](https://supabase.com/dashboard).
2. Create a new organization and project (e.g. `lingoboom` or `talktails`).
3. Select region: **South Asia (Mumbai) `ap-south-1`** for lowest latency to Indian language users.
4. Set a strong database password and store it securely.

### 3.2 Schema & Table Migrations
Execute the SQL migrations located in `supabase/` or `Database.md` within the Supabase SQL Editor to provision the 12 production tables:
- `profiles`
- `user_preferences`
- `user_languages`
- `conversations`
- `messages`
- `learning_progress`
- `achievements`
- `subscriptions`
- `usage_records`
- `daily_practice_sessions`
- `daily_practice_items`
- `daily_practice_attempts`

### 3.3 Row Level Security (RLS) Configuration
Enable RLS on all tables and apply user isolation policies:
```sql
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile" 
ON profiles FOR SELECT 
USING (auth.uid() = id);

CREATE POLICY "Users can update own profile" 
ON profiles FOR UPDATE 
USING (auth.uid() = id);
```

### 3.4 Cloud Storage Bucket Configuration
1. Open **Storage** in the Supabase Dashboard.
2. Create a new bucket named: `email-assets`.
3. Set bucket access to **Public** (read-only for anonymous clients).
4. Upload email template assets and avatar icons (`pano_welcome.jpg`, `pano_password_reset.jpg`, etc.).

---

## 4. Mobile Client Build & Configuration (Flutter)

### 4.1 Configuration Injection
The Flutter client reads configuration from environment variables injected at build time via Dart defines:

```bash
flutter build apk --release \
  --dart-define=BACKEND_URL=https://lingoboop-ai.onrender.com \
  --dart-define=SUPABASE_URL=https://kmkvyhtwxhsweznzubrr.supabase.co \
  --dart-define=SUPABASE_ANON_KEY=your_supabase_anon_key
```

### 4.2 Android Manifest Permissions
Ensure the following permissions are granted for full companion functionality:
- `android.permission.INTERNET`
- `android.permission.RECORD_AUDIO`
- `android.permission.SYSTEM_ALERT_WINDOW` (Floating overlay permission)
- `android.permission.FOREGROUND_SERVICE`
- `android.permission.FOREGROUND_SERVICE_SPECIAL_USE`

---

## 5. Verification & Health Monitoring

### 5.1 Backend Health Verification
Execute an HTTP GET request to verify the Render deployment:

```bash
curl -i https://lingoboop-ai.onrender.com/health
```

Expected Response:
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "app_name": "LingoBoom-Backend",
    "environment": "production",
    "gnani_provider": {
      "configured": true,
      "base_url": "https://api.vachana.ai",
      "stt_model": "prisma-v3.0",
      "tts_model": "timbre-v3.0",
      "api_key_status": "present"
    },
    "supabase": {
      "configured": true,
      "service_role_configured": true
    }
  }
}
```

### 5.2 Supabase Health Verification
Verify that project health status is **Healthy** under the Supabase Dashboard overview (Project Health, Database, PostgREST, Auth, and Storage).
