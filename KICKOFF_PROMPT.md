# Paste this as the first message of the build session

I'm building **Suara Anak** for Dormathon 2026 (Track 1: Predictive Model). Hard submission
Sun 11 Oct 09:00 MYT. `CLAUDE.md`, `docs/PRD.md` and `.claude/rules/` are already in the repo —
read `CLAUDE.md` and `docs/PRD.md` first.

**Summary:** Scams against Malaysian parents unfold over days. Existing bots react when money is
already requested. Suara Anak predicts **P(money request within the next 48 hours)** at every point
in a forwarded/exported chat, explains it with feature reasons and matches from a scam-pattern
library, and warns Mak with a pre-recorded consented video of her own child, while alerting the
child on WhatsApp. Headline metric: **lead time vs a keyword baseline**.

**Team:** I own backend + ML. My partner owns the avatar and the visual design (`design/`) — don't
invent any UI styling. Work autonomously; report what you did and what's next. Ask only for
irreversible or genuinely ambiguous decisions.

**Do now, in order (commit after each, show me the output):**
1. Scaffold the repo per `CLAUDE.md` (README, `.env.example`, `.gitignore`, `requirements.txt`
   incl. fastapi, uvicorn, sqlmodel, pandas, scikit-learn, lightgbm, joblib, twilio, whatstk;
   Vite React TS app). Add `scripts/run_demo.sh`. Fill in the Commands section of `CLAUDE.md`.
2. Stub `/api/predict` returning the exact contract shape, plus `/api/conversations` with 3 seeded
   demo conversations — so the frontend can start immediately.
3. `ml/generate.py` per `.claude/rules/ml.md`: two prompt templates, hard negatives, validator.
   Generate 20 scam + 20 benign and stop so we can review quality together.
4. After review: full dataset, then `features.py`, `train.py`, `evaluate.py` (prefix labels,
   split by conv_id, keyword baseline, lead time, false alarms → `reports/metrics.md`).
5. Replace the stub with the real model, then add `patterns.py` evidence and the Twilio alert.

Track progress against the milestones in `docs/PRD.md` §7 and warn me early if the 18:00
end-to-end milestone is at risk.
