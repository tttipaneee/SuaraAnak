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
- WhatsApp export parsing: `api/parse_export.py` wraps `whatstk.df_from_whatsapp`; map the
  contact who isn't Mak to `sender="other"`; derive day index from dates.
- Twilio (`api/whatsapp.py`): read creds from `.env`; send the Anak alert once per conversation at the
  first `strong_stop`; log every send in the `alert` table; never crash the request if Twilio fails.
- CORS open to the Vite dev origin only. Serve `assets/avatar/` as static files.
- Every endpoint has a Pydantic response model matching the contract.
- Keep `scripts/run_demo.sh` working: seed DB, start API, start web — one command.
