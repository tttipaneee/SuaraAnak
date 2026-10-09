---
paths:
  - "web/**"
  - "design/**"
---

# Frontend rules — Suara Anak

- Partner owns the visual design. Follow the files in `design/` exactly; never invent layouts,
  colours, fonts or copy styles. If `design/` is empty, build plain unstyled components that only
  wire data from the API contract (see `CLAUDE.md`), so the design can be applied later without rework.
- Screens and what they must show are listed in `docs/PRD.md` §6 (FR6, FR8, FR9, FR15, FR16).
- Use "predict"/"ramal" wording; never "detect".
- Evidence text must say "dalam pangkalan data kami" and never show numbers the API didn't return.
- Avatar clips come from `assets/avatar/` via the API's `clip` field; show the "AI" watermark tag.
- The presenter route `/demo` must work offline against seeded data (except WhatsApp).
