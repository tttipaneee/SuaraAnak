---
paths:
  - "ml/**"
  - "data/**"
  - "reports/**"
  - "api/predictor.py"
  - "api/patterns.py"
---

# ML & data rules — Suara Anak

## Training data (`data/*.jsonl`, one conversation per line)
```json
{"conv_id": "inv_0042", "label": "scam", "scam_type": "investment", "source": "llm_gen_v1",
 "messages": [{"t": 0, "day": 1, "hour": 21, "sender": "other", "text": "Salam kenal...",
               "stage": "contact"}],
 "first_ask_index": 27}
```
- `label` scam|benign · `scam_type` investment|love|job|none · `sender` other|mak
- `stage` (scam only): contact → rapport → trust_small_win → ask → pressure
- `first_ask_index`: first message where the other party asks Mak for money; null if benign
- Files: `train_gen.jsonl`, `val_gen_heldout_template.jsonl`, `test_handwritten.jsonl` (~30, team-written, never trained on)
- ~300 scam + ~300 benign; benign includes hard negatives (family money talk, sellers, couriers,
  colleagues, friend borrowing RM50). BM/English/Manglish, 15–45 msgs over 3–14 days.
- ≥2 prompt templates; hold one out for validation. Validator rejects malformed or too-obvious
  conversations (e.g. money mentioned in the first 3 messages).
- No real names, phone numbers or bank accounts.

### Generation risk
Commercial LLMs may refuse to role-play scammers (the COVA paper hit this). If refused: frame the
request as fraud-prevention training data, seed it with team-paraphrased real case scripts, or
fill templates by hand and use the LLM only to paraphrase. Never report refusals as data.

## Model
- Unit = conversation prefix (messages 0..t). y_t = 1 if scam AND first ask occurs within 48 hours
  after message t (by day/hour). **Drop prefixes at/after `first_ask_index`.**
- Features: flattery/affection, intimacy escalation, money/investment/profit words, platform switch
  (app/Telegram/link), urgency, secrecy, authority claims, unknown contact, days since first contact,
  late-night share, message-rate trend, questions about Mak's finances/family, char 3–5-gram TF-IDF
  on the last N messages.
- Models: Logistic Regression (explainable baseline) and LightGBM; keep the better on the held-out
  template. Reasons: LR coefficients × feature values, or LightGBM `pred_contrib=True`.
- Split by `conv_id` only. Fixed random seeds. Save `ml/models/model.joblib` + `model_card.md`.

## Metrics (`ml/evaluate.py` → `reports/metrics.md`)
1. Lead time (messages + days) from first `heads_up` (≥0.40) to `first_ask_index`
2. Recall-before-ask (% scam convs flagged before any money request)
3. False alarm rate on benign, reported separately for hard negatives
4. ROC-AUC / PR-AUC at prefix level
5. Keyword baseline (fires on money/transfer words) on the same sets
Report generated-val and hand-written-test separately. Numbers on slides come only from this file.

## Scam-pattern library (`ml/patterns.py`)
- Rows: one per scammer message, from (a) team-paraphrased lines from PDRM/NSRC/news
  (`source=paraphrased_report`) and (b) scammer messages in the **training** split (`source=generated`).
  Never from `test_handwritten.jsonl`.
- Each row: text, scam_type, stage, case_id, followed_by_ask.
- Match: char 3–5-gram TF-IDF cosine ≥ 0.45 (tune on val). Return distinct cases matched,
  % later asked for money, most common stage, closest line.
- No match → no evidence. UI wording must say "dalam pangkalan data kami" and show the source mix.
