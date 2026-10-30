# Write-up: AI Proposal Triage (MATACS-TH-2026-02)

## The flow in short
Upload and category choice, then load the admin's criteria version, extract PDF text, scan for hidden instructions, **AI extraction** of facts with quotes, validation, mandatory pass/fail, missing-data check, **scoring formula in code**, ranking, **AI one-line reason**, and finally human review and decision. See `diagram/flow.png`.

## 1. AI vs. rules
**AI is used for two steps only:**
- Reading messy proposal text and turning it into structured facts (install count, price, weeks, warranties, whether a named lead, crew, safety plan or monitoring plan is mentioned).
- Writing a one-line plain-English reason from numbers the code already calculated.

**Plain code does everything else:** the criteria lookup, PDF text extraction, the mandatory pass/fail gate, the budget and 16-week checks, all scoring maths, ranking, tie-breaks and logging.

**Why there:** language is where AI helps, because every bidder words things differently. Pass/fail rules, arithmetic and ranking must be exact and repeatable, and code does them better. The AI never decides who ranks where.

## 2. Consistency and explainability
- Scores come from a fixed formula, so the same extracted facts always give the same score. Examples: price points fall in a straight line from $260,000 (0) to $195,000 (full), timeline from 16 weeks (0) to 8 weeks (full), experience is installs divided by 12 (capped at 1), warranty is 50% panel (cap 25 yrs) plus 50% workmanship (cap 15 yrs), team is four yes/no items.
- Extraction runs at temperature 0 with a pinned model version, and the extracted JSON is saved. A re-rank re-uses the saved JSON unless the reviewer asks for a fresh extraction. So the same proposals ranked twice give the same result. If a fresh extraction runs, small wording changes can alter a value, and the system shows a diff rather than silently changing the rank.
- Every extracted value carries a quote from the proposal. Code checks the quote really appears in the text and rejects it otherwise. This catches made-up facts.
- The reviewer sees, per proposal, each criterion with its extracted value, quote, points and weight, then the total.

## 3. Human in the loop
- **Reviewer checkpoint 1:** all flagged items (hidden instructions, failed validation, unreadable PDFs) and ranked proposals that carry a missing-data warning. A person decides to clarify with the bidder, disqualify, or accept with a stated assumption.
- **Checkpoint 2:** the reviewer can correct any extracted value, with a required reason, and the score recalculates instantly.
- **Checkpoint 3:** ineligible proposals are listed with the failed rule so a person can confirm or contest.
- **Final award is always a human decision.** The tool only produces a recommendation list.
- **Audit log (per run):** user, time, criteria version, model name and version, prompt version, the file hash of each PDF, extracted JSON with quotes, validation results, every score, final ranking, each human edit (who, what, before and after, reason), and the award decision.

## 4. Changing criteria
- Criteria are **versioned**. An admin edit creates a new version and never overwrites the old one. Each scored run stores the version it used.
- Old results stay valid and are labelled "scored under version N". They are not silently changed.
- The system shows a "criteria changed" banner and offers **Re-score**. Because extracted facts are stored, re-scoring only re-runs the formula, which is fast and costs nothing in AI calls. If a new criterion needs facts that were not extracted (for example a new "local presence" criterion), only the extraction for that criterion re-runs.
- Weights that do not add to 100% are blocked by validation.

## 5. Risks and safeguards
| Risk | Safeguard |
|---|---|
| Wrong extraction | Quotes required and checked against the text; schema validation; reviewer sees the evidence and can correct it |
| Made-up facts | Missing facts must be null, not guessed; unsupported values are rejected; missing data lowers the score and is flagged, not filled in |
| Prompt injection or manipulation | Proposal text is wrapped as untrusted data; the prompt forbids obeying it; the AI cannot score or rank; hidden instructions are flagged and held for a person |
| Bias (for example favouring big firms or polished writing) | Scoring is by fixed formula on facts, not on tone; no company name or reputation is scored; the formula is open to audit |
| Cost | Extraction runs once per proposal and is cached; scoring and re-scoring are free code; short prompts; spending cap per run |
| Unreadable or scanned PDFs | Text-length check; OCR fallback; flag if still unreadable |

## Sample set results
- **A (73.5), E (60.9) and D (58.7)** are ranked 1, 2 and 3.
- **D** gives no completion time, so its timeline scores 0 and it carries a flag asking the reviewer to request the schedule. It still ranks because it passes both mandatory checks and has the lowest price. If it confirms about 14 weeks or less it would move to second, so the reviewer should ask before relying on this position.
- **B** is ineligible: the electrical licence is held by an unnamed subcontractor, not the bidder, even though it is the cheapest.
- **C** is flagged: hidden text tries to force a first-place ranking. Ignoring it, its shadow score is about 52.4. A person decides whether to disqualify.

**Assumptions:** the scoring bands above are my choice because the brief gives only the weights; an admin would set them in the admin screens. I treated "since 2021" as inside the 5-year window, and treated C's "since 2020" as partly uncertain.
