# SecuriVision: Violation Detection System

AI-powered violation detection system using YOLO. SecuriVision detects **Smoking, Fire, and Fighting** in real time via RTSP or local webcams, logs violations to a PostgreSQL database, saves video clips, and streams live footage and alerts to a web dashboard.

## Features

- Three selectable detection modes: **Smoking, Fire, and Fighting**
- Smoking detection using spatially matched **Person + Cigarette + Smoke** detections
- Dedicated YOLO models for fire/smoke and fighting/violence detection
- Live MJPEG camera stream with bounding box overlay
- Multi-camera support (up to 3 cameras — RTSP or webcam)
- Automatic 15-second video clip recording on violation (5 s pre + 10 s post)
- Violations log with filters and per-violation clip download (shows "No Clip" when recording was interrupted)
- Dashboard with charts (monthly, weekly, 24-hour summary)
- Google OAuth login + local email/password login
- Admin user management: invite by email, deactivate/reactivate, auth provider badges
- Email invite flow — inactive account created → invite link sent via Gmail SMTP → user sets password → account activated
- PostgreSQL storage with IP geolocation tagging

## Quick Start

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Configure environment** create a `.env` file:
   ```
   DB_HOST=localhost
   DB_PORT=5432
   DB_NAME=securivision_db
   DB_USER=postgres
   DB_PASSWORD=yourpassword

   GOOGLE_CLIENT_ID=your_google_oauth_client_id
   GOOGLE_MAPS_API_KEY=your_maps_api_key

   SMTP_EMAIL=you@gmail.com
   SMTP_PASSWORD=your_gmail_app_password

   CAMERA1_SOURCE=rtsp://user:pass@ip:554/stream
   # CAMERA1_SOURCE=webcam   ← use this for local webcam
   ```

   > **Gmail App Password**: Go to myaccount.google.com → Security → 2-Step Verification → App passwords. Do NOT use your real Gmail password.

3. **Run the server:**
   ```bash
   python main.py
   ```

4. **Open the UI:** http://localhost:8000

   API docs: http://localhost:8000/docs

5. **First admin account:** Use the database directly or the `/api/admin/users` endpoint to create the initial admin. All subsequent users are invited via the dashboard.

## Authentication

- **Google Sign-In**: Only users already added by an admin can sign in with Google. Unknown Google accounts are rejected.
- **Email/Password**: Used for accounts created via the invite flow.
- **Invite flow**: Admin adds a user (name + email) → server creates an inactive account and sends an invite email → user clicks the link, sets a password → account is activated.
- Accounts are tied to one provider. Google accounts cannot log in with email/password and vice versa.

## Project Structure

```
├── main.py                  # FastAPI app & all API routes
├── app/
│   ├── detection_service.py # YOLO inference, clip recording, violation logic
│   ├── database.py          # PostgreSQL queries (asyncpg)
│   ├── auth.py              # JWT tokens, Google OAuth verification, invite emails
│   └── models.py            # Pydantic request/response models
├── FYP-UI-REPO/             # Frontend (vanilla JS, Chart.js) — served at /
├── weights/                 # YOLO model weights (not tracked in git)
│   ├── best.pt              # Main smoking model (person + cigarette)
│   ├── 11k.pt               # Smoke model
│   ├── fire.pt              # Fire/smoke model
│   └── fight.pt             # Fighting/violence model
└── clips/                   # Saved violation video clips
```

## Detection Logic

Each camera operates in one selectable detection mode:

### Smoking

1. The main model detects a **Person** (≥ 0.70 confidence) and **Cigarette** (≥ 0.50 confidence) and spatially matches them.
2. The smoke model detects **Smoke** (≥ 0.50 confidence) near the matched person and cigarette.
3. Smoke detections are briefly buffered to bridge gaps between frames.
4. A smoking violation is triggered when **Person + Cigarette + Smoke** are confirmed together and the 20-second cooldown has elapsed.

### Fire

1. The fire model detects fire or dense smoke in the camera feed.
2. A sliding detection window confirms repeated hits to reduce false alerts.
3. A fire violation is triggered after confirmation and a 40-second fire cooldown.

### Fighting

1. The fighting model detects the **violence** class in the camera feed.
2. At least four consecutive detections with ≥ 0.50 confidence confirm the event.
3. A fighting violation is triggered after confirmation and a 20-second cooldown.

For every confirmed violation, SecuriVision saves a 15-second clip to `clips/`, stores its type, location, confidence, and bounding-box data in PostgreSQL, and broadcasts a WebSocket alert to connected dashboard clients.

## Environment Variables Reference

| Variable | Required | Description |
|---|---|---|
| `DB_HOST` | Yes | PostgreSQL host |
| `DB_PORT` | Yes | PostgreSQL port (default 5432) |
| `DB_NAME` | Yes | Database name |
| `DB_USER` | Yes | Database user |
| `DB_PASSWORD` | Yes | Database password |
| `GOOGLE_CLIENT_ID` | Yes | OAuth 2.0 Client ID from Google Cloud Console |
| `GOOGLE_MAPS_API_KEY` | Yes | Maps API key for location display |
| `SMTP_EMAIL` | Yes | Gmail address for sending invite emails |
| `SMTP_PASSWORD` | Yes | Gmail App Password (not your login password) |
| `CAMERA1_SOURCE` | Yes | RTSP URL or `webcam` |
| `CAMERA2_SOURCE` | No | Second camera (RTSP URL or `0`) |
| `CAMERA3_SOURCE` | No | Third camera |
