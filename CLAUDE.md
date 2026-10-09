# Suara Anak — Claude instructions

Dormathon 2026 · Track 1: Predictive Model ("Predict Early, Decide Better").
Team: **Ravin** (backend, ML, integrations) · **Partner** (avatar, frontend design, slides, pitch).
Hard submission: **Sun 11 Oct 2026 09:00 MYT** (aim 08:30).

Full product requirements: `docs/PRD.md` — read it before planning a feature or when scope is unclear.
Module rules load automatically from `.claude/rules/` (ml.md, api.md, web.md).

## What we're building
Mak forwards or exports a chat with a new contact. A trained model predicts **P(money request within
the next 48 hours)** at every point in the conversation, explains why (feature reasons + matches in our
scam-pattern library), and when risk crosses a level, Mak sees a pre-recorded consented video of her
own child saying what to do, while the child gets a WhatsApp alert.

## The core claim — never drift from it
**Prediction, not detection.** Proof = lead time vs a keyword baseline that only fires at the money request.
Say "predict" everywhere; never add a feature that only works after money is mentioned.

## Repo map
```
data/      JSONL conversations (generated + hand-written test)
ml/        generate.py · features.py · train.py · evaluate.py · patterns.py · models/
api/       FastAPI: main.py · db.py · predictor.py · explain.py · whatsapp.py · parse_export.py · routes/
web/       Vite + React + TS (design from Partner in design/)
assets/avatar/   heads_up.mp4 · strong_stop.mp4 · safe.mp4
reports/   metrics.md (only source for numbers on slides)
scripts/   seed_demo.py · run_demo.sh
```

## Commands
Fill in as they're created; keep them working.
- Setup: `[TBD]`
- Generate data: `[TBD]`
- Train / evaluate: `[TBD]`
- Run demo (seed + API + web): `scripts/run_demo.sh`

## API contract (shared by frontend and backend)
- `GET /api/conversations` → `[{id, title, source, message_count}]`
- `GET /api/conversations/{id}` → `{id, title, messages:[{idx, day, hour, sender, text}]}`
- `POST /api/predict` `{conversation_id}` or `{messages:[...]}` →
```json
{"conversation_id": 3,
 "timeline": [{"idx": 0, "day": 1, "risk": 0.08, "level": "safe", "stage": "contact",
               "reasons": ["unknown contact"],
               "evidence": {"matched_text": "...", "similar_cases": 23, "pct_followed_by_ask": 0.87,
                            "stage": "trust_small_win", "scam_type": "investment"}}],
 "first_alert_idx": 18, "first_alert_level": "strong_stop",
 "clip": "/assets/avatar/strong_stop.mp4", "explanation": "...",
 "library": {"paraphrased_reports": 60, "generated_cases": 240}, "model_version": "lgbm_v1"}
```
`evidence` is `null` when nothing matches. Levels: `safe` < 0.40 ≤ `heads_up` < 0.70 ≤ `strong_stop`.
- `POST /api/upload-export` (multipart .txt) → `{conversation_id}` (P1)
- `POST /api/alerts/test` · `GET /api/metrics` · `POST /api/twilio/inbound` (P2)
Changing the contract: update this section and tell both teammates.

## Rules
1. **No leakage.** Train only on prefixes before `first_ask_index`; split by `conv_id`; never train on or build the pattern library from `test_handwritten.jsonl`.
2. **Risk score comes only from the trained model.** The LLM only generates synthetic data and wording.
3. **Never invent numbers or evidence.** Metrics come from `reports/metrics.md`; evidence only from the library; otherwise `[X]` or nothing.
4. **Demo path first.** Keep the end-to-end flow working; improve behind it. Stub predictor until the real one passes evaluation.
5. **Time-box 45 minutes.** If stuck, propose a fallback from the cut list in `docs/PRD.md` §9.
6. **Simple over clever.** No deep learning; justify any new heavy dependency in one line.
7. **One command per task**, documented above as you go.
8. **Secrets only in `.env`** (TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_WHATSAPP_FROM, ANAK_WHATSAPP_TO, LLM_API_KEY). Never commit or print them.
9. **Commit after each working milestone**; never leave `main` broken.
10. **Work autonomously.** Report what was done, what's next, and any risk to the 18:00 end-to-end milestone. Ask only for irreversible or genuinely ambiguous decisions.
11. **Privacy.** No real names, numbers or bank accounts in data, tests, seeds or screenshots.
12. **Verify before claiming done**: run the script/test, show the output.
