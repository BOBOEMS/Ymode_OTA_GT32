# Taste
- Prefers a review-first workflow: audit the code thoroughly, then produce a detailed report plus a prioritized issue checklist before touching anything. Confidence: 0.8
- After producing the review report, expects changes to be applied following that checklist — but only when something is genuinely worth changing; if nothing is significantly worth modifying, stop after the report instead of making gratuitous edits. Confidence: 0.8
- Prefers conservative handling of risk: don't unilaterally change frozen/spec-defined or behavior-changing code — flag it in the report with a concrete fix and leave it for an explicit decision. Confidence: 0.7
- Expects review deliverables to be grounded in verification (build + run tests, static analysis) rather than pure code reading. Confidence: 0.6
- Communicates in Chinese and expects responses and written deliverables (reports, documentation) in Chinese. Confidence: 0.7
- Wants detailed, structured deliverables — full issue inventory with severity, file/line references, rationale, and fix/verification notes rather than a short summary. Confidence: 0.75
