---
paths:
  - "api/**"
  - "scripts/**"
---

# Backend rules — Suara Anak

- FastAPI app in `api/main.py`; routers in `api/routes/`; SQLModel tables in `api/db.py` (schema in `docs/PRD.md` §4).
- `api/predictor.py` exposes `score_timeline(messages) -> list[TimelinePoint]`. Until a trained model
  exists, return a **stub** with the exact contract shape (see `CLAUDE.md`) so the frontend is never blocked.
  Switch to the real model only after `evaluate.py` runs successfully.
- Risk score comes only from the trained model. The LLM is used only in `api/explain.py` for wording,
  with a template fallback when the API fails or is slow (> 3 s).
- Telegram (`api/telegram_bot.py`): plain Bot API over `httpx`, long polling `getUpdates` started in the
  FastAPI lifespan only when tokens are set (app must run without them). Contact bot relays console
  lines → Mak and Mak's replies → stored as `sender="mak"`; scammer lines are `sender="other"`. Score the
  prefix on every new message. `/start` replies with the user's chat id (for `.env`). Day/hour come from
  elapsed time scaled by `DEMO_SECONDS_PER_DAY` (unset = real time).
- Alerts (`api/alerts.py`): Suara Anak bot; `sendMessage` + `sendVideo` uploading the clip file from
  `assets/avatar/` (multipart, no public URL). Mak: first `heads_up` and first `strong_stop` per
  conversation; Anak: first `strong_stop`. Log every send in the `alert` table; never crash or block
  scoring if Telegram fails. Alert text uses only model reasons and library evidence (no invented numbers).
- WhatsApp export parsing (P1 fallback): `api/parse_export.py` wraps `whatstk.df_from_whatsapp`; map the
  contact who isn't Mak to `sender="other"`; derive day index from dates.
- CORS open to the Vite dev origin only. Serve `assets/avatar/` as static files.
- Every endpoint has a Pydantic response model matching the contract.
- Keep `scripts/run_demo.sh` working: seed DB, start API, start web — one command.
