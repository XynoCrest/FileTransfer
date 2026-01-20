# Copilot Coding Agent & Copilot Reviewer Onboarding Instructions

(Trust these instructions first. Only search the repo if information you need is missing, contradicted by errors, or explicitly requested by the user.)

---

## 1. Repository Purpose (High-Level Summary)

This repository (ed-snowflake-datawarehouse) manages the Enterprise Data Warehouse (EDW) “Gold” layer objects in Snowflake: tables, views, sequences, stored procedures, functions, reference/control tables, bridge/fact/dimension tables, and deployment helper scripts. Deployments are orchestrated with the schemachange migration framework, using carefully structured SQL file naming conventions to control ordering and idempotency.

Two categories of objects are managed:
- EDW (core warehouse objects supporting analytics / dimensional model)
- EDO (operational / delta / control and supporting objects)

The key value of this repo is to provide repeatable, governed evolution of Snowflake database structures across environments (feature → project → integration → prod) via branching discipline and scripted deployment.

Primary language: SQL (Snowflake dialect).  
Auxiliary: Markdown (documentation), possibly YAML/JSON for schemachange configuration (fetch if needed for edits).  
There is no application runtime build step in the traditional sense; “build” = validation + structural consistency of SQL artifacts and successful schemachange execution against a Snowflake account.

---

## 2. Branching & Promotion Model (DO NOT DEVIATE)

From the README strategy (essential to avoid rejected PRs):

Project Repos (forks of ED main):
- main (synced via “rehydration” from ED main)
- p-dev (project-level development integration)
- p-qut (project-level pre-integration testing)
- release branches (one per delivery train)
- feature branches (short-lived; branch from the project’s release branch)

ED (this) Repository:
- main (production)
- dev (integration testing – DEV environment)
- qut (UAT integration testing – QUT environment)
- release branches (created from main, aggregate multiple project deliveries before promotion)

Golden Flow (ED repo):
release branch → dev → qut → main (tag release)

ALWAYS ensure:
1. You target the correct branch for the environment stage.
2. You do NOT skip the required merge progression.
3. Rehydration PRs keep downstream forks current (main → project main → release → p-dev / p-qut / feature as needed).

When generating a PR:
- Confirm base branch matches user intent (e.g., adding new object typically targets a release or dev depending on instructions).
- Never merge directly to main unless explicitly instructed and process-compliant.

---

## 3. SQL File Naming & Execution Order (CRITICAL FOR DEPLOYMENT SUCCESS)

schemachange depends on filename patterns:

Prefixes (mutually exclusive by purpose):
- Versioned: `V__` (unique increasing version numbers; usually for one-time migrations). Use ONLY if consistent with existing repo patterns (verify presence of existing V__ files first).
- Repeatable: `R__` (re-executed when file changes; used extensively here for most EDW and EDO objects).
- Always: `A__` (executed every run regardless of changes; use sparingly).

Within Repeatable set, EDW & EDO objects use numeric sub-prefix ranges to enforce ordering (do not collide numbers for different object types unless intentional). From README:

EDW Ranges (examples):
- 000 Sequence
- 005–009 PreDeploymentScript
- 010–019 Reference Table
- 020–029 Dimension Table
- 030–039 Bridge Table
- 040–049 Fact Table
- 050–059 Function
- 060–069 View
- 100–199 Stored Procedure
- 210–219 Data Seed
- 310–319 PostDeploymentScript

EDO Ranges (examples):
- 000 Sequence
- 005–009 PreDeploymentScript
- 010–019 Control Table
- 020–029 Control Table Trigger/Associated Script
- 100–199 Stored Procedure
- 310–319 PostDeploymentScript

File pattern examples (Repeatable):
R__020_EDW_DimCustomer.sql  
R__040_EDW_FactSales.sql  
R__060_EDW_vw_SalesSummary.sql  
R__100_EDW_sp_Load_DimCustomer.sql

ALWAYS:
- Match the correct numeric range.
- Preserve consistent prefix tokens (EDW_/EDO_ + object type).
- Use singular/plural conventions consistent with existing files (inspect similar examples).
- Keep object DDL idempotent where possible (CREATE OR REPLACE etc., when acceptable).

FAILURE MODES TO AVOID:
- Duplicate version numbers (for V__ files) → schemachange error.
- Misordered dependencies (e.g., creating a VIEW before underlying TABLE) → runtime failure.
- Putting seed data before table creation → failure.

---

## 4. Typical Build / Validation Workflow

Because this is schema migration–oriented infrastructure, “build” = lint + structural review + dry-run or execution of schemachange against a non-production environment.

Expected tools (verify via repo contents before first change):
- Python (schemachange is Python-based) OR container wrapper.
- schemachange CLI (`pip install schemachange` or environment-provided).
- Snowflake account credentials (DO NOT create or guess; rely on pipeline secrets).
- A configuration file (commonly `schemachange-config.yml` or similar) specifying change history schema, change history table, and folders.

Recommended local validation sequence (documented for agent; adjust if config differs):
1. (If Python) python -V (expect 3.9+; do not assume, discover if needed)
2. Install dependencies:
   pip install schemachange snowflake-connector-python
3. Export required env vars (never commit secrets):
   export SNOWFLAKE_ACCOUNT=...  
   export SNOWFLAKE_USER=...  
   export SNOWFLAKE_ROLE=...  
   export SNOWFLAKE_WAREHOUSE=...  
   export SNOWFLAKE_DATABASE=...  
4. (Optional) Dry run if supported:
   schemachange -f ./<migrations_root> --modules-folder ./modules --snowflake-account $SNOWFLAKE_ACCOUNT --dry-run
5. Execute:
   schemachange -f ./<migrations_root> --modules-folder ./modules -a $SNOWFLAKE_ACCOUNT -u $SNOWFLAKE_USER -r $SNOWFLAKE_ROLE -w $SNOWFLAKE_WAREHOUSE -d $SNOWFLAKE_DATABASE -c schemachange-config.yml
6. Confirm the change history table updated (auto-managed by schemachange).
7. If adding seeds (DataSeed scripts), re-run to confirm repeatable behavior is stable (no unintended recreation loops).

ALWAYS run schemachange locally (or where permitted) before opening a PR modifying SQL logic. If local execution is not possible due to secured secrets, at minimum perform:
- Dependency ordering check.
- Naming pattern validation.
- Diff review for unintended mass object replacement.

DO NOT:
- Introduce ad hoc SQL outside the managed directory hierarchy.
- Rename existing files arbitrarily (breaks replay logic).
- Change a file’s prefix category (e.g., R__ to V__) without clear migration plan.

---

## 5. Tests & Quality Gates

If the repository later exposes:
- Stored procedure unit tests (SQL or Python harness)
- Linting (SQLFluff or custom)
- GitHub Actions workflows (e.g., .github/workflows/deploy.yml or validate.yml)

Then enforce:
1. Run lint before commit (e.g., sqlfluff lint path).
2. Avoid trailing semicolons inside procedural blocks if style disallows.
3. Keep object definitions minimal and deterministic (NO volatile timestamps in repeatable seeds unless necessary).

If workflows exist, typical triggers:
- Pull Request: validation (syntax + dry-run)
- Merge to dev/qut/main: environment deployment

Replicate only the safe, non-production steps locally (never attempt to deploy to production using secrets that do not belong to a sandbox).

---

## 6. Adding / Modifying Objects (Procedure)

When asked to “add a new dimension table” (example):
1. Pick next available numeric slot in the appropriate range (e.g., 020–029 for dimension).
2. Create R__0NN_EDW_Dim<Subject>.sql with:
   - CREATE OR REPLACE TABLE ...
   - Surrogate key sequence usage (if sequences pattern exists; if sequences are separate, ensure an R__000_... sequence file already exists or add one).
3. If related view: separate file in 060–069 range after table file.
4. If load stored procedure: add in 100–199 range after base objects.
5. If seed data: 210–219 range referencing table created earlier.
6. Validate dependencies: sequences → tables → views → procedures → seeds → post-deploy.
7. Keep each logical artifact in its own file (do not combine multiple object types).

Changing existing stored procedure:
- Maintain signature unless explicitly required.
- Preserve comments / headers.
- If altering logic that other objects depend on, search for references (only then perform targeted repo search).

---

## 7. Common Failure Patterns & Preventive Guidance

Potential errors (anticipate & mitigate):
- “Duplicate version” → rename or increment version number before commit.
- “Object does not exist” in view/procedure creation → ordering wrong; adjust numeric prefix or split dependencies.
- “Permission denied” → environment role misconfiguration; DO NOT attempt to fix by adding GRANTs unless established pattern exists.
- “Repeatable script always re-executing” → file checksum changes due to nondeterministic content (remove volatile constructs).

Always keep migrations deterministic. Do not insert environment-specific logic (e.g., hard-coded database names) if config provides variable substitution.

---

## 8. Repository Layout (Known / Assumed Patterns)

Known from README:
- README.md
- docs/
  - BranchingStrategy.md
  - EDWObjectsCreationGuide.md
  - CodingStandards.md
  - img/ (contains UpdatedBranchingStrategy.png)

Likely (verify before edit; only search if needed):
- A root migrations or scripts directory (examples: /scripts, /sql, /migrations, /snowflake, /deploy)
- schemachange config (schemachange-config.yml or similar)
- Possibly modules/ for shared SQL snippets or procedure helper packages
- .github/workflows/ for CI/CD
- .gitignore

If you must locate where to place a new migration:
1. Identify existing R__/V__/A__ SQL files.
2. Match their directory convention (e.g., /databases/<DB>/<schema>/ if multi-database).
3. Place the new file alongside peers; do not invent new directory levels.

If configuration file declares a “change history table,” do NOT modify that table directly; schemachange manages it.

---

## 9. Style & Coding Standards (Enforce Consistency)

From README references (Coding Standards doc exists):
- Use consistent casing (Snowflake is case-insensitive unless quoted; prefer unquoted UPPER for identifiers unless repository standard differs).
- Use singular vs plural naming consistent with existing dimension / fact naming (e.g., DimCustomer vs DimCustomers—inspect current usage).
- Include controlled comments at top of each file (e.g., purpose, author, date) only if pattern established; otherwise omit to avoid diff churn.
- Avoid trailing whitespace; keep indentation consistent (2 or 4 spaces—match existing).

---

## 10. What To Do If Information Is Missing

Before running an expensive search:
- Re-read these instructions.
- Infer placement from existing numeric ranges and prefixes.
- Only then, perform a focused search (e.g., list files in migrations folder) if you need an exact filename or example.

Do NOT perform broad, repeated repository-wide searches unless essential to complete the requested change.

---

## 11. Safe Operating Principles (Golden Rules)

ALWAYS:
- Respect numeric ordering and prefix semantics.
- Keep each logical object isolated in its own migration file.
- Validate object dependency chain manually before proposing PR.
- Use existing naming conventions exactly.

NEVER:
- Introduce arbitrary shell tooling changes.
- Rotate or purge historical migration files.
- Hardcode secrets or credentials.
- Collapse multiple unrelated changes into a single migration file.

---

## 12. PR Preparation Checklist (Use Before Creating a PR)

- [ ] Correct branch target (release/dev as appropriate).
- [ ] File naming follows prefix + numeric + domain pattern.
- [ ] No duplicate numbers in the same object type range.
- [ ] Dependencies appear earlier (lower numeric) than dependents.
- [ ] No accidental modifications to unrelated files.
- [ ] New logic deterministic (no volatile timestamps or random seeds unless intentional).
- [ ] (If accessible) Local schemachange run succeeds with 0 errors.
- [ ] README or docs updated ONLY if adding new structural conventions.

---

## 13. ACRE Validation Checklist (MANDATORY FOR AGENT & REVIEWER)

**Purpose:**  
The following checklist reflects critical validation items derived from the Automated Code Review Engine (ACRE) used in related data projects. Copilot Coding Agent and Copilot Reviewer must assess each PR for these items.  
For each item:  
- Indicate one: `Pass`, `Review`, `Fail`, or `N/A` (Not Applicable).
- If `Review`, `Fail`, or `N/A`, provide a concrete reason and cite relevant lines, filenames, or config references.
- Where possible, include code or naming examples.

**Legend:**  
- **Fail:** Requires immediate review and changes.  
- **Review:** Requires review and potential changes.  
- **Pass:** No review is required.  
- **N/A:** Item does not apply in this PR (explain why).

**Explicit Output Requirement:**  
After assessing all items, produce a summary table in your review as follows:

| Validation Item               | Status  | File & Line(s)      | Notes/Remediation             |
|-------------------------------|---------|---------------------|-------------------------------|
| Hard Coded Print              | Pass    | N/A                 | No print/debug statements     |
| Select Star Statements        | Review  | R__020_file.sql:21  | Use explicit column names     |
| Import Star Statements        | N/A     | -                   | No Python code in this PR     |
| ...                           | ...     | ...                 | ...                           |

If an item is N/A, state the reason (e.g., “No Python files modified in this PR”).

### 13.1 Validation Items & Examples

1. **Hard Coded Print**
   - *Action*: Review print and display/debug statements present in production code or deployed SQL scripts.
   - *Example (Review)*: `PRINT 'Debug: entering procedure';` in a production migration or stored procedure.

2. **Select Star Statements**
   - *Action*: Remove unqualified `SELECT *` statements in production code; always specify columns explicitly.
   - *Example (Review)*: `SELECT * FROM EDW_DimCustomer;`
   - *Example (Pass)*: `SELECT CustomerID, CustomerName FROM EDW_DimCustomer;`

3. **Import Star Statements** (Python or procedural code only)
   - *Action*: Remove `import *` statements unless explicitly required and justified.
   - *Example (Review)*: `from mymodule import *`
   - *Example (Pass)*: `from mymodule import load_customers, transform_data`
   - *N/A Example*: No Python files modified in PR.

4. **Truncation Details**
   - *Action*: Review use of `TRUNCATE TABLE` in scripts. If present, ensure it's intentional (full load) and documented.
   - *Example (Review)*: `TRUNCATE TABLE EDW_FactSales;` with no comment or context.
   - *Example (Pass)*: `-- Full reload required for project X\nTRUNCATE TABLE EDW_FactSales;`

5. **Incorrect File Name**
   - *Action*: SQL file must follow established naming conventions (see Section 3 above).
   - *Example (Fail)*: `my_script.sql` instead of `R__020_EDW_DimCustomer.sql`

6. **Incorrect File Name (py)**
   - *Action*: If a SQL file contains a stored procedure implemented in Python, filename must end with `_py.sql`.
   - *Example (Fail)*: `R__100_EDW_sp_Load_DimCustomer.sql` with embedded Python.
   - *Example (Pass)*: `R__100_EDW_sp_Load_DimCustomer_py.sql`
   - *N/A Example*: No Python-based stored procedures in PR.

7. **Incorrect Time Stamp**
   - *Action*: Review use of `TIMESTAMP_LTZ` or hardcoded datetime values like `2999-12-31 23:59:59`. These may indicate placeholder or non-production logic.
   - *Example (Review)*: `WHERE ExpiryDate = '2999-12-31 23:59:59'`
   - *Example (Pass)*: `WHERE ExpiryDate = :max_expiry_date`

8. **Configuration Settings**
   - *Action*: Review for `ALTER SESSION` statements; these should not appear in object creation scripts except in tables/views contexts as documented.
   - *Example (Review)*: `ALTER SESSION SET TIMEZONE = 'UTC';` in a migration script.

9. **Folders Discrepancies in Repo**
   - *Action*: Exclude unnecessary folders from PRs; only include intended deployment code.
   - *Example (Review)*: Committing `/test/`, `/legacy/`, or `/notebooks/` directories not used for deployment.

10. **Incorrect Environment Emails**
    - *Action*: Remove hardcoded personal or non-organizational emails from source, config, or deployment scripts.
    - *Example (Fail)*: `notify_email = 'user@gmail.com'`
    - *Example (Pass)*: `notify_email = 'ed-data@suncor.com'`

11. **Unmatch Filename/Object**
    - *Action*: SQL filename should match the object it creates or alters (table/view/procedure).
    - *Example (Review)*: `R__020_EDW_DimProduct.sql` creates `EDW_DimCustomer`.

12. **Hardcoded Database**
    - *Action*: Flag hardcoded database references; use config variable substitution if possible.
    - *Example (Review)*: `FROM PROD_EDW.EDW_DimCustomer`
    - *Example (Pass)*: `FROM {{ database }}.EDW_DimCustomer`

13. **Invalid Table DDL**
    - *Action*: Table DDL scripts should use `CREATE OR ALTER`, not `CREATE OR REPLACE`.
    - *Example (Fail)*: `CREATE OR REPLACE TABLE ...`
    - *Example (Pass)*: `CREATE OR ALTER TABLE ...`

14. **Invalid Block Comment**
    - *Action*: Block comments (`/** ... */`) must appear at the beginning of code blocks (within first 4 lines).
    - *Example (Pass)*:  
      ```sql
      /**
       * Load new customers to dimension
       */
      CREATE OR REPLACE TABLE ...
      ```
    - *Example (Fail)*: Block comment mid-file or after code.

15. **Incorrect Reserved Keywords**
    - *Action*: Use uppercase for reserved SQL keywords.
    - *Example (Review)*: `select id, name from customers`
    - *Example (Pass)*: `SELECT id, name FROM customers`

16. **Incorrect Indentation**
    - *Action*: Use spaces (not tabs), 4 spaces per indentation; align keywords in subqueries.
    - *Example (Review)*:
      ```sql
      SELECT id
        FROM customers
       WHERE ...
      ```
    - *Example (Pass)*:
      ```sql
      SELECT id
          FROM customers
          WHERE ...
      ```

---

**Reviewer must always produce the above summary table as part of their review output, with explicit Pass/Review/Fail/N/A status for each item, and explanatory notes for all Review, Fail, or N/A statuses.**

### 13.2 Reviewer Scoring Guidance

Copilot Reviewer MUST assign both a **Sustainability Score** and a **Performance Score** (each out of 10, where 10 is best) for every review.  
- **Sustainability Score (out of 10):**  
  Reflects maintainability, clarity, code reuse, adherence to repo patterns, and ease of long-term support.  
- **Performance Score (out of 10):**  
  Reflects expected runtime efficiency, scalability, query optimization, and avoidance of known performance anti-patterns.

**Include these scores in your review output summary as follows:**

> **Sustainability Score:** X/10  
> **Performance Score:** Y/10

Briefly justify each score (1–2 sentences) referencing the code, patterns, or concerns observed.

**Example:**
> Sustainability Score: 8/10 – Follows repo conventions and clear structure, but some inline comments missing in complex transformation logic.  
> Performance Score: 9/10 – Efficient set-based SQL with no SELECT *, but could optimize join predicates further for large fact tables.

## 14. Copilot Reviewer Guidance (Apply on Every Pull Request)

This section augments the above for GitHub Copilot Reviewer. The Reviewer MUST systematically assess each PR across: (1) Risk of Failure, (2) Performance Risk, (3) Sustainability, (4) Conformance to Standard Development Patterns, (5) Architectural & Data Model Change Detection.

### 14.1 Review Output Structure (Mandatory Template)

Reviewer MUST produce a structured summary using this ordered headings (omit a heading only if truly not applicable):

1. Summary
   - **Sustainability Score:** X/10
   - **Performance Score:** Y/10
2. Scope Classification  
   - Object Types Touched (Sequences / Tables / Dimensions / Facts / Bridges / Views / Stored Procedures / Functions / Data Seeds / Control Tables / Scripts)  
   - Layers Impacted (Silver / Gold / Control / Orchestration)  
3. Architectural Change Indicators  
4. Data Model Change Analysis (Facts & Dimensions)  
5. Risk of Failure (High/Medium/Low + rationale)  
6. Performance Risk (SQL + Orchestration)  
7. Sustainability & Maintainability  
8. Pattern Compliance & Naming / Ordering Validation  
9. Dependency & Sequencing Validation  
10. Idempotency & Repeatability Assessment  
11. Comment & Documentation Adequacy  
12. Detected Anti-Patterns / Smells  
13. Recommended Remediations (prioritized)  
14. Green / Conditional / Block Recommendation (state one and explain) 

### 14.2 Risk of Failure – Evaluation Criteria

Flag High risk if any of:
- View/procedure references objects created later or absent.
- Fact/Dimension table structural change without coordinated dependent view/procedure updates.
- Removal or change of surrogate key strategy (sequence vs identity) without full downstream impact analysis.
- Introduction of non-deterministic constructs in repeatable scripts (CURRENT_TIMESTAMP, RANDOM()) affecting re-run stability.
- Misuse of prefix category (e.g., changing R__ to V__ arbitrarily).
- Seed scripts referencing tables not yet created (numeric ordering violation).
- Breaking change (column type narrowing, column drop, renaming) without migration or compatibility handling.

Medium risk examples:
- Adding large nullable variant columns without compression or justification.
- New stored procedure logic with complex dynamic SQL lacking defensive error handling or transaction control where needed.

Low risk:
- Purely additive columns with backfill logic handled.
- New views selecting stable underlying objects with deterministic logic.

### 14.3 Performance Risk (SQL Statement & Orchestration)

Assess:
- Large FACT/DIM build queries: presence of unnecessary SELECT * (recommend explicit column list).
- Unfiltered wide joins or CROSS JOIN without reduction predicates.
- Repeated complex subquery or window expressions (recommend CTE / temp table / materialization if large).
- Explosive cardinality due to missing join conditions (possible cartesian).
- Use of DISTINCT as de-duplication band-aid (investigate upstream grain).
- Window functions with excessively broad PARTITION BY (risk: skew / memory).
- Lack of incremental load guard (e.g., full-scan rebuild where incremental pattern established).
- Frequent CREATE OR REPLACE on large views causing downstream invalidation (assess if stable).
- Stored procedure loops performing row-by-row operations instead of set-based DML.
- Data seeds inserting high volumes repeatedly (should be static / MERGE pattern).

Snowflake-specific heuristics (flag if applicable):
- Missing clustering (only if existing repo patterns employ clustering for large partition-pruned tables).
- Overuse of unnecessary MATERIALIZED VIEWS (if added) or misuse of transient vs permanent objects.
- Potential micro-partition pruning loss (broad functions on partitioning/filter columns).

### 14.4 Sustainability & Maintainability

Check:
- Consistent naming aligned with existing EDW/EDO patterns (EDW_Dim*, EDW_Fact*, EDW_vw_*, EDW_sp_*).
- Logical separation: one object per migration file (no bundling).
- Reasonable in-line comments for:
  - Complex transformation logic
  - Business rule filters
  - Non-obvious surrogate key / dedup strategies
  - Performance-critical joins or incremental logic
- Avoid deeply nested procedural branching when SQL set-based refactor feasible.
- Stored procedures:
  - Parameter validation or at least predictable assumptions.
  - Minimal dynamic SQL; if used, sanitized and commented.
- Reuse of helper modules if a pattern exists (do not re-implement utilities).

### 14.5 Pattern Compliance & Ordering

Reviewer MUST verify:
- Numeric prefix correct for object type range.
- No collisions (same R__ number reused incorrectly for a different semantic object when pattern forbids).
- Dependency chain ordering: sequences < tables < bridges/facts/dimensions < views < procedures/functions < seeds < post-deploy.
- No unsanctioned introduction of new numeric ranges without doc update.

### 14.6 Data Model Change Detection (Facts & Dimensions)

Identify and classify:
- Additive column (non-breaking) – note if surrogate / natural key relation changes.
- Column removal / rename – flag High unless compatibility shim present.
- Change in grain (e.g., new key added making table more granular or removing key making it more aggregated).
- Surrogate key generation change (sequence to HASH, etc.).
- Fact table measure recalculation logic changes (impact to historical consistency).
- Dimension SCD handling change (Type 1 vs Type 2 pattern alteration).
Provide explicit before/after reasoning if diff context available.

### 14.7 Architectural Change Indicators

Flag if PR introduces or implies:
- New source system ingestion staging references (new schema naming conventions or external stage usage).
- Addition of new Silver-layer transformation pattern feeding a Gold object.
- New cross-system integration (joins across previously unrelated domains).
- Introduction of orchestration processes (tasks/procedures) altering load cadence.

If detected: state required follow-up (architecture review, lineage update, documentation addition).

### 14.8 Idempotency & Repeatability

For R__ scripts:
- Ensure deterministic CREATE OR REPLACE (no volatile tokens).
- Data seeds: prefer MERGE pattern over INSERT-only if re-runnable.
- No embedded timestamps or session-specific identifiers impacting checksum parity (unless approved pattern).
For V__ scripts:
- Ensure one-time migration rationale (not misused for repeatable objects).

### 14.9 Comment & Documentation Adequacy

Minimum expectation:
- Complex stored procedure: header block (purpose, high-level flow) if established pattern exists.
- Non-trivial transformation: inline comments explaining business filters (e.g., excluding test customers).
- Performance-sensitive code: comment rationale for chosen approach (e.g., use of analytic function vs join).
If missing where complexity > moderate → recommend adding.

### 14.10 Anti-Patterns / Smells to Flag

List any of:
- SELECT * in persistent DDL or heavy transformations.
- Use of DISTINCT to mask duplicate join logic.
- Repeated scalar subqueries instead of join or CTE.
- Duplicated logic across multiple stored procedures (candidate for consolidation).
- Hard-coded database/schema names when substitution variables pattern is used elsewhere.
- DDL or DML outside intended directories (rogue script).
- Uncontrolled exception swallowing in stored procedure logic (losing error context).
- Using CURRENT_TIMESTAMP / RANDOM in repeatable seeds (forces perpetual re-execution).
- Unnecessary DROP followed by CREATE where CREATE OR REPLACE is safer.

### 14.11 Remediation Prioritization

Provide prioritized, actionable recommendations:
1. Must Fix Before Merge (breakage, ordering, high perf risk)
2. Should Fix (sustainability, maintainability, medium perf risk)
3. Could Improve (style / readability not blocking)

### 14.12 Recommendation

End with one:
- APPROVE (No material risks)
- APPROVE WITH CONDITIONS (List Must Fix items explicitly)
- REQUEST CHANGES (Blocking issues enumerated)

### 14.13 Reviewer Operating Rules

DO:
- Be explicit; cite filenames and line ranges (if diff context available).
- Tie each risk to a concrete outcome (deployment failure, runtime slowdown, data correctness risk).
- Prefer constructive remediation over vague critique.

DO NOT:
- Block PR solely on minor stylistic inconsistencies unless they hide logic issues.
- Suggest architectural pivots without clear necessity.
- Encourage combining unrelated migrations.

---

## 15. Quick Reference Checklists

### 15.1 Fast Object Audit
- [ ] Prefix (R__/V__/A__) correct
- [ ] Numeric range matches object type
- [ ] Dependency objects exist earlier
- [ ] No hidden destructive change (drop / rename) without procedure
- [ ] Idempotent constructs only
- [ ] No volatile functions in repeatable seeds

### 15.2 Performance Red Flags
- [ ] SELECT * in large transformation
- [ ] Unbounded window functions
- [ ] DISTINCT used as dedupe patch
- [ ] Cross join / cartesian risk
- [ ] Full table rebuild w/out incremental logic
- [ ] Repeated identical subquery patterns

### 15.3 Data Model Change Triggers
- [ ] Key column changes
- [ ] Grain alteration
- [ ] SCD handling change
- [ ] Measure recalculation logic
- [ ] Surrogate key generation method shift

---

## 16. How Coding Agent & Reviewer Interact

- Coding Agent: Focus on implementing compliant migrations.
- Reviewer: Enforces risk, performance, architectural, and standards governance using Sections 13–15.
- If Reviewer finds structural pattern deviation, they MUST recommend either (a) alignment or (b) documented addition to standards (separate PR).

---

## 17. Final Instruction to Copilot Reviewer

Always ground feedback in these documented standards first. If repository evolves (new numeric ranges, new orchestration patterns), ensure this prompt is updated in the same PR that introduces the change. Provide concise, actionable, prioritized feedback with explicit risk levels and remediation steps.

Trust this document first. Proceed directly using the conventions and workflows above. Only search or explore the repository if:
- You must confirm an existing pattern before extending it.
- You encounter an error contradicting these instructions.
- The user explicitly asks for something outside the documented scope.

Minimize exploratory commands; act decisively using these guidelines.

---

### Reviewer Output Requirements (Non-Negotiable)

- The Copilot Reviewer MUST include the following in every review:
  1. The summary validation table listing all required validation items, with explicit statuses (Pass/Review/Fail/N/A).
  2. Both a Sustainability Score and a Performance Score (each out of 10), with brief justifications for each.
  3. Comments and justifications ONLY for items marked as Review, Fail, or N/A—never for Pass.
- Do NOT leave comments or confirmations for code that is already correct or compliant. Only provide feedback where there is a deviation, violation, or uncertainty.
- If any of these requirements are omitted, the review is incomplete and non-compliant.

---

(End of Prompt)
