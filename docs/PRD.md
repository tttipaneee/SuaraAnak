# Suara Anak — Product Requirements (Dormathon 2026)

Track 1: Predictive Model — "Predict Early, Decide Better". Team: Ravin (backend, ML, integrations)
and Partner (avatar, frontend design, slides, pitch). Build Sat 10 Oct 09:00 → submit Sun 11 Oct 09:00 MYT.
Judging: campus pitch → top 3 per track per campus → live cross-campus final.

## 1. Problem & solution
**Problem.** Investment, love and job scams against Malaysian parents unfold over days: contact →
rapport → trust/small win → money request → pressure. Selangor lost RM987M to online scams in 2025
(highest state). Recovery after transfer is tiny. Existing tools (ScamShield, Scamio, keyword
filters) react when money is already being requested.

**Solution.** Mak does nothing special: Suara Anak watches her chats with **new contacts** as messages
arrive, predicts the probability of a **money request within the next 48 hours** at every point in the
conversation, shows evidence from a scam-pattern library, and when risk rises sends her a pre-recorded,
consented video of her own child with what to do (in a separate "Suara Anak 🛡" chat), while alerting
the child.

**How it runs (hackathon build, all on Telegram, free Bot API).** No API can read a person's WhatsApp
chats, so the live demo uses two Telegram bots:
- **Contact bot** ("Daniel 😊"): stands in for the unknown contact. The presenter types the scammer's
  lines in the laptop's scammer console → the bot delivers them to Mak's phone; Mak's replies come
  back through the bot. Every message is stored and scored live.
- **Suara Anak bot** ("Suara Anak 🛡"): sends alerts + the child's video to Mak, and the warning to Anak.
Mak's phone therefore shows two separate chats: the scam conversation and Suara Anak.
**Production path (pitch only, not built):** Telegram Business chatbot connected to Mak's own account
(non-contacts only), or an Android companion that reads new-contact WhatsApp notifications.

**Proof of prediction.** Lead time vs a keyword baseline that only fires at the request.

**Out of scope.** Real Android app, reading Mak's real WhatsApp/Telegram account, bank integration,
live lip-sync, predicting days-until-request, Mandarin/Tamil (unless everything else is done).

## 2. Tech stack
| Layer | Choice |
|---|---|
| Backend/ML | Python 3.11, pandas, scikit-learn (LR baseline), LightGBM, joblib |
| API | FastAPI + uvicorn, Pydantic |
| Database | SQLite via SQLModel (`suara.db`) |
| Messaging (live demo + alerts) | Telegram Bot API via `httpx` long polling (no webhook, no ngrok): Contact bot (relay) + Suara Anak bot (alerts, `sendVideo`) |
| WhatsApp export parsing (fallback, P1) | `whatstk` (`df_from_whatsapp`) with a thin wrapper |
| LLM | Team's API key — synthetic data generation + explanation wording only, never the risk score |
| Frontend | Vite + React + TypeScript (styling/chart libs chosen with Partner's design) |
| Avatar | Pre-rendered MP4 clips (Grafilab GPU render, or HeyGen/D-ID); fallback photo + voice note; optional TalkingHead (3D, browser) |
| Voice | ElevenLabs or Azure TTS (child's consented voice) |
| Sponsor tools | Kiro (spec-first; keep `.kiro/`), Grafilab (rendering) |
| Demo | 2 phones (Mak, Anak) + laptop (scammer console + dashboard); Telegram Desktop as Mak, or scrcpy / QuickTime mirroring |

## 3. Functional requirements
P0 = must work in demo · P1 = should · P2 = only if ahead.

| ID | Requirement | Pri |
|---|---|---|
| FR1 | Ingest a conversation into ordered messages with day/hour: live Telegram relay (FR17), replay JSON, WhatsApp "Export chat" .txt via whatstk | P0 live + JSON / P1 .txt |
| FR2 | Predict P(money request within next 48h) for each message prefix with a trained model | P0 |
| FR3 | Top 3 reasons per prediction from model features | P0 |
| FR4 | Classify current scam stage (contact / rapport / trust_small_win) | P1 |
| FR5 | Levels: `safe` < 0.40 ≤ `heads_up` < 0.70 ≤ `strong_stop` | P0 |
| FR6 | Mak replay screen: conversation day by day, current level, evidence on flagged messages, avatar clip when a level is crossed, call-child + 997 actions | P0 |
| FR7 | Telegram alerts from the Suara Anak bot, once per level per conversation: Mak gets text + child's clip at first `heads_up` and first `strong_stop`; Anak gets a warning at first `strong_stop`. Never crash if Telegram fails | P0 |
| FR8 | Anak screen: risk over conversation with thresholds, level, reasons, evidence, original messages, call-Mak action | P0 |
| FR9 | Onboarding: consent + clip approval (static mock OK) | P1 |
| FR10 | `evaluate.py`: lead time, recall-before-ask, false alarm rate, ROC/PR-AUC, keyword baseline → `reports/metrics.md` | P0 |
| FR11 | Explanation sentence in BM via LLM (fallback template) | P1 |
| FR12 | Browser upload of WhatsApp export (fallback ingest) | P1 |
| FR13 | Settings: names, Anak chat, language | P2 |
| FR14 | Scam-pattern evidence: similar lines from the library, case count, % of those cases that later asked for money. Explains only; never changes the score | P0 |
| FR15 | Judges' dashboard: metrics from `/api/metrics` + model card | P0 (pitch) |
| FR16 | Presenter controls `/demo`: pick conversation, reset, step, auto-play. Replay mode = offline fallback for the live demo | P0 |
| FR17 | Live relay: scammer console on `/demo` ("Send next line" from `data/demo_script.md` + free text) → Contact bot → Mak's phone; Mak's replies relayed back; both stored and scored on arrival; dashboard updates within ~2 s | P0 |
| FR18 | Stage time compression: `DEMO_SECONDS_PER_DAY` maps elapsed seconds to `day`/`hour` so a ~2-minute script spans ~6 days | P0 |

Non-functional: < 1 s prediction per conversation; demo works offline except Telegram (replay mode covers no-network); no real personal data; secrets in `.env`.

## 4. Data & database
Training data format, model and pattern-library specs live in `.claude/rules/ml.md` (single source of truth).

App database (SQLite):
```
family        id PK, mak_name, anak_name, mak_telegram_chat_id, anak_telegram_chat_id, language, created_at
consent       id PK, family_id FK, anak_agreed BOOL, agreed_at, clips_approved BOOL
avatar_clip   id PK, family_id FK, level ENUM(heads_up, strong_stop, safe), file_path, watermarked BOOL
conversation  id PK, family_id FK, title, source ENUM(replay, upload, telegram), external_chat_id, created_at
message       id PK, conversation_id FK, idx INT, day INT, hour INT, sender ENUM(other, mak), text
prediction    id PK, conversation_id FK, message_idx INT, risk FLOAT, level ENUM, stage TEXT,
              reasons JSON, model_version TEXT, created_at
evidence      id PK, prediction_id FK, message_idx INT, pattern_ids JSON, similar_cases INT,
              pct_followed_by_ask FLOAT, top_match_text, stage
alert         id PK, conversation_id FK, message_idx INT, level ENUM(heads_up, strong_stop),
              recipient ENUM(mak, anak), channel ENUM(telegram), sent_at, status
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
Flow (live): scammer console / Mak's reply → `telegram_bot.py` (long polling, relays + stores message) →
`predictor.score_timeline()` on the prefix → risk/level/reasons → `patterns.match()` adds evidence → saved →
first `heads_up` / `strong_stop` → `alerts.send_alert()` (Suara Anak bot → Mak + Anak) → dashboard polls and renders.
Flow (replay fallback): seeded conversation → same predictor → frontend steps through it.
API contract: see `CLAUDE.md` (shared by frontend and backend).

## 6. Frontend design
**TBD by Partner** (`design/`). Screens needed (function only): Mak chat replay (FR6), Anak alert & risk (FR8),
Onboarding (FR9), Judges' dashboard (FR15), Presenter controls + scammer console (FR16, FR17).
On the phones, the alert UI is the Telegram "Suara Anak 🛡" chat itself (text + video), not a custom screen.

## 7. Milestones — divided by 2
Syncs (10 min): **13:00, 18:00, 22:00, 02:00, 06:00** — demo what works, cut from §9 if behind.

| Time (MYT) | Ravin | Partner |
|---|---|---|
| 09:00–11:00 | Briefing; scaffold, `.env`; create both bots in BotFather, `/start` on both phones, relay echo test; stub `/api/predict` | Briefing; record selfie + voice + consent; first test clip; screen designs; bot names + avatars |
| 11:00–13:00 | `generate.py` + validator; 20 scam + 20 benign → **joint review 13:00** | Finish designs → `design/`; replay screen against stub API |
| 13:00–16:00 | Full dataset; 15 hand-written test convs; features + LR + `evaluate.py` with baseline | Render heads_up / strong_stop / safe clips (watermarked); 15 hand-written test convs |
| 16:00–18:00 | Real `/api/predict` + SQLite + seed; live relay scored on arrival + Telegram alerts with clip | Avatar clip in replay; scammer console; write `data/demo_script.md` |
| **18:00** | **End-to-end live demo on the real model (2 phones)** | |
| 18:00–22:00 | `patterns.py` + evidence; LightGBM vs LR; time compression tuning | Paraphrase 50–100 scam lines for the library; evidence in UI; slides v1 |
| 22:00–02:00 | Final metrics on hand-written set; `/api/metrics`; P1 upload if on track | Judges' dashboard; competitor + ethics slides; real numbers in |
| 02:00–06:00 | Bug fixes; README + model card; **freeze 04:00** | Backup demo video by 04:00; rehearse ×2 |
| 06:00–08:30 | Final run on both phones; submit | Rehearse ×2; final slides; submit |

Sleep: alternate 90-minute naps 02:00–06:00.

## 8. Demo script (live, ~90 s)
Setup: laptop on the projector shows the dashboard + Mak's Telegram (Telegram Desktop or mirrored
phone). Phone 1 = Mak, phone 2 = Anak (display names "Mak"/"Anak", phone numbers hidden, personal
chats archived). Mockup: https://claude.ai/artifact/QKBLhfUd1fZxW53uqvXiqg
1. "Mak just got a message from a stranger. She does nothing special."
2. Presenter sends the investment-scam script line by line; Mak replies on her phone. Risk line climbs.
3. Day 3 profit story → `heads_up`: Suara Anak chat pings with the child's clip.
4. Day 4 app push → `strong_stop`: second clip; Anak's phone buzzes — hold it up.
5. Day 6 the scammer asks for RM5,000: "A keyword filter only fires now. We warned her days earlier."
6. Replay the benign hard negative (family money talk) — stays low.
7. Pitch line: production connects to Mak's own account (Telegram Business / Android companion).
Fallbacks: no network → replay mode on `/demo`; everything fails → backup video.

## 9. Cut list (in order)
1. FR12 upload 2. FR4 stage UI 3. LightGBM (keep LR) 4. FR11 LLM explanation (template)
5. Onboarding 6. Lip-sync (photo + voice) 7. Evidence on Anak screen (keep on Mak's)
8. Free-text box in scammer console (keep "Send next line") 9. Live relay (FR17) → replay mode only

## 10. Ethics (also a slide)
Avatar only of the consenting child · pre-approved protective scripts only · watermarked AI ·
deletable · opt-in, only new-contact chats processed · 30-day retention · PDPA-aware ·
second opinion, not a replacement for NSRC 997.

## 11. References
- Telegram Bot API: https://core.telegram.org/bots/api
- Telegram Business chatbots (production path): https://core.telegram.org/bots/api#businessconnection
- whatstk: https://whatstk.readthedocs.io/
- COVA (synthetic multi-turn elder-scam conversations, 8 categories, attacker/victim agents): https://arxiv.org/pdf/2604.11752
- COVA-X (10,985 conversations, outcome labels): https://arxiv.org/pdf/2606.06879
- TalkingHead (3D avatar, browser lip-sync): https://github.com/met4citizen/TalkingHead
