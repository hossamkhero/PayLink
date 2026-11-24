# Prompt Log

| Date | Author | Stage | Tool and model | Prompt | Summary of answer | Verdict |
|---|---|---|---|---|---|---|
| 2026-11-23 | Mohamed Khaled | Requirements | DeepSeek V4.1 Flash | "turn FR-1 to FR-28 into user stories. format each as: US-NN, priority, source FR, then given/when/then bullets" | 28 stories in the required format; Given/When/Then used correctly; hard cases embedded in FR-10..13, FR-18. | **Accepted** — this was the base for `requirements.md` §3; I edited only US-08 (modifiability, not AI-generated) and split FR-15 into two stories. |
| 2026-11-23 | Mohamed Khaled | Requirements | DeepSeek V4.1 Flash | "add edge cases for FR-10 idempotency" | Returned one AC: same key twice in sequence → one payment. Missed the in-flight duplicate case and the different-key-same-payload case — both of which are in brief §2.7 HC1. | **Rejected** — AC set was incomplete and would have failed HC1. |
| 2026-11-23 | Mohamed Khaled | Requirements | DeepSeek V4.1 Flash | "rewrite FR-10 AC to cover exactly: (1) same key twice sequential, (2) same key duplicate while first still in flight, (3) different key, same payload. no other cases." | Three ACs matching those three cases; worded close enough to ship after minor grammar cleanup. | **Fixed** — became US-11 AC1, AC2, AC3. |
