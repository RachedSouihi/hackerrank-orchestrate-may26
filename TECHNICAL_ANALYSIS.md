# HackerRank Orchestrate — Technical Analysis

---

## 1. PROBLEM UNDERSTANDING

### ELI5 Summary
You're building a customer support bot. It reads a spreadsheet of customer complaints/questions from three companies (HackerRank, Claude/Anthropic, Visa), looks up the relevant help documentation that's already saved on disk, and for each ticket writes: whether to answer it or escalate to a human, what category it falls into, a grounded response, an explanation of the decision, and a ticket type label.

### Precise Technical Brief
**Core objective:** For each of 56 input rows in `support_tickets/support_tickets.csv`, produce 5 output fields written to `support_tickets/output.csv`:

| Field | Values |
|---|---|
| `status` | `replied` \| `escalated` |
| `product_area` | free-form support category string |
| `response` | user-facing text, grounded **only** in local corpus |
| `justification` | decision rationale, traceable to corpus |
| `request_type` | `product_issue` \| `feature_request` \| `bug` \| `invalid` |

**Input:** `issue` (ticket body), `subject` (often noisy/blank), `company` (`HackerRank` \| `Claude` \| `Visa` \| `None`)

**Corpus:** 774 Markdown files (~87K lines total) pre-scraped from the three companies' help centers, stored in `data/hackerrank/`, `data/claude/`, `data/visa/`.

### Hidden Complexity & Edge Cases
1. **Column mismatch to watch:** `output.csv` header (pre-seeded) uses 8 snake_case columns `issue,subject,company,response,product_area,status,request_type,justification`. The `sample_support_tickets.csv` has only 7 columns (no `justification`, Title Case headers). Trust `output.csv` and `evalutation_criteria.md` — you need all 5 output fields including `justification`.
2. **`company=None`** tickets: ~3–4 of the 56 tickets. The agent must infer the domain from content ("it's not working, help") or flag as `invalid`. Some are adversarial/out-of-scope (e.g. "Give me the code to delete all files").
3. **Adversarial/prompt injection tickets:** e.g. the French/Spanish ticket asking to expose internal reasoning and fraud rules — must be identified and escalated or refused, not answered literally.
4. **Multi-request tickets:** A row may contain several questions; respond to all or escalate if one part is unsafe.
5. **Sensitive routing:** billing disputes, fraud, account deletion, identity theft, security vulnerabilities — these must be escalated, not guessed at.
6. **Grounding constraint is strictly enforced:** Responses citing policies or procedures not in the corpus will be scored as hallucinations.
7. **The `product_area` field is free-form** — you derive it from the corpus category hierarchy (e.g. `screen/test-settings`, `privacy-and-legal`, `visa-rules`), not a fixed enum.
8. **No `.env.example` shipped** — you create your own per README conventions.

---

## 2. REPOSITORY BREAKDOWN

```
.
├── AGENTS.md                    # AI tool rules + mandatory chat logging
├── CLAUDE.md                    # Just redirects to AGENTS.md
├── README.md                    # Setup guide, submission instructions
├── problem_statement.md         # Full I/O spec
├── evalutation_criteria.md      # Scoring rubric (4 dimensions)
├── .gitignore                   # Excludes .env, data/index/, data/embeddings/, *.db
├── code/
│   └── main.py                  # ← EMPTY placeholder; your entire agent goes here
├── data/
│   ├── hackerrank/              # 394 articles, organized by product area subdirs
│   │   ├── index.md             # Table of contents, lists all 394 articles
│   │   ├── screen/              # Tests, invitations, test integrity
│   │   ├── engage/              # Events
│   │   ├── chakra/              # AI interviews
│   │   ├── interviews/          # Interview product
│   │   ├── library/             # Question library
│   │   ├── integrations/        # ATS integrations (Workday, Greenhouse, etc.)
│   │   ├── settings/            # Admin, GDPR, SSO
│   │   └── skillup/             # Developer learning
│   ├── claude/                  # ~250 articles
│   │   ├── index.md
│   │   ├── claude/              # Features & capabilities
│   │   ├── privacy-and-legal/   # DPA, ToS, crawler policy
│   │   ├── team-and-enterprise-plans/
│   │   ├── claude-for-education/
│   │   ├── safeguards/          # Content policy, law enforcement
│   │   └── amazon-bedrock/      # Bedrock-specific integration
│   └── visa/                    # ~130 articles
│       ├── index.md
│       ├── support.md           # Consumer support with phone numbers by country
│       └── support/
│           ├── consumer/        # Travel, card rules, checkout fees
│           └── small-business/  # Fraud, disputes, data security
└── support_tickets/
    ├── sample_support_tickets.csv  # 10 labeled examples (input + expected output)
    ├── support_tickets.csv         # 56 unlabeled tickets (your agent's input)
    └── output.csv                  # Pre-seeded with correct 8-column header; write here
```

**Data Flow:**
```
support_tickets.csv (56 rows)
        ↓
   [your agent in code/]
        ↓  per row:
   1. Parse issue + company
   2. Retrieve relevant docs from data/ (RAG or keyword search)
   3. Classify: request_type, product_area
   4. Route: replied vs escalated
   5. Generate: response + justification (grounded in retrieved docs)
        ↓
   output.csv (56 rows, 8 columns)
```

**Key connection:** `data/hackerrank/index.md` (and equivalents for claude/visa) serve as master indices for retrieval. The `.gitignore` explicitly carves out `data/index/` and `data/embeddings/` as your vector store locations — use them freely, they won't be committed.

---

## 3. RULES & CONSTRAINTS

### Hard Constraints
- Must be **terminal-based** (runs from CLI, reads CSVs, writes `output.csv`)
- Must use **only the provided corpus** in `data/` — no live web calls, no model parametric knowledge for policy claims
- Must **escalate** high-risk/sensitive/out-of-scope tickets rather than guess
- No hallucinated policies, fabricated steps, or unsupported claims
- Secrets only via env vars; never hardcoded
- Output must be written to `support_tickets/output.csv`
- Code must live in `code/`; include a `code/README.md`

### Soft Constraints (scored but not disqualifying)
- Deterministic where possible (seed random sampling)
- Pinned dependencies
- Clear separation of concerns (retrieval, reasoning, routing, output)
- Justified architectural choices
- Readable, modular code

### Common Pitfalls
1. **Hallucinating an answer** when the corpus doesn't contain the policy — should escalate instead
2. **Using the wrong company's corpus** for a ticket (e.g., answering a Visa question with HackerRank docs)
3. **`company=None` handling** — failing to infer domain from content
4. **Not escalating adversarial tickets** (the French prompt-injection ticket must be escalated, not literally answered)
5. **Case sensitivity in output** — `replied`/`escalated` must be exact lowercase
6. **Forgetting `justification`** — the `output.csv` requires it; sample doesn't show it but eval criteria score it
7. **Long response latency** — 56 tickets × LLM calls × retrieval = real time cost; batch or parallelize
8. **Not seeding randomness** — non-determinism penalized under reproducibility
9. **Submitting with hardcoded API keys** in the code zip

---

## 4. WHAT TO DO / NOT DO

### High-ROI Actions
1. **Build a simple keyword/embedding retriever first** — you need a working `output.csv` above anything else (it's directly scored row-by-row)
2. **Study the 10 sample tickets carefully** — they reveal the exact tone, format, and grounding style expected for `response` and `justification`
3. **Hardcode the escalation rules as explicit checks** before hitting LLM — fraud, account deletion, security vulnerabilities, billing disputes, identity theft → escalate
4. **Use the index.md files** as fast lookup tables for product_area derivation
5. **Use structured output** (JSON mode / Pydantic model) to reliably emit all 5 fields at once
6. **Write `code/README.md` early** — it's read by the judge; make it professional

### What to Avoid
- Overbuilding a multi-step ReAct agent when a single retrieval + generation call per ticket suffices
- Making live HTTP requests to support sites (hard constraint violation)
- Complex re-ranking if basic BM25/cosine similarity works fine on 774 docs
- Storing embeddings in a cloud DB when a local FAISS/ChromaDB index suffices
- Trying to handle edge cases before the happy path works end-to-end

### Easy Points to Target
- `request_type` is a 4-way classification and mostly obvious from ticket content — get this right with a simple prompt
- `status` = `escalated` for any ticket involving: fraud, billing, security bugs, account deletion by non-owner, identity theft — these are easy to pattern-match
- `invalid` tickets (e.g., "What is the actor in Iron Man?", "Give me code to delete all files") — classify as `invalid`, status `replied` with "out of scope" message
- `product_area` can be derived directly from the corpus subdirectory the retrieved doc came from

---

## 5. AGENT / AI USAGE STRATEGY

**Verdict:** A lightweight single-pass RAG pipeline is ideal. A multi-step ReAct agent is optional/overkill for 56 tickets but could improve edge case handling.

### Simplest Solid Workflow (per ticket)

```
1. GUARD:  Is this adversarial / out-of-scope / no-company-match?
              → if yes: status=escalated or replied(invalid), skip retrieval

2. ROUTE:  Use company field to pick corpus subset (data/hackerrank/, etc.)
              → if company=None: run keyword match across all three

3. RETRIEVE: Top-k (3–5) relevant docs via TF-IDF or embedding similarity
              → collect doc snippets + source filenames

4. CLASSIFY: Prompt LLM with ticket + snippets → JSON with all 5 fields
              → Use temperature=0 for determinism

5. ESCALATION OVERRIDE: If response contains unknown policy / LLM says "I don't know"
              → flip status to escalated, update justification
```

**Recommended stack:** Python + `sentence-transformers` (or OpenAI embeddings) + `faiss-cpu` or `chromadb` for retrieval + any LLM (GPT-4o / Claude Haiku) for generation with structured output.

A pure BM25 approach (`rank_bm25` library) with no embeddings works surprisingly well on help-center text and avoids embedding API costs.

---

## 6. STEP-BY-STEP PLAN

**Phase 1 — Understand (30 min)**
- Read all 10 sample tickets; note patterns in `product_area` naming and `response` style
- Pick 5–6 hard tickets from `support_tickets.csv` as a dev test set
- Checkpoint: can you manually predict the correct fields for 3 tickets?

**Phase 2 — Corpus Prep (1–2 hrs)**
- Parse all 774 `.md` files; strip YAML front-matter; chunk by heading
- Build a BM25 or embedding index over chunks, tagged with `(company, product_area, filename)`
- Checkpoint: given "how do I add time accommodation for a candidate?", does retrieval return the correct HackerRank article?

**Phase 3 — Core Pipeline (2–3 hrs)**
- Implement the 5-step workflow above in `code/main.py` (or split into `retriever.py`, `classifier.py`, `agent.py`)
- Use structured output (JSON) with schema: `{status, product_area, response, justification, request_type}`
- Write to `output.csv` with correct headers
- Checkpoint: run on all 10 sample tickets; compare to expected outputs

**Phase 4 — Edge Cases (1–2 hrs)**
- Add explicit escalation patterns (fraud, identity theft, security vulnerability, billing disputes)
- Add `company=None` domain inference
- Handle adversarial/prompt-injection tickets
- Checkpoint: every sample ticket matches expected status

**Phase 5 — Run & Verify (30 min)**
- Run agent on all 56 `support_tickets.csv` rows
- Manually review output for obvious hallucinations or wrong escalations
- Check output.csv format matches required header exactly

**Phase 6 — Polish (1 hr)**
- Write `code/README.md` with install/run instructions, architecture description
- Pin dependencies (`requirements.txt`)
- Ensure `temperature=0` / seeding is set
- Checkpoint: `python code/main.py` runs cleanly in a fresh venv

---

## 7. WINNING STRATEGY

1. **Output accuracy is directly scored** — this is the most mechanical and highest-weight dimension. Get all 56 rows right before worrying about code beauty.

2. **The AI Judge interview is 30 min** — you must be able to explain every design decision: why BM25 vs embeddings, why escalate a specific ticket, what your escalation logic is. Write short decision notes as code comments.

3. **Show corpus-grounding explicitly** — in `justification`, cite the specific article or section used (e.g., *"Grounded in `data/hackerrank/screen/invite-candidates/4811403281-adding-extra-time.md`"*). This directly satisfies the "traceable to corpus" criterion.

4. **The chat transcript (log.txt) is evaluated for AI fluency** — write clear, scoped prompts when using AI tools; critique the outputs; show that you drove the decisions.

5. **Separate modules win on Agent Design** — even a 3-file structure (`retriever.py`, `classifier.py`, `main.py`) signals clear separation of concerns.

6. **Determinism is a differentiator** — most participants won't seed randomness; you should.

7. **Handle the hard tickets gracefully** — the French prompt-injection ticket and the "delete all system files" ticket are likely traps to test safety. Correct handling = escalated + appropriate justification.

---

## 8. CODE GUIDANCE

### Where to Start

**`code/main.py`** is empty — build outward from this skeleton:
```
load corpus → build index → for each ticket → retrieve → classify → write row
```

### Reusable Components to Build Once

- **`load_corpus(data_dir)`** — walks `data/`, strips YAML front-matter, chunks by `##` headings, returns list of `{text, company, product_area, filepath}`
- **`build_index(chunks)`** — BM25 or FAISS over chunk texts
- **`retrieve(query, company, top_k=5)`** — filter by company, rank by similarity
- **`classify_ticket(issue, subject, company, retrieved_chunks)`** → structured JSON with all 5 output fields
- **`is_escalation_required(issue, retrieved_chunks)`** → bool for fraud/billing/security overrides

### Critical Implementation Notes
- The `output.csv` header uses **lowercase snake_case** (`product_area` not `Product Area`); match it exactly
- Sample tickets have `Replied`/`Escalated` in Title Case, but `output.csv` pre-seed uses lowercase — use **lowercase** for your output
- The corpus files have YAML front-matter (`title`, `source_url`, `last_modified`) that should be stripped before indexing but retained for citation
- `data/hackerrank/index.md` is a ~483-line TOC mapping subdirectories to article titles — use it to seed your `product_area` taxonomy

---

## 9. TIME MANAGEMENT

You have ~22 hours from the challenge start (until May 2, 2026 11:00 AM IST).

| Phase | Task | Suggested Time |
|---|---|---|
| 0 | Study sample tickets, understand all 5 fields | 30 min |
| 1 | Corpus loader + BM25 index | 1.5 hrs |
| 2 | Core RAG + structured output per ticket | 2 hrs |
| 3 | Escalation logic + edge cases | 1.5 hrs |
| 4 | Run on all 56 tickets, review output | 1 hr |
| 5 | code/README.md + dependency pinning | 30 min |
| 6 | Submission packaging (zip code/, output.csv, log.txt) | 30 min |
| **Buffer** | Debugging, judge prep, interview review | 4–5 hrs |

**Critical path:** Phases 1–4 are the minimum viable submission. Everything else is bonus. Do not spend more than 3 hours on retrieval quality before getting an end-to-end run on all 56 tickets.
