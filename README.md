# Inker Robotics Poster Automation Platform

This repository contains the completely finalized, streamlined architecture for the Inker Robotics automated newsletter and poster generator. 

<img width="334" height="458" alt="image" src="https://github.com/user-attachments/assets/7c5a9a85-7d46-4d88-acb9-1fddad3bd5ae" />

---

## 🎯 What It Does

Automatically generates, reviews, and distributes **AI-written newsletters** as high-res WhatsApp/Telegram posters.

**Workflow:**
1. AI agents scout RSS feeds (CrewAI)
2. Writers craft content for multiple audiences (Groq LLM)
3. Together AI generates unique images (FLUX.1)
4. Posters render as 1080×1920 JPEGs via Playwright
5. Manager approves via email link
6. Auto-dispatch to WhatsApp + Telegram

**Key features:**
- ✅ Multi-audience editions (Student & Faculty themes)
- ✅ Dual-theme HTML/JPEG rendering
- ✅ Zero-cost deployment (Render free tier + GitHub Actions)
- ✅ SQLite multi-tenant support
- ✅ Secure approval tokens (webhook-based)
- ✅ Fallback to Telegram (if WhatsApp fails)

---

## 📁 Project Architecture

Single unified FastAPI backend + vanilla HTML dashboard (no heavy frontend framework bloat).

### `backend/` - The Core Engine
* **`main.py`** - FastAPI routes, auth, Web UI serving, job scheduling
* **`engine.py`** - Orchestration: CrewAI pipeline → image gen → HTML rendering → dispatch
* **`.env`** - API keys (Groq, OpenAI, Tavily, Meta, Telegram, Gmail)

### `backend/frontend/` - Web Dashboard
* **`index.html`** - Tailwind CSS agent config UI (RSS feeds, prompts, schedule)
* **`review.html`** - Manager approval + feedback UI
* **`logos/`** - Client branding assets

### `backend/services/` - Pluggable Modules
* **`newsletter_renderer.py`** - Playwright → HTML-to-JPEG screenshot pipeline
* **`whatsapp_meta_service.py`** - Meta Graph API dispatch
* **`telegram_service.py`** - Telegram Bot API fallback

### `backend/core/` - Config & DB
* **`config.py`** - Environment loading
* **`database.py`** - SQLAlchemy + SQLite setup
* **`models.py`** - DayAgentConfig, PipelineExecution, SecureApproval, User
* **`schemas.py`** - Pydantic validation

### `orchestrator.py` - AI Workflow (not shown in files)
* CrewAI agents for scouting + writing
* Multi-audience edition generation
* Feedback injection for regenerations

---

## 🚀 Quick Start

### Local Development
```bash
# Backend (Terminal 1)
cd backend
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your API keys
uvicorn main:app --reload

# Frontend (Terminal 2)
cd frontend
npm install
npm run dev

# Open http://localhost:3000
```

### Manual Test Run
```bash
curl -X POST http://127.0.0.1:8000/api/generate/manual \
  -H "Authorization: Bearer <your_jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{"publish_weekday": 0}'
```

See [DEPLOY.md](./DEPLOY.md) for production setup.

---

## 🔑 Required Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) | LLM for newsletters |
| `OPENAI_API_KEY` | [platform.openai.com](https://platform.openai.com) | Auto-prompt generation + fallback |
| `TAVILY_API_KEY` | [tavily.com](https://tavily.com) | Web search for news |
| `TOGETHER_API_KEY` | [together.ai](https://together.ai) | FLUX.1 image generation |
| `META_WHATSAPP_TOKEN` | Meta Business Settings | WhatsApp dispatch |
| `TELEGRAM_BOT_TOKEN` | @BotFather on Telegram | Telegram fallback |
| `TELEGRAM_CHAT_ID` | IDBot on Telegram | Target Telegram group |
| `JWT_SECRET` | Generate: `openssl rand -hex 32` | Auth token signing |
| `CRON_SECRET` | Generate: `openssl rand -hex 32` | Scheduler webhook protection |
| `BACKEND_URL` | Your Render URL | Callback links in emails |
| `FRONTEND_URL` | Your Vercel URL | Manager review interface |

Copy `.env.example` and fill in your values.

---

## 📅 Scheduling & Publish Days

**Publish days:** Monday, Tuesday, Friday
- Newsletter generation runs **the day before** at scheduled time
- Configuration → edit target time + audiences per day
- GitHub Actions triggers daily; backend filters to only active days

---

## 🔐 Authentication & Multi-Tenancy

- **Email/password** registration and login
- Each user has separate agents + execution history
- SQLite with unique constraint on `(user_id, publish_weekday)`
- JWT tokens valid for 7 days

**Default user (if none exists):**
- Email: `admin@inker.com`
- Password: `password123`
- ⚠️ Change this in production

---

## 📊 Database Schema

| Table | Purpose |
|-------|---------|
| `users` | Email, hashed password |
| `day_agent_configs` | Per-user agent config (prompts, feeds, audience, target time) |
| `pipeline_executions` | Run history + status (pending → processing → awaiting_review → complete/failed) |
| `secure_approvals` | Manager approval tokens + status |

---

## 🔄 Pipeline Flow

```
[Scheduled Time] → [Start Background Task]
    ↓
[Load Day Agent Config] (prompts, RSS feeds)
    ↓
[CrewAI Pipeline]
  ├─ Scout: Search RSS + Tavily for relevant articles
  └─ Writer: Craft 2 editions (Student + Faculty) with feedback
    ↓
[Generate Images] (Together AI FLUX.1 for each section)
    ↓
[Render HTML] (Dual-edition email + poster JPEGs)
    ↓
[Save to DB] + [Generate Approval Token]
    ↓
[Send Manager Email] (with Approve/Edit links)
    ↓
[Await Manager Action]
  ├─ Approve → Set status=complete → Auto-dispatch WhatsApp
  └─ Edit → Inject feedback → Re-run pipeline
    ↓
[Dispatch]
  ├─ Meta Graph API → WhatsApp (primary)
  └─ Telegram Bot API → Telegram group (fallback)
```

---

## 🎨 Rendering & Output Formats

### Email HTML
- Responsive design with Student (dark) + Faculty (light) themes
- Section cards with images, titles, body, source links
- Interactive buttons: Approve / Edit / Share

### WhatsApp Poster
- 1080×1920px JPEG (mobile-optimized)
- Client logo, branded header, section cards
- High-DPI rendering (device_scale_factor=2)
- JPEG compression for < 5MB (Twilio limit)

### Telegram Fallback
- Same JPEG poster
- Formatted caption (bold label, italic subject, bold title, body)
- Link to full article
- 1024-char limit with safe truncation

---

## 🚨 Known Limitations & Future Improvements

### Performance Bottlenecks
- Image generation is sequential (3s delays for rate limiting) → should use async/ThreadPoolExecutor
- Full-page Playwright screenshots → constrain to viewport
- Database deduplication in Python → use SQL `DISTINCT ON`
- Blocking HTTP requests → switch to `httpx` async

### Safety & Scale
- SQLite on disk isn't ideal for multi-user production → migrate to Postgres
- No rate limiting on API endpoints
- No request logging/monitoring
- Email approval tokens don't expire

See [PERFORMANCE.md](./docs/PERFORMANCE.md) for optimization roadmap.

---

## 📝 API Reference

### Agent Management
- `GET /api/agents` - List user's day agents
- `POST /api/agents` - Create/update agent
- `GET /api/agents/{id}` - Get specific agent
- `PUT /api/agents/{id}` - Update agent
- `DELETE /api/agents/{id}` - Delete agent

### Generation
- `POST /api/generate/manual` - Trigger generation for user
- `GET /api/executions` - List user's past runs
- `POST /api/generate/scheduled` - Internal cron trigger (requires `x_cron_secret`)

### Manager Approval
- `GET /api/preview?token=...` - View newsletter before approval
- `POST /api/webhook/regenerate` - Request regeneration with feedback
- `GET /api/webhook/approve?token=...` - Approve for publishing

### Utility
- `GET /health` - System status + next scheduled day
- `GET /api/schedule` - Upcoming publish dates
- `POST /api/auth/register` - Sign up
- `POST /api/auth/login` - Log in
- `GET /api/auth/me` - Current user info

See `backend/main.py` for full endpoint signatures.

---

## 🐳 Docker & Deployment

```bash
# Local Docker
docker compose up --build

# Render.com (see DEPLOY.md)
# Push to GitHub → Render auto-deploys
```

---

## 📚 Additional Docs

- **[DEPLOY.md](./DEPLOY.md)** - Production setup (Render, Vercel, GitHub Actions, Gmail)
- **[.env.example](./.env.example)** - Required environment variables

---

## 🤝 Contributing

1. Create a feature branch
2. Test locally (`npm run dev` + `uvicorn`)
3. Submit PR with clear description

---

## 📄 License

[Add your license here]

---

## ⚡ Support & Troubleshooting

### WhatsApp not sending?
- Check `META_WHATSAPP_TOKEN` is set and valid
- Verify phone number is in Meta sandbox (5 max in dev mode)
- Check execution logs: `GET /api/executions`

### Images aren't generating?
- Verify `TOGETHER_API_KEY` is active and has credits
- Check rate limits (3s delay between images)
- Try manual test: `POST /api/generate/manual`

### Scheduler not triggering?
- Ensure GitHub Actions is enabled
- Check CRON_SECRET matches backend env var
- Verify `BACKEND_URL` is correct and publicly accessible
- Test manually: `curl -X POST ...` with correct header

### Database issues?
- Delete `app.db` to reset (development only)
- Check `backend/core/database.py` for schema
- Verify SQLite write permissions on Render disk

---

## 📞 Questions?

Open an issue on GitHub or check the [Discussions](../../discussions) tab.
``
