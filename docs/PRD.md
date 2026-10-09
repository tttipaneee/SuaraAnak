# Suara Anak — Product Requirements (Dormathon 2026)

Track 1: Predictive Model — "Predict Early, Decide Better". Team: Ravin (backend, ML, integrations)
and Partner (avatar, frontend design, slides, pitch). Build Sat 10 Oct 09:00 → submit Sun 11 Oct 09:00 MYT.
Judging: campus pitch → top 3 per track per campus → live cross-campus final.

## 1. Problem & solution
**Problem.** Investment, love and job scams against Malaysian parents unfold over days: contact →
rapport → trust/small win → money request → pressure. Selangor lost RM987M to online scams in 2025
(highest state). Recovery after transfer is tiny. Existing tools (ScamShield, Scamio, keyword
filters) react when money is already being requested.

**Solution.** Mak forwards or exports a chat with a new contact. Suara Anak predicts the probability
of a **money request within the next 48 hours** at every point in the conversation, shows evidence
from a scam-pattern library, and when risk rises plays a pre-recorded, consented video of her own
child with what to do, while alerting the child on WhatsApp.

**Proof of prediction.** Lead time vs a keyword baseline that only fires at the request.

**Out of scope.** Real Android app, bank integration, live lip-sync, predicting days-until-request,
Mandarin/Tamil (unless everything else is done).

## 2. Tech stack
| Layer | Choice |
|---|---|
| Backend/ML | Python 3.11, pandas, scikit-learn (LR baseline), LightGBM, joblib |
| API | FastAPI + uvicorn, Pydantic |
| Database | SQLite via SQLModel (`suara.db`) |
| WhatsApp export parsing | `whatstk` (`df_from_whatsapp`) with a thin wrapper |
| Messaging | Twilio WhatsApp Sandbox (alerts to Anak) |
| LLM | Team's API key — synthetic data generation + explanation wording only, never the risk score |
| Frontend | Vite + React + TypeScript (styling/chart libs chosen with Partner's design) |
| Avatar | Pre-rendered MP4 clips (Grafilab GPU render, or HeyGen/D-ID); fallback photo + voice note; optional TalkingHead (3D, browser) |
| Voice | ElevenLabs or Azure TTS (child's consented voice) |
| Sponsor tools | Kiro (spec-first; keep `.kiro/`), Grafilab (rendering) |
| Demo | ngrok (webhook), scrcpy / QuickTime (phone mirroring) |

## 3. Functional requirements
P0 = must work in demo · P1 = should · P2 = only if ahead.

| ID | Requirement | Pri |
|---|---|---|
| FR1 | Ingest a conversation (replay JSON; WhatsApp "Export chat" .txt via whatstk) into ordered messages with day/hour | P0 JSON / P1 .txt |
| FR2 | Predict P(money request within next 48h) for each message prefix with a trained model | P0 |
| FR3 | Top 3 reasons per prediction from model features | P0 |
| FR4 | Classify current scam stage (contact / rapport / trust_small_win) | P1 |
| FR5 | Levels: `safe` < 0.40 ≤ `heads_up` < 0.70 ≤ `strong_stop` | P0 |
| FR6 | Mak replay screen: conversation day by day, current level, evidence on flagged messages, avatar clip when a level is crossed, call-child + 997 actions | P0 |
| FR7 | WhatsApp alert to Anak once per conversation at first `strong_stop` | P0 |
| FR8 | Anak screen: risk over conversation with thresholds, level, reasons, evidence, original messages, call-Mak action | P0 |
| FR9 | Onboarding: consent + clip approval (static mock OK) | P1 |
| FR10 | `evaluate.py`: lead time, recall-before-ask, false alarm rate, ROC/PR-AUC, keyword baseline → `reports/metrics.md` | P0 |
| FR11 | Explanation sentence in BM via LLM (fallback template) | P1 |
| FR12 | Browser upload of WhatsApp export | P1 |
| FR13 | Settings: names, Anak number, language | P2 |
| FR14 | Scam-pattern evidence: similar lines from the library, case count, % of those cases that later asked for money. Explains only; never changes the score | P0 |
| FR15 | Judges' dashboard: metrics from `/api/metrics` + model card | P0 (pitch) |
| FR16 | Presenter controls `/demo`: pick conversation, reset, step, auto-play | P0 |

Non-functional: < 1 s prediction per conversation; demo works offline except Twilio; no real personal data; secrets in `.env`.

## 4. Data & database
Training data format, model and pattern-library specs live in `.claude/rules/ml.md` (single source of truth).

App database (SQLite):
```
family        id PK, mak_name, anak_name, anak_whatsapp, language, created_at
consent       id PK, family_id FK, anak_agreed BOOL, agreed_at, clips_approved BOOL
avatar_clip   id PK, family_id FK, level ENUM(heads_up, strong_stop, safe), file_path, watermarked BOOL
conversation  id PK, family_id FK, title, source ENUM(replay, upload), created_at
message       id PK, conversation_id FK, idx INT, day INT, hour INT, sender ENUM(other, mak), text
prediction    id PK, conversation_id FK, message_idx INT, risk FLOAT, level ENUM, stage TEXT,
              reasons JSON, model_version TEXT, created_at
evidence      id PK, prediction_id FK, message_idx INT, pattern_ids JSON, similar_cases INT,
              pct_followed_by_ask FLOAT, top_match_text, stage
alert         id PK, conversation_id FK, message_idx INT, channel ENUM(whatsapp), sent_at, status
scam_pattern  id PK, text, scam_type, stage, source ENUM(paraphrased_report, generated),
              case_id TEXT, followed_by_ask BOOL
```
Seed: one demo family + 3 replay conversations (investment scam, love scam, benign hard negative).

## 5. Architecture
```
suara-anak/
├─ CLAUDE.md · README.md · .env.example · .gitignore
├─ .claude/rules/ (ml.md, api.md, web.md)   ← path-scoped rules for Claude
├─ docs/PRD.md                              ← this file
├─ design/                                  ← Partner's screen designs
├─ data/  ml/  api/  web/  assets/avatar/  reports/  scripts/
```
Flow: conversation → `predictor.score_timeline()` → risk/level/reasons per message →
`patterns.match()` adds evidence → saved → first `strong_stop` → `whatsapp.send_alert()` → frontend renders.
API contract: see `CLAUDE.md` (shared by frontend and backend).

## 6. Frontend design
**TBD by Partner** (`design/`). Screens needed (function only): Mak chat replay (FR6), Anak alert & risk (FR8),
Onboarding (FR9), Judges' dashboard (FR15), Presenter controls (FR16).

## 7. Milestones — divided by 2
Syncs (10 min): **13:00, 18:00, 22:00, 02:00, 06:00** — demo what works, cut from §9 if behind.

| Time (MYT) | Ravin | Partner |
|---|---|---|
| 09:00–11:00 | Briefing; scaffold, `.env`, Twilio sandbox echo on both phones; stub `/api/predict` | Briefing; record selfie + voice + consent; first test clip; screen designs |
| 11:00–13:00 | `generate.py` + validator; 20 scam + 20 benign → **joint review 13:00** | Finish designs → `design/`; replay screen against stub API |
| 13:00–16:00 | Full dataset; 15 hand-written test convs; features + LR + `evaluate.py` with baseline | Render heads_up / strong_stop / safe clips (watermarked); 15 hand-written test convs |
| 16:00–18:00 | Real `/api/predict` + SQLite + seed | Avatar clip in replay; Anak screen |
| **18:00** | **End-to-end demo on the real model** | |
| 18:00–22:00 | `patterns.py` + evidence; LightGBM vs LR; WhatsApp alert | Paraphrase 50–100 scam lines for the library; evidence in UI; slides v1 |
| 22:00–02:00 | Final metrics on hand-written set; `/api/metrics`; P1 upload if on track | Judges' dashboard; competitor + ethics slides; real numbers in |
| 02:00–06:00 | Bug fixes; README + model card; **freeze 04:00** | Backup demo video by 04:00; rehearse ×2 |
| 06:00–08:30 | Final run on both phones; submit | Rehearse ×2; final slides; submit |

Sleep: alternate 90-minute naps 02:00–06:00.

## 8. Demo script
Mak's phone (projected) replays the investment scam: Day 1 ~8% → Day 3 ~30% → Day 4 crosses 0.70 →
strong_stop clip plays, evidence shown → Anak's phone buzzes → Day 6 the scammer asks for RM5,000.
Then the benign hard negative (family money talk) stays low. Backup video on laptop.

## 9. Cut list (in order)
1. FR12 upload 2. FR4 stage UI 3. LightGBM (keep LR) 4. FR11 LLM explanation (template)
5. Onboarding 6. Lip-sync (photo + voice) 7. Evidence on Anak screen (keep on Mak's)

## 10. Ethics (also a slide)
Avatar only of the consenting child · pre-approved protective scripts only · watermarked AI ·
deletable · only forwarded chats processed · 30-day retention · PDPA-aware · second opinion, not a replacement for NSRC 997.

## 11. References
- whatstk: https://whatstk.readthedocs.io/
- Twilio WhatsApp + Python: https://www.twilio.com/en-us/blog/send-whatsapp-message-30-seconds-python
- COVA (synthetic multi-turn elder-scam conversations, 8 categories, attacker/victim agents): https://arxiv.org/pdf/2604.11752
- COVA-X (10,985 conversations, outcome labels): https://arxiv.org/pdf/2606.06879
- TalkingHead (3D avatar, browser lip-sync): https://github.com/met4citizen/TalkingHead
