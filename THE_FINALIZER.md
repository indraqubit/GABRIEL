# THE_FINALIZER — Production Codebase Audit Agent

**Date:** 2026-06-26
**Status:** Design specification (read-only doc, no implementation yet)
**Co-located with:** `PRODUCTION-CODEBASE-AUDIT-2026-06-26.md` (the audit that motivated this design)
**Mode:** Read-only auditor (per locked decision)
**Architecture:** Project-specific rule packs + framework-agnostic core

---

## 1. Purpose

**THE_FINALIZER** scans any production codebase and produces a single markdown report distinguishing:

- **HACKS** — anti-patterns that should be removed
- **BOILERPLATE** — clean patterns that should be replicated
- **RECOMMENDATIONS** — sequenced, prioritised, file:line-cited

The agent is the codified methodology behind `PRODUCTION-CODEBASE-AUDIT-2026-06-26.md`. That audit proved the methodology works on the Vortex Realm JWT codebase; this agent generalises it.

**Non-goals (explicit):**
- Not a code fixer. Read-only output. Auto-fix is a separate future capability.
- Not a replacement for ESLint, Prettier, or typecheckers. Operates on semantic patterns those tools miss (e.g. "console.log in prod paths", "hardcoded URL fallback when SSOT exists").
- Not a CI gate. Output is for human review by COUNSELOR/SURGEON, not for pipeline blocking.

**Scope (LIFESTAR↔BLESSED explicit per RFC Fix 9):**

THE_FINALIZER v1 is scoped to **BLESSED-derived categories only**. BLESSED is a CAROL-level subset of **LIFESTAR** (the project's 8-principle architectural methodology: Lean, Immutable, Findable, Explicit, SSOT, Testable, Accessible, Reviewable — see `LIFESTAR/PRINCIPLES.md`). Three letters overlap exactly (Lean, Explicit, SSOT) and are covered by the rule catalogue. Five do not appear in BLESSED at all:

| LIFESTAR letter | v1 coverage | Status |
|---|---|---|
| Lean | yes | covered (magic numbers, DRY violations, dead code) |
| Immutable | no | deliberately deferred to v2 |
| Findable | no | deliberately deferred to v2 |
| Explicit | yes | covered (magic strings, typed errors, `as any`) |
| SSOT | yes | covered (Fix 1, Fix 8, env-fallback duplication) |
| Testable | no | deliberately deferred to v2 |
| Accessible | no | deliberately deferred to v2 |
| Reviewable | no | deliberately deferred to v2 |

This is a deliberate scope choice, not an oversight. THE_FINALIZER's audience is COUNSELOR/SURGEON reading reports, where Bounds/Stateless/Deterministic/Encapsulation matter more than Testable/Accessible/Reviewable. LIFESTAR alignment is a v2 milestone — promotion to v1 is an ARCHITECT product decision.

---

## 2. Why This Exists

The 2026-06-26 audit of Vortex Realm JWT found:

- ~245 hacks across 195 production files (~31% hack ratio)
- ~540 clean patterns cited
- 6 P0 BLOCKERS, 16 P1 HIGH, 48 P3 MEDIUM findings
- **~80% of findings were concentrated in 5 files** — pattern detection would have surfaced these in minutes, not the 4-subagent parallel scan that took the audit

**The repetition is the problem.** Every codebase will have:

- Magic numbers in rate limits
- `console.log` debug statements left in prod
- Empty `catch {}` blocks
- Hardcoded URL fallbacks that duplicate SSOT
- `as any` casts hiding missing types
- `useEffect` for derived state (LAW 6 violations)
- Dev bypasses shipped in prod build
- Commented-out code blocks

**THE_FINALIZER** is the detector for these patterns, parameterised by framework rule packs.

---

## 3. Architecture

```
THE_FINALIZER
├── Core (framework-agnostic)
│   ├── File discovery (globs, extensions, exclude rules)
│   ├── Rule loader (merges packs based on detected file types)
│   ├── Pattern matcher (regex + AST-lite for type-aware rules)
│   ├── Findings aggregator (dedup, group by file, rank by severity)
│   ├── Report generator (markdown template, citations, BLESSED scoring)
│   └── Sprint grouping engine (groups findings → sprints, takes sprint rules as data)
│
├── Sprint Rules (separate data artifact, swappable without code change)
│   └── sprintRules.json          ← Default 5-sprint config; pack-specific overrides allowed
│
├── Rule Packs (project-specific, JSON config)
│   ├── netlify-functions.json     ← Vortex Realm JWT
│   ├── react-typescript.json      ← Generic React + TS
│   ├── node-cjs.json             ← Generic Node CJS
│   └── cpp-juce.json             ← portable-godclass / audio plugins
│
├── Shared Modules (DRY/SSOT compliance)
│   └── Imports from scripts/lib/* (logger, file-utils, runMain, BLESSED scoring utils)
│
└── Configuration
    ├── finalizer.config.json     ← Active packs, severity thresholds, output path, sprint rules path
    └── finalizer.config.example.json
```

### 3.1 Core responsibilities

| Component | Responsibility | Reference impl |
|-----------|---------------|----------------|
| File discovery | Apply include/exclude globs (positive globs + `!` exclusion prefix), classify by extension | `scripts/jacob/pattern-detector.mjs` (read for inspiration) |
| Rule loader | Merge multiple `.json` packs; pack precedence = later wins; per-pack `recommendationEngine` overrides merged into global sprint rules; per-pack `clusterRules` and `dagRules` declared (see §17, §18) | New |
| Pattern matcher | Regex for `detectionMethod: "regex"`; skip `ast-pending`; AST for `ast` if engine loaded | New — regex Phase 1, AST Phase 2 |
| Findings aggregator | Deduplicate (same file:line + category = 1 finding), group, rank | New |
| **Verification pipeline** | **For each candidate finding, run pattern-strength check → domain-knowledge check → multi-site consistency check → categorical confidence assignment (A/B/C/D). One pipeline, one confidence output.** Replaces prior "Confidence evaluator + Semantic verifier" split. | New — see §3.5 |
| Priority calculator | Compute sprint priority from severity + categorical confidence (Priority is independent of cost — see §7.5) | New |
| Scheduler | Compute scheduling recommendation from priority + structural scope (separate function — see §7.5) | New |
| Cluster grouper | Group findings into clusters via co-location and category similarity (no human input at runtime) | New — see §17 |
| Architecture hypothesizer | For each cluster, look up `clusterRules` from active rule packs and emit the declared hypothesis | New — see §17 |
| DAG builder | Build finding DAG from inferred edges via `dagRules` declared in rule packs | New — see §18 |
| Report generator | Fill `REPORT_TEMPLATE.md` with findings + categorical confidence + priority + scheduling + clusters + DAG + stats | New |
| Sprint grouping engine | Generic: takes `(findings[], sprintRules)` → `sprints[]`. No hardcoded sprint count. | New |

Every rule MUST declare a `detectionMethod` field. This resolves the contradiction between the original spec (which ambiguously listed AST-required rules in Phase 1) and the cross-framework validation doc (which listed the same rules as "regex-validated").

```json
{
  "packName": "netlify-functions",
  "version": "1.0.0",
  "appliesTo": {
    "extensions": [".cjs", ".js"],
    "pathPrefixes": ["netlify/functions/"],
    "excludePathPrefixes": ["netlify/functions/templates/", "netlify/functions/__tests__/"]
  },
  "hacks": [
    {
      "id": "console.log.prod-path",
      "category": "debug_in_prod",
      "severity": "P3",
      "detectionMethod": "regex",
      "pattern": "^\\s*console\\.(log|debug)\\(",
      "scope": "anywhere",
      "whitelist": { "patterns": [] },
      "rationale": "Bare console output bypasses logger SSOT; prod logs leak to Netlify console"
    },
    {
      "id": "json-parse-unprotected",
      "category": "unprotected_json",
      "severity": "P1",
      "detectionMethod": "regex",
      "pattern": "JSON\\.parse\\([^)]*event\\.body",
      "scope": "routes/*.cjs",
      "whitelist": { "patterns": [] },
      "rationale": "Malformed body throws and crashes handler; should be wrapped in try/catch"
    },
    {
      "id": "if-false-block",
      "category": "dead_code",
      "severity": "P0",
      "detectionMethod": "regex",
      "pattern": "if\\s*\\(\\s*false\\s*\\)\\s*\\{",
      "scope": "anywhere",
      "whitelist": { "patterns": [] },
      "rationale": "Dead code with lint disabled is a maintenance bomb"
    },
    {
      "id": "use-effect-derived-state",
      "category": "law_6_violation",
      "severity": "P2",
      "detectionMethod": "ast-pending",
      "pattern": null,
      "scope": "client/**/*.{ts,tsx}",
      "whitelist": { "patterns": [] },
      "rationale": "useEffect for state that's a function of props/state — should be useMemo (FRONTEND_LAW_VAULT LAW 6). Requires AST to detect reliably."
    }
  ],
  "positive": [
    {
      "id": "service-purity-banner",
      "pattern": "/\\*\\*[\\s\\S]*?NO req\\/res[\\s\\S]*?\\*/",
      "scope": "services/*.cjs",
      "rationale": "Documents the service-layer contract"
    }
  ],
  "neutral": [
    {
      "id": "uses-std-variant",
      "pattern": "\\bstd::variant\\b",
      "scope": "anywhere",
      "rationale": "std::variant is appropriate in some contexts, inappropriate in others. No judgment."
    }
  ],
  "dagRules": [
    {
      "from": "magic-number-rate-limit",
      "to": "duplicate-literal-same-value",
      "edgeType": "shares-root-cause",
      "condition": "same-file"
    }
  ],
  "clusterRules": [
    {
      "id": "config-drift",
      "signature": ["magic-number", "duplicate-literal", "env-fallback-url"],
      "hypothesis": "Missing central constants module — fixes at one location can eliminate 60%+ of cluster findings",
      "structuralScope": "Project"
    }
  ]
}
```

### 3.3 Severity tiers

| Tier | Meaning | Example |
|------|---------|---------|
| **P0** | Production risk; ship-blocker | Dev bypass in prod build, 294-line `if (false)` block, silent `catch {}` in SSOT |
| **P1** | Correctness/maintainability | Unprotected `JSON.parse`, hardcoded URL fallback, `as any` chain |
| **P2** | Convention/DRY | Magic numbers without named constants, repeated string literals |
| **P3** | Style | Inline `style={{}}` for animation (LAW 1), minor `console.error` residue |

### 3.4 Detection method values

| Value | Meaning | Matcher behaviour |
|-------|---------|-------------------|
| `"regex"` | Pattern is a regex string; matcher executes Phase 1 | Evaluated every scan |
| `"ast-pending"` | Rule rationale + scope shipped today; matcher cannot evaluate | Skipped every scan, listed in `--stats` output as "deferred" |
| `"ast"` | Pattern is an AST query string; matcher executes Phase 2+ | Evaluated only when AST engine is loaded (Phase 2+) |

When matcher loads a pack:
- Evaluates `detectionMethod: "regex"` rules normally
- Skips `detectionMethod: "ast-pending"` rules entirely (does NOT attempt regex approximation — silent undercount preferred over false positive)
- Evaluates `detectionMethod: "ast"` rules only if `finalizer.config.json` declares `astEngine: "typescript"` or similar

Rules can be promoted from `"ast-pending"` → `"ast"` → (no field, default `"regex"`) as implementation progresses.

### 3.5 Verification Pipeline (consolidated)

The v3 spec consolidates what v2 split into two layers (`Confidence evaluator` + `Semantic verifier`) into **one verification pipeline**. Reason: those two layers produced confidence in two passes — matcher emits candidate, confidence scores it, verifier rerates it. The two passes are redundant when both run.

**Pipeline stages** (in order, all run on every candidate finding):

1. **Pattern strength check** — how clean is the regex match? Offset drift? Ambiguous category? Outputs initial confidence bucket.

2. **Domain knowledge check** — does the rule declare a `semanticDomain`? If yes, route to per-framework verifier. Verifier outputs `confirmed | refuted | indeterminate`. Indeterminate findings drop one confidence bucket (A→B→C).

3. **Multi-site consistency check** — if the candidate finding is `ssot_violation`, check whether the values across sites are actually different in framework semantics (the JUCE `Oversampling` case). Same name + same value = POSITIVE finding (boilerplate), not NEGATIVE (drift).

4. **Categorical confidence assignment** — final output is one of four buckets (see §7.2):
   - `A` (auto-confirm) — pattern clean, no ambiguity
   - `B` (quick review) — pattern clean, but category is heuristic
   - `C` (manual review) — domain knowledge required
   - `D` (suppress) — likely false positive, included for visibility only

**Pipeline schema** — rules MAY declare:

```json
{
  "id": "oversample-factor-mismatch",
  "category": "ssot_violation",
  "severity": "P1",
  "detectionMethod": "regex",
  "semanticDomain": "juce-dsp-oversampling",
  "pattern": "...",
  "scope": "...",
  "rationale": "..."
}
```

| Field | Type | Meaning |
|-------|------|---------|
| `semanticDomain` | string | Identifier for the framework semantic to consult (e.g. `"juce-dsp-oversampling"`). If absent, pipeline stage 2 is skipped. |

**Verifier architecture** (deferred to Phase 4+; not Phase 1):

```
scripts/finalizer/verifiers/
├── juce-dsp-oversampling.mjs   // knows: factor is 2^arg
├── react-use-effect.mjs         // knows: derived state, deps analysis
└── netlify-handler.mjs          // knows: event signature
```

**Why consolidate**: the v2 design ran matcher → confidence → verifier → confidence. Two confidence passes created ambiguity about which one was authoritative. The v3 pipeline has one confidence output per finding, computed from a deterministic sequence of checks. JUCE `Oversampling` false positive would have been caught at stage 2 (domain knowledge check) and bucket-downgraded from `B` to `C` before publication — preventing the P0 silent publication that happened in v2.

---

## 4. Detection Patterns — Reference Catalogue

The audit found these patterns. Each becomes a rule ID in a pack.

### 4.1 HACK patterns (high-yield)

| Rule ID | Category | Pattern (regex sketch) | Severity |
|---------|----------|------------------------|----------|
| `console-log-prod` | debug_in_prod | `console\.(log\|debug)\(` | P3 |
| `catch-empty-truly-silent` | silent_swallow | `catch\s*(\(\s*\))?\s*\{\s*\}` | P0 |
| `catch-fallback-undocumented` | silent_swallow | `catch\s*\(\s*\w+\s*\)\s*\{\s*(return\s+(null\|false\|\[\]\|\{\})\|/\*[^*]*\*/)` | P3 |
| `magic-number-rate-limit` | magic_number | `checkRateLimit\([^,]+,\s*\d+,\s*\d+` | P3 |
| `env-fallback-url` | env_fallback_duplication | `process\.env\.\w+\s*\|\|\s*["']https?://` | P2 |
| `hardcoded-url-fallback` | env_fallback_duplication | `["']https?://(byiqaudio\|example)\.com["']` | P2 |
| `json-parse-unprotected` | unprotected_json | `JSON\.parse\(` | P1 |
| `todo-comment` | todo_comment | `//\s*(TODO\|FIXME\|HACK\|XXX)` | P3 |
| `commented-code-block` | commented_code | `^\s*//.*\n\s*//.*\n\s*//.*` (≥3 lines) | P3 |
| `if-false-block` | dead_code | `if\s*\(\s*false\s*\)\s*\{` | P0 |
| `manual-workaround-flag` | manual_workaround | `window\.__\w+\s*=` | P0 in prod pages |
| `dev-email-bypass` | hardcoded_dev_data | `(testuser\|admin)@(example\|dev)\.\w+` | P0 in prod pages |
| `use-effect-derived-state` | law_6_violation | useEffect for state that's a function of props/state | P2 (AST rule) |
| `use-effect-empty-deps-with-refs` | law_8_violation | useEffect `[]` referencing non-stable closures | P2 (AST rule) |
| `fetch-no-abort-signal` | no_abort | `fetch(...)` without `signal:` param | P1 |
| `set-timeout-no-cleanup` | no_cleanup | `setTimeout` in component without `clearTimeout` in cleanup | P2 |
| `as-any-cast` | type_anti_pattern | `as\s+any\b` | P2 (production only) |
| `type-cast-chain` | type_anti_pattern | `as\s+\w+\s*&\s*\{` | P2 (≥3 occurrences in file) |
| `bare-createClient` | ssot_violation | `createClient\(` outside factory file | P2 |
| `bare-console-error` | ssot_violation | `console\.error\(` in non-logger file | P3 |
| `process-env-fallback` | env_fallback | `process\.env\.\w+\s*\|\|\s*['"]` | P3 |
| `literal-duplication` | literal_duplication | Same literal value (string, number, URL) appearing in ≥3 files without a shared constant | P1 |
| `early-return-multi` | early_returns | ≥3 `return null` in single function | P3 |
| `inline-style-animation` | law_1_violation | `style=\{\{[^}]*animation\|transform\|transition` | P3 |
| `dangerously-set-inner-html` | dangerous_html | `dangerouslySetInnerHTML` | P3 |
| `eval-constructor` | dangerous_code | `eval\(\|new Function\(` | P0 |
| `string-concat-sql` | sql_injection | `['"`]\s*\+\s*\w+\s*\+\s*['"`].*(?:SELECT\|INSERT\|UPDATE\|DELETE)` | P0 |

### 4.2 BOILERPLATE patterns (positive detection)

| Rule ID | Pattern | Scope |
|---------|---------|-------|
| `service-purity-banner` | `NO req/res` comment | services/ |
| `jsdoc-public-fn` | `/**` block above `exports.`/`module.exports` | routes/, services/ |
| `supabase-factory-usage` | `createSupabaseClient(` | services/, routes/ |
| `abort-signal-fetch` | `AbortSignal.timeout\|controller.signal` | any |
| `cleanup-useeffect` | `return () =>` in useEffect | React components |
| `usememo-derived` | `useMemo` for derived values | React components |
| `event-handler-side-effect` | `onClick\|onChange` not in useEffect | React components |
| `explicit-prop-interface` | `interface \w+Props` before component | React components |
| `ssot-style-import` | `import.*stylePresets` | React components |
| `logger-shared` | `logger\.(info\|warn\|error\|debug)` | any |
| `handle-error-typed` | `handleError(ErrorCreators\.\w+\(\.\.\.\))` | routes/ |
| `upper-snake-const` | `const [A-Z_]+ =` | module top |

---

## 5. CLI Surface (proposed)

```bash
# Default — scan everything in current project
npm run finalizer

# Scan specific paths
npm run finalizer -- --include "netlify/functions/**" --include "client/src/**"

# Use specific rule packs
npm run finalizer -- --packs netlify-functions,react-typescript

# Output to file (default: stdout)
npm run finalizer -- --output reports/audit-2026-06-26.md

# Severity filter (default: all)
npm run finalizer -- --severity P0,P1

# Show only hack findings (no boilerplate)
npm run finalizer -- --hacks-only

# Show only boilerplate (positive patterns)
npm run finalizer -- --boilerplate-only

# Generate recommendations section only
npm run finalizer -- --recommendations

# Dry-run with stats only
npm run finalizer -- --stats

# Compare two runs (diff mode)
npm run finalizer -- --diff reports/audit-prev.md
```

### 5.1 npm script entry

```json
{
  "scripts": {
    "finalizer": "node scripts/finalizer/finalizer.mjs",
    "finalizer:hacks": "node scripts/finalizer/finalizer.mjs --hacks-only",
    "finalizer:boilerplate": "node scripts/finalizer/finalizer.mjs --boilerplate-only",
    "finalizer:stats": "node scripts/finalizer/finalizer.mjs --stats",
    "finalizer:diff": "node scripts/finalizer/finalizer.mjs --diff"
  }
}
```

---

## 6. Output Format

The agent produces a markdown file matching the structure of `PRODUCTION-CODEBASE-AUDIT-2026-06-26.md`:

```markdown
# Production Codebase Audit — [date]

## 1. Scope Stats
[file count, line count, finding counts by class: positive / neutral / negative, confidence histogram]

## 2. Architecture Hypotheses (NEW in v3 — see §17)
[for each detected cluster: cluster signature, hypothesis, estimated % of findings eliminated by single systemic fix]

## 3. Negative Findings — Hacks (with 3D evaluation)

### 3.1 Tier 1: BLOCKERS (Severity P0)
| File:Line | Rule | Bucket | Scope | Priority | Review? |
|----------|------|--------|-------|---------:|---------|
| netlify/functions/utils/url.util.cjs:29 | catch-empty-truly-silent | A | local | 4.0 | — |
| client/pages/Maintenance.tsx:433 | if-false-block | A | local | 4.0 | — |
| ...      | ...  | ...    | ...   | ...     | ...     |

### 3.2 P0 Findings Requiring Human Review (Bucket C or D)
| File:Line | Rule | Bucket | Reason |
|----------|------|--------|--------|
| PluginProcessor.h:73 | oversample-factor-mismatch | C | JUCE domain knowledge required |
| ...      | ...  | ...    | ...   |

### 3.3 Tier 2: HIGH (Severity P1)
### 3.4 Tier 3: MEDIUM (Severity P2)

## 4. Neutral Observations (NEW in v3 — context-dependent patterns)
[findings from the `neutral` rule pack array; does NOT enter sprint planning]

## 5. Positive Patterns — Replicable Code
[stats + top 5 cleanest files]

## 6. Finding DAG (NEW in v3 — see §18)
[visual or text representation of inferred edges between findings; example: "magic-number@X and duplicate-literal@X share-root-cause"]

## 7. Suppressed Findings Appendix
[findings at bucket D; not in sprint planning but visible for transparency]

## 8. Summary
[finding count by class, 3D distribution histogram, top 5 hackiest files, top 5 most-frequent clusters]

## 9. Recommended Fix Priority (sorted by Priority, then Scheduling)
[sorted by priority = severity × confidence_bucket_weight]

## 10. Comments — What's Working / What's Not
[auto-generated commentary based on positive vs negative pattern distribution]

## 11. Recommendations — Sequenced Sprints
[sprint plan grouped by priority; each finding has priority + scheduling; architectural hypotheses are flagged as "consider systemic fix instead of per-finding"]
```

### 6.1 Recommendation Engine Logic

The engine groups findings into sprints using **Priority** (independent of cost) and emits **Scheduling** recommendations separately. Sprint grouping is data-driven from the finding set, not a fixed template.

Default sprint taxonomy (overridable per project via `sprintRules.json`):

| Sprint | Default inclusion |
|--------|-------------------|
| **Sprint 1 (P0 BLOCKERS)** | severity P0 AND confidence ∈ {A, B} |
| **Sprint 1-Review (P0 Human Review)** | severity P0 AND confidence ∈ {C, D} (engineered review before sprint) |
| **Sprint 2 (SSOT consolidation)** | category `literal_duplication`, regardless of severity |
| **Sprint 3 (Magic numbers)** | category `magic_number`, severity P1–P3 |
| **Sprint 4 (TypeScript hygiene)** | category `type_anti_pattern` or `as_any` |
| **Sprint 5 (Frontend Law)** | category `law_*_violation` |

For each finding in sprint, the report shows:
- Priority (importance — drives sprint inclusion)
- Scheduling (timing — drives sprint ordering within sprint)
- Structural Scope (local/module/project/external — drives which engineer can fix it)

**Architectural hypotheses** (NEW in v3, see §17): if the report's cluster detector identifies a cluster whose hypothesis suggests a single systemic fix, the recommendation text reads:

> *"Cluster 'Configuration Drift' has 47 findings. Hypothesis: 'Missing central constants module (constants.h)'. Estimated impact: a single new file `src/constants.h` consolidating 12 magic-number declarations would eliminate ~80% of findings in this cluster. Consider prioritizing the systemic fix over per-finding fixes."*

This shifts the report from "list of 108 findings" to "list of 108 findings + 7 systemic fixes that would address most of them."

> **Note (RFC Fix 8):** `literal_duplication` and `silent_swallow` are distinct categories. The engine must not collapse them under the informal "SSOT" label during Sprint 2 grouping — they have different fix shapes.

> **Note (v3, suppressed findings)**: Bucket D findings appear in §7 (Suppressed Findings Appendix) but do NOT enter sprint planning. They are surfaced for transparency — if 30% of all findings get suppressed, that itself is a signal worth reporting.

> **Note (4D-aware grouping)**: Sprint grouping is now driven by 4D priority score, not severity alone. A P0 with confidence 0.4 may end up in Sprint 1-Review; a P1 with confidence 0.95 and `trivial` cost may sprint ahead of a P2 with confidence 0.5.

---

## 7. Three-Dimension Evaluation Framework

Severity alone is insufficient for actionable audit output. THE_FINALIZER evaluates each finding on **three orthogonal dimensions**:

| Dimension | Question it answers | Type |
|-----------|---------------------|------|
| **Severity** | *If* the finding is correct, how bad is the impact? | enum: P0 / P1 / P2 / P3 |
| **Confidence** | How likely is this finding a real issue (not a false positive)? | categorical: A / B / C / D |
| **Structural Scope** | Where does the fix live? (NOT how long it takes) | enum: `local` / `module` / `project` / `external` |

**Removed dimensions** (compared to v2):
- `Fix Cost` (time estimate) — removed because static analyzers cannot reliably estimate human effort
- `Intent` — removed from finding schema; moved to separate `annotations` lifecycle (see §7.9)

### 7.1 Severity

**Type:** enum `P0 | P1 | P2 | P3` (unchanged from §3.3).

| Tier | Meaning |
|------|---------|
| **P0** | Production risk; ship-blocker |
| **P1** | Correctness/maintainability |
| **P2** | Convention/DRY |
| **P3** | Style |

Severity is **producer-side**: how much damage if this code runs as-is.

### 7.2 Confidence (categorical)

**Type:** enum `A | B | C | D`.

The v2 design used a float `[0.0, 1.0]`. This was **rejected** because:

1. **False precision.** A float implies continuous measurement. Behind the float are bucket heuristics ("regex clean" → bucket A, "needs framework knowledge" → bucket C). Reviewers anchor on the digits (`0.82` looks more accurate than `B`) without the underlying signal changing.

2. **Comparison artifacts.** `0.78` vs `0.81` look meaningfully different; in buckets, both are `B`. Float invites noise distinctions.

3. **Threshold pollution.** Every gate (auto-confirm, suppress, etc.) needs a threshold, which forces converting categorical back to numeric. Better to keep categorical throughout.

**Buckets:**

| Bucket | Meaning | Verifier output |
|--------|---------|----------------|
| **`A`** (auto-confirm) | Pattern matched cleanly. Line citation verified. No domain ambiguity. | Pipeline stage 2: skip (no `semanticDomain`) or `confirmed` |
| **`B`** (quick review) | Pattern matched cleanly but category is heuristic (e.g. magic-string-duplication may have intentional context). | Pipeline stage 2: skip or `confirmed` |
| **`C`** (manual review) | Pattern requires framework domain knowledge to verify. Likely real but cannot confirm without human. | Pipeline stage 2: `indeterminate` (bucket-downgraded from `B`) or no verifier registered |
| **`D`** (suppress) | Pattern is suspicious. Prior false positives in same category. Do not enter sprint planning; surface for visibility only. | Pipeline stage 2: `refuted` or repeated-history-false-positive |

**Default** when evaluator cannot determine bucket: `C` (manual review). Conservative default — better to require review than to publish silently.

**Mandatory rule**: P0 findings at bucket `C` or `D` MUST be flagged `humanReviewRequired: true`. P1+ at bucket `D` are suppressed from sprint recommendations but appear in the report's "Suppressed Findings" appendix.

**Worked example — false positive (v3 post-mortem)**: the JUCE `Oversampling<float>{ 2, 2, ... }` finding was originally published at P0 with implicit confidence 1.0 (v1 design). Under v2's float, it would have been assigned 0.5. Under v3's categorical, pipeline stage 2 routes it to a juce-dsp-oversampling verifier, which returns `indeterminate`, and bucket-downgrades from `B` to `C`. The P0 publication would never have happened — the finding would have been flagged for human review with explicit "requires JUCE domain knowledge" reason.

Confidence is **validator-side**: how sure are we that this finding represents a real issue.

### 7.3 Structural Scope

**Type:** enum `local | module | project | external`.

The v2 design used `Fix Cost` (time estimate). This was **rejected** because:

1. Static analyzers have no signal about sprint capacity, developer availability, dependencies, or release timing.
2. The same fix can take 5 minutes or 3 weeks depending on factors invisible to the analyzer (e.g. is this a public API change?).
3. Time estimates from static analysis are consistently wrong in one direction (underestimating complexity).

**Reformulation**: instead of asking *how long*, ask *where* the fix lives. This is **structurally detectable** from import statements, file scope, and dependency direction.

| Level | Meaning | Example | Detectable from |
|-------|---------|---------|----------------|
| **`local`** | Single file, no cross-file impact | Rename variable, replace `substr()` with `substring()`, add cleanup in same file | Same file as finding |
| **`module`** | Multiple files within one logical module | Consolidate two SSOT declarations, add named constant shared within module | Imports within module boundary |
| **`project`** | Multiple modules, requires interface or contract change | Refactor an API signature, extract shared library, change error-handling convention | Cross-module dependency |
| **`external`** | Cannot be fixed from this codebase alone | Update foreign product source tree, change vendor license config, sync with upstream | Import points to foreign source tree (e.g. `dsp_silverdad/License/`) |

**Default** when evaluator cannot determine scope: `module` (most findings).

Structural Scope affects **scheduling**, not **priority**. See §7.5.

### 7.4 Removed: Intent (moved to annotations)

The v2 design placed `intent` as a nullable field on each finding with an anti-default rule (never default to `"unintentional"`). This was **rejected** because:

1. **98% of findings will have `intent: null`** at scan time. A nullable field that almost always carries its default value is noise.
2. **`null` default invites god-object temptation** — an engineer reviewing a finding might feel compelled to fill `intent` with `"unknown"` rather than leave it null, claiming false completeness.
3. **Intent is not a property of the finding** — it is a property of the *team's relationship with the code*. It belongs in a separate lifecycle, not on the finding itself.

**Replacement** (§7.9): Intent lives in a separate `annotations` file, written by humans, with provenance. The finding is the analyzer's output; the annotation is the engineer's relationship to it.

### 7.5 Priority vs. Scheduling (separated)

The v2 design used `priority = severity × confidence × cost_penalty` (single formula). This was **rejected** because:

1. **Cost should not reduce importance.** A P0 with `external` scope remains P0 regardless of how expensive the fix is. Reducing its priority because of cost hides a production risk behind a planning concern.
2. **The two questions are different**: "Is this important?" vs "When can we work on it?" Confounding them produces misleading rankings.

**Replacement**: two functions, one for importance, one for timing.

```
Priority   = severity_rank × confidence_bucket_weight
Scheduling = Priority × scope_delay_factor
```

Where:
- `severity_rank`: P0=4, P1=3, P2=2, P3=1
- `confidence_bucket_weight`: A=1.0, B=0.9, C=0.6, D=0.2 (categorical, not continuous)
- `scope_delay_factor`: local=1.0, module=1.0, project=0.6, external=0.3

**Example priority calculations:**
- P0 + A + local: `4 × 1.0 × 1.0 = 4.0` (top of sprint 1)
- P0 + A + external: `4 × 1.0 × 0.3 = 1.2` (priority still meaningful; scheduling delayed)
- P1 + C + project: `3 × 0.6 × 0.6 = 1.08` (low priority, manual review required, scheduled later)

**Key property**: P0 + external still has higher Priority than P2 + local. Importance is not consumed by cost.

### 7.6 Confidence-aware P0 gate (categorical)

P0 findings at bucket `C` or `D` are flagged `humanReviewRequired: true`. The report includes a separate section:

```
## P0 Findings Requiring Human Review

| File:Line | Rule | Bucket | Reason |
|----------|------|--------|--------|
| PluginProcessor.h:73 | oversample-factor-mismatch | C | JUCE domain knowledge required to interpret constructor semantics |
| ...      | ...  | ...    | ...   |
```

The owning engineer reviews before sprint kickoff. Verdicts:
- **Confirm**: raise confidence bucket (e.g. C → B), flag resolved
- **Refute**: drop finding, log to false-positive history
- **Keep as-is**: leave bucket, add annotation explaining engineering decision

### 7.7 Finding classification (positive / neutral / negative)

The v1/v2 binary `Boilerplate` vs `Hack` is **insufficient**. Many patterns (`std::variant`, `std::optional`, RAII, smart pointers) are **neutral** — appropriate in some contexts, inappropriate in others. The static analyzer cannot judge context.

**Three classes** (rule packs declare per-rule):

| Class | Meaning | Action |
|-------|---------|--------|
| `positive` | Pattern is exemplary code. Multi-site consistency, named constants, proper SSOT propagation. | Appears in "Boilerplate" section of report. Worth replicating elsewhere. |
| `neutral` | Pattern is neither good nor bad. The judgment depends on context the analyzer lacks. | Appears in "Neutral Observations" section. Engineers review for context fit. **Does not enter sprint planning.** |
| `negative` | Pattern is a hack or anti-pattern. The standard `hack` findings. | Appears in "Hacks" section. Enters sprint planning with severity + confidence. |

Rule pack schema adds:

```json
{
  "positive": [ ... ],     // boilerplate (renamed for clarity)
  "neutral":  [ ... ],     // new
  "negative": [ ... ]      // was "hacks"
}
```

The v1/v2 `boilerplate` array becomes `positive`. The `hacks` array becomes `negative`. The new `neutral` array captures the previously-unnamed middle class.

### 7.8 Schema: Finding object (v3)

```json
{
  "ruleId": "if-false-block",
  "file": "client/pages/Maintenance.tsx",
  "line": 433,
  "category": "dead_code",
  "severity": "P0",
  "confidence": "A",
  "structuralScope": "local",
  "priority": 4.0,
  "scheduling": 4.0,
  "humanReviewRequired": false,
  "rationale": "if (false) block with eslint-disable; production dead code",
  "evidence": "verified at file:line, no offset drift"
}
```

For findings requiring review:

```json
{
  "ruleId": "oversample-factor-mismatch",
  "file": "PluginProcessor.h:73",
  "line": 73,
  "category": "ssot_violation",
  "severity": "P1",
  "confidence": "C",
  "structuralScope": "module",
  "priority": 1.8,
  "scheduling": 1.8,
  "humanReviewRequired": true,
  "reviewReason": "Constructor argument is oversampling exponent (2^factor); literal interpretation is false positive. See JUCE docs.",
  "rationale": "Static pattern matched but framework domain knowledge required to disambiguate"
}
```

### 7.9 Annotations (separate lifecycle from findings)

Intent is no longer part of the finding schema. It lives in a separate `annotations` file, written by humans, with full provenance.

**File format** (per project, NOT part of scan output):

```yaml
# annotations.yaml — written by engineers, NOT by THE_FINALIZER
annotations:
  - findingId: "if-false-block@Maintenance.tsx:433"
    intent: "intentional"
    source: "engineer:indraqadarsih"
    timestamp: "2026-06-26T05:30:00Z"
    rationale: "Performance benchmark in Sprint 47; block kept to allow A/B test comparison against disabled branch"
  - findingId: "json-parse-unprotected@auth.routes.cjs:80"
    intent: "intentional"
    source: "engineer:someone-else"
    timestamp: "2026-06-15T12:00:00Z"
    rationale: "Validated upstream; throw is caught by global handler"
```

**Rules:**
- The matcher NEVER writes this file. Annotations are a human-controlled ledger.
- Source must be a named human or git commit SHA. Anonymous intent is not accepted.
- The `annotations.yaml` file is loaded at sprint planning time, not at scan time. It does not affect what the analyzer reports — only how the team acts on reports.

### 7.10 Migration from v2

| v2 dimension | v3 disposition |
|--------------|----------------|
| `severity: P0/P1/P2/P3` | unchanged |
| `confidence: 0.0-1.0` (float) | `confidence: A/B/C/D` (categorical) |
| `fixCost: trivial/small/medium/large/external` | replaced by `structuralScope: local/module/project/external` |
| `intent: null/"intentional"/"unintentional"/"unknown"` | moved to `annotations.yaml` (separate lifecycle) |
| `priority = severity × confidence × cost` (single) | split into `priority` (severity + confidence) and `scheduling` (priority + scope) |
| `humanReviewRequired: bool` | retained; now driven by categorical confidence thresholds |
| `boilerplate` array in pack | renamed to `positive` |
| `hacks` array in pack | renamed to `negative` |
| (none) | new `neutral` array for context-dependent patterns |

**Migration of legacy confidence values** (if older reports exist):
- 0.9–1.0 → A
- 0.7–0.9 → B
- 0.5–0.7 → C
- <0.5 → D

### 7.11 Why this redesign (rationale)

The v2 design (float confidence + fix cost + intent) had three structural problems addressed here:

| v2 problem | v3 fix |
|-----------|--------|
| Float confidence invites false precision | Categorical buckets; no `0.78` vs `0.81` distinction |
| Cost should not reduce importance | Priority (importance) and Scheduling (timing) separated |
| Intent on finding creates noise | Moved to annotations.yaml, separate lifecycle |

The v3 design preserves what v2 got right (3-dimension evaluation, human review gate, semantic verifier) and removes what v2 got wrong (false precision, conflated axes, god-object fields).

---

## 8. Differences from Existing Agents

| Existing | What it does | THE_FINALIZER's niche |
|----------|--------------|----------------------|
| `scripts/jacob/` | Pattern detection across project (broad) | THE_FINALIZER: focused on hack/boilerplate split |
| `scripts/sheriff/` | Lifecycle Law compliance | THE_FINALIZER: not lifecycle, not law compliance — semantic cleanliness |
| `scripts/code-quality/auto-code-review.mjs` | Auto code review with diffs | THE_FINALIZER: read-only scan, produces report not patches |
| `scripts/ssot/god-file-refactor-planner.mjs` | God-file detection | THE_FINALIZER: per-file pattern density, not file-size |
| `scripts/code-quality/enhanced-debt-detector.mjs` | Tech debt detection | THE_FINALIZER: hack-vs-boilerplate framing, dual output |
| `scripts/eslint/eslint-rule-analyzer.mjs` | ESLint rule patterns | THE_FINALIZER: cross-language semantic patterns ESLint misses |
| `scripts/dry-scanner.mjs` | DRY violation detection | THE_FINALIZER: anti-patterns, not duplications |

**THE_FINALIZER's position in the agent ecosystem:** It is the **read-only semantic auditor** — it tells you what is hack and what is boilerplate, then leaves fixing to SURGEON with the audit as input. It does not compete with ESLint, JACOB, or the dry-scanner; it complements them with a different lens.

---

## 9. Implementation Roadmap

### 9.1 Phase 1 — Core + one pack (1-2 days)

**Phase 1 explicitly does NOT cover rules with `detectionMethod: "ast-pending"` or `"ast"`.** Specifically excluded from Phase 1 smoke-test expectations:

- `use-effect-derived-state` (LAW 6 violation)
- `use-effect-empty-deps-with-refs` (LAW 8 violation)
- Any `cpp-juce.json` rules requiring call-graph analysis (e.g., audio-thread violations, lock-ordering)

These 3+ findings from the 2026-06-26 Vortex audit should be **subtracted from the expected-match count** when comparing Phase 1 smoke test output against the original audit:
- `client/components/common/UserPortal.tsx:111` (useEffect for state sync — LAW 6)
- `client/components/user/PurchaseList.tsx:75` (useEffect for derived `filteredPurchases` — LAW 6)
- `client/components/user/UserLicenseCenter.tsx:113` (useEffect `[]` deps with non-stable closure — LAW 8)

A silent undercount of exactly these 3 findings is the **expected outcome**, not a bug. Phase 1 will document this in the smoke-test report.

Phase 1 file list:

- [ ] `scripts/finalizer/finalizer.mjs` — main entry, imports from `scripts/lib/*`
- [ ] `scripts/finalizer/config-loader.mjs` — reads `finalizer.config.json` + pack files
- [ ] `scripts/finalizer/pattern-matcher.mjs` — regex matcher, deduplication, skips `ast-pending` rules
- [ ] `scripts/finalizer/report-generator.mjs` — fills template, includes BLESSED scoring per §6.2
- [ ] `scripts/finalizer/recommendation-engine.mjs` — generic sprint-grouping, takes `sprintRules.json` per §8.4
- [ ] `scripts/finalizer/packs/netlify-functions.json` — port findings from 2026-06-26 audit
- [ ] `scripts/finalizer/packs/react-typescript.json` — generic React + TS (Phase 1 ships the pack *file* with all rules marked `ast-pending` for LAW 6/8; matcher skips them)
- [ ] `scripts/finalizer/packs/node-cjs.json` — generic Node CJS
- [ ] `scripts/finalizer/finalizer.config.example.json` — with documented glob semantics
- [ ] `scripts/finalizer/REPORT_TEMPLATE.md`
- [ ] Add to `package.json` scripts
- [ ] Smoke test on Vortex Realm JWT — match original audit findings, minus 3 expected Phase-1-excluded findings (documented above)

### 9.2 Phase 2 — AST-lite rules (3-5 days)

- [ ] Add `typescript` package as optional dev dependency
- [ ] Implement AST-based rule engine for: `as any` chain, `useEffect` deps analysis, `useMemo` candidates
- [ ] Add rule `type-cast-chain` (≥3 `as X & {...}` in file)
- [ ] Add rule `use-effect-derived-state` (effect body only assigns from props/state)

### 9.3 Phase 3 — Diff mode (1 day)

- [ ] `--diff <file>` flag — compare current scan vs. saved baseline
- [ ] Output "Regressions" / "Improvements" sections
- [ ] Useful for tracking cleanup sprint progress

### 9.4 Phase 4 — Cross-framework packs (2-3 days)

- [ ] `cpp-juce.json` pack — port patterns from `scripts/code-quality/cpp-juce-guardrails.mjs`
- [ ] `python-flask.json` pack — for plugin installer scripts
- [ ] `nextjs.json` pack — generic Next.js conventions

### 9.5 Phase 5 — Comment generation (2-3 days)

- [ ] Section generators: "What's Working" (boilerplate stats), "What Needs Attention" (hack concentration analysis)
- [ ] Recommendation narrative generator — apply sprint rules, generate prose

---

## 10. Shared-Module Dependencies (DRY/SSOT compliance)

THE_FINALIZER MUST use existing shared modules:

```javascript
import { resolveFromRoot } from '../lib/path-utils.mjs';
import { readText, writeText, ensureDir, listFiles } from '../lib/file-utils.mjs';
import { exec } from '../lib/command-utils.mjs';
import log from '../lib/logger.mjs';
import { runMain } from '../lib/error-utils.mjs';
```

**Phase 3** adds `parseArgs` from `../lib/args-utils.mjs` when CLI flags (`--diff`, `--config`, `--include`, etc.) are introduced. Phase 1 reads configuration from `finalizer.config.json` only — no CLI argument parsing.

Reference: `scripts/lib/CONTRACT.md` for full module contracts.

**No re-implementing** path resolution, file I/O, command execution, JSON parsing, logging, error handling, or argument parsing.

---

## 11. Lifecycle Law Compliance

Per `timeLaw/LIFECYCLE_LAW_V1.md`, the script MUST include:

```javascript
// LAW 4: Top-level error handlers (MANDATORY)
function installFatalHandlers() {
  function fatal(err) {
    console.error('FATAL:', err);
    process.exit(1);
  }
  process.on('unhandledRejection', fatal);
  process.on('uncaughtException', fatal);
}
installFatalHandlers();

// LAW 1: Explicit exit (REQUIRED at end)
await runMain(async () => {
  // ... main logic
});
// runMain() handles process.exit(0) or process.exit(1)
```

Reference: `scripts/templates/script-template.mjs` for canonical structure.

---

## 12. Configuration Schema

### 12.1 Glob semantics

- `include` and `exclude` are arrays of **glob patterns**.
- Patterns without a `!` prefix are **positive** (include these paths).
- Patterns with a `!` prefix are **exclusion overrides** (remove these paths even if matched by a positive pattern).
- Resolution order: positive patterns match first → exclusion patterns subtract from the match set → final set is union of remaining positive matches.
- This mirrors `eslint`, `prettier`, and `vitest` conventions.

Example:
```json
{
  "include": ["client/**/*.{ts,tsx}"],
  "exclude": ["!client/**/*.test.tsx", "client/**/__mocks__/**"]
}
```
Result: all `.ts/.tsx` files under `client/` are matched, except `.test.tsx` files (explicit override) and anything under `__mocks__/` directories.

### 12.2 Schema

```json
{
  "include": ["netlify/functions/**/*.cjs", "client/**/*.{ts,tsx}", "shared/**/*.ts"],
  "exclude": [
    "**/node_modules/**",
    "**/dist/**",
    "**/__tests__/**",
    "**/templates/**",
    "**/deprecated*/**",
    "**/*.test.*",
    "**/*.mock.*"
  ],
  "packs": ["netlify-functions", "react-typescript", "node-cjs"],
  "packsDir": "./packs",
  "severityThreshold": "P3",
  "outputPath": "./PRODUCTION-CODEBASE-AUDIT-{date}.md",
  "sprintRulesPath": "./sprintRules.json",
  "reportTemplate": "./REPORT_TEMPLATE.md",
  "astEngine": null
}
```

Notes on schema changes:
- `recommendationEngine` block removed — sprint rules now live in a separate `sprintRules.json` file (per Fix 3). `sprintRulesPath` points to the global config.
- `astEngine: null` means regex-only. Set to `"typescript"` once Phase 2 ships to enable AST rules.
- Per-pack `recommendationEngine` overrides are supported via optional `recommendationEngine` block inside individual pack JSON files; merged into global sprint rules at load time.

---

## 13. Open Questions

1. **Rule pack distribution.** Should packs ship with the agent, or be project-local JSON? Proposal: ship a `default-packs/` directory with the agent, projects override via local config. Simplest path.
2. **AST vs regex tradeoff.** Regex catches ~85% of patterns from the 2026-06-26 audit. AST adds dependency weight (~30MB for `typescript`). Trade-off is acceptable to defer; regex is Phase 1, AST is Phase 2.
3. **False positive rate.** The 2026-06-26 audit flagged some borderline cases (multiple early returns, magic numbers in constants like HTTP status codes). Rule packs need a `whitelist` field per rule:
   ```json
   { "id": "magic-number-rate-limit", "whitelist": { "patterns": ["200|201|204|400|401|403|404|500"] } }
   ```
4. **Recommendation engine calibration.** The 5-sprint sequencing was derived from one project's findings. May need to be retuned as more audits are run.
5. **CI integration vs read-only.** Currently spec'd as read-only + manual report. Should `--diff` mode (Phase 3) be allowed to fail CI on P0 regressions? That's a separate decision.

---

## 14. References

- **PRODUCTION-CODEBASE-AUDIT-2026-06-26.md** — methodology source
- **scripts/lib/CONTRACT.md** — shared modules contract (DRY/SSOT)
- **scripts/templates/script-template.mjs** — canonical script structure
- **timeLaw/LIFECYCLE_LAW_V1.md** — LAW 1-4 for Node scripts
- **CODING-CONTRACT.md** — U1-U6 + F1-F7
- **FRONTEND_LAW_VAULT.md** — LAW 1, 6, 7, 8 for React components
- **AGENTS.md** — Production-Only Test Policy (scope definition)

---

## 15. Decision Log

- **2026-06-26** — Spec doc created. Scope locked: read-only auditor, project-specific packs with framework-agnostic core, co-located in `2026/production_codebase/`. Implementation deferred pending ARCHITECT approval.
- **2026-06-26** — RFC review accepted all 8 fixes. Schema now includes `detectionMethod` (regex/ast-pending/ast), `whitelist`, `requiresLogging` fields; BLESSED scoring layer added as §7; sprint rules extracted to `sprintRules.json` data; `literal_duplication` category distinct from `silent_swallow`; cross-framework validation doc downgraded count-level claims to working-hypothesis status.
- **2026-06-26** — RFC Fix 9 accepted. LIFESTAR↔BLESSED scoping added to §1 Purpose: v1 covers 3/8 LIFESTAR letters (Lean, Explicit, SSOT); 5 letters (Immutable, Findable, Testable, Accessible, Reviewable) explicitly deferred to v2. RFC's two anchor examples (mWritePosition incident, SilverDAD/GOLDAD `kOversampleFactor` drift) unverifiable in current repo — flagged for ARCHITECT review before RFC is logged to SPRINT-LOG.
- **2026-06-26** — JUCE `Oversampling` false-positive integrated as documented methodology gap. Added §3.5 (semantic verifier with `requiresDomainKnowledge` field), §7.7 (multi-site consistency as positive signal), §16 (validation findings & methodology gaps). Spec amendment preserves Phase 1 regex-only implementation; verifier registry deferred to Phase 4+. False-positive post-mortem in §16.1 documents the specific failure mode (regex matcher cannot infer framework constructor semantics) and the mitigation (per-framework semantic verifiers). Tier 1 BLOCKER count for FINALIZER-2 audit corrected 29 → 28 → 27.
- **2026-06-26** — **§7 rewritten**: BLESSED 5-criterion scoring (Beautiful/Logical/Explicit/Standalone/SSOT) replaced by **4-dimension evaluation framework** (Severity / Confidence / Fix Cost / Intent) per architecture review. Key changes: (a) Confidence becomes explicit (0.0-1.0 scale with calibration anchors), enabling §7.6 confidence-aware P0 gate that would have caught the JUCE Oversampling false positive at confidence 0.5 instead of publishing at P0 silently; (b) Fix Cost is relative enum (trivial/small/medium/large/external), not absolute time; (c) Intent is nullable with provenance requirement (no silent default to "unintentional" — bias mitigation); (d) Sprint priority formula `severity × confidence × cost_penalty` replaces severity-only ordering. §3.1 updated to add Confidence evaluator + Priority calculator as Core components. §6 output template updated to surface 4D view + humanReviewRequired flag. §16.1 reframed as "would have been caught at confidence 0.5 under 4D framework."
- **2026-06-26** — **§7 rewritten again (v3)**, **§17 Architecture Layer added**, **§18 Finding DAG added**, **§19 Learning deferred to v2 milestone**. v2's 4D framework still had structural gaps surfaced in architecture review: (1) float confidence (0.0-1.0) created false precision — replaced with categorical buckets A/B/C/D; (2) Fix Cost as time estimate was unreliable from static analysis — replaced with **Structural Scope** (local/module/project/external); (3) Intent as nullable finding field was god-object temptation — moved to separate `annotations.yaml` lifecycle; (4) Priority formula conflated importance and timing — split into Priority (severity + confidence) and Scheduling (priority + scope); (5) flat finding list lacked root-cause layer — added §17 Architecture Layer with cluster detection + rule-pack-declared hypotheses; (6) findings had no relationships — added §18 Finding DAG with inferred edges; (7) positive/negative binary was insufficient — added Neutral class for context-dependent patterns; (8) Confidence + Semantic Verifier were redundant layers — consolidated into single Verification Pipeline in §3.5. §3.1 updated with cluster grouper, architecture hypothesizer, DAG builder. Implementation cost revised 10-13 days → 13.5-16.5 days. Spec evolves from "lint classifier" to "architect assistant."

---

## 16. Validation Findings & Methodology Gaps

This section captures **what we learned by running THE_FINALIZER methodology on real codebases** (N=3 audits: Vortex Realm JWT, GOLDAD, FINALIZER-2). Each finding either confirms a hypothesis or exposes a methodology gap. Gaps are mitigated by spec amendments (cited inline).

### 16.1 False positive: JUCE `Oversampling<float>` exponent semantics

**Severity (legacy)**: methodology gap (P0 false positive in production audit)
**Reframed under 4D (v2)**: this finding would have been published at `severity=P1, confidence=0.5, fixCost=small, humanReviewRequired=true` — because the constructor argument requires framework domain knowledge. Under the 4D framework, it would not have been published as P0 silently.
**Reframed under v3 (categorical)**: this finding would have been routed through the consolidated Verification Pipeline (§3.5). Pipeline stage 2 (domain knowledge check) routes it to a `juce-dsp-oversampling` verifier. The verifier returns `indeterminate` because the constructor's exponent semantics require JUCE domain knowledge. The finding is bucket-downgraded from initial `B` to `C` (manual review). P0 publication never happens; finding is published with `humanReviewRequired: true` and `bucket: "C"`. The categorical bucket makes the gating deterministic — no threshold conversion needed.

**Date**: 2026-06-26

**Affected audit**: `FINALIZER-2-CODEBASE-AUDIT-2026-06-26.md`, Tier 1 BLOCKER #10 (now closed as false positive).

**The pattern that failed**:

```cpp
// Source/PluginProcessor.h:73-76
juce::dsp::Oversampling<float> mOversamplerSlot0 {
    2, 2, juce::dsp::Oversampling<float>::filterHalfBandPolyphaseIIR, true };
```

Audit reasoning: "Constructor uses `{ 2, 2, ... }` — actual factor is 2, not 4. The constant `kOversampleFactor = 4` is wrong."

**Why it's wrong** (verified via JUCE documentation):

```cpp
// juce::dsp::Oversampling<T>::Oversampling(
//     size_t numChannels,    // first arg
//     size_t factor,         // second arg — EXPONENT, not literal factor
//     FilterType type,
//     bool isMaxQuality = true,
//     bool useIntegerLatency = false
// )
//
// Documentation: "factor — the processing will perform 2^factor times oversampling"
```

So `Oversampling<float>{ 2, 2, ... }` = `numChannels=2, factor=2` → `2^2 = 4×` oversampling. All five sites in the codebase agree on 4×:
- `PluginProcessor.h:69` — comment: `"Shared 4× oversampler"`
- `PluginProcessor.h:73-76` — constructor `(2, 2, ...)` = `2^2 = 4×`
- `PluginColdState.cpp:16` — `kOversampleFactor { 4 }`
- `dsp_silverdad/DSP/DSPConstants.h:29` — `kOversampleFactor { 4 }`
- `ClarityStage.cpp:100-101` — `osBlock.getNumSamples() / 4`, `std::pow(0.9995, ... / 4.0)`

The matcher correctly identified that the same value appeared in multiple places. The matcher **incorrectly interpreted this as drift** rather than as consistent SSOT propagation.

**Detection failure mode**: regex pattern matcher flagged a numerical "inconsistency" that was actually consistency under non-obvious framework semantics. The matcher had no way to express confidence, so it published at P0 with implicit confidence 1.0.

**4D-framework post-mortem**: under the new evaluation framework, this finding would have been scored `confidence=0.5` (requires JUCE domain knowledge to disambiguate). The confidence-aware P0 gate (§7.6) would have flagged it for human review. The owning engineer would have read the JUCE docs (30 seconds), confirmed the false positive, and removed it from the sprint — without the matcher ever publishing it as a confirmed P0.

**Mitigation (applied)**:

1. **§3.1 added** Confidence evaluator to core responsibilities
2. **§3.5 added** `requiresDomainKnowledge` + `semanticDomain` fields — rules that need framework semantics declare themselves
3. **§7.2 added** Confidence dimension (0.0–1.0) with calibration scale
4. **§7.6 added** Confidence-aware P0 gate — confidence < 0.7 at P0 → humanReviewRequired flag
5. **§7.7 added** (preserved from prior amendment): matcher rule for multi-site consistency
6. **Tier 3 finding kept** (now under 4D): rename `kOversampleFactor { 4 }` to `kOversampleRatio { 4 }` for clarity. Under 4D this becomes `severity=P3, confidence=1.0, fixCost=trivial` — definitely actionable.

**What this teaches for THE_FINALIZER's spec**:

| Lesson | Spec amendment |
|--------|----------------|
| Regex matchers cannot infer framework-specific constructor semantics | Verification Pipeline (§3.5) with stage-2 domain knowledge check |
| Matchers need a way to express "I'm not sure" | Categorical confidence A/B/C/D + P0 gate (§7.6) |
| Float confidence creates false precision | Categorical buckets (§7.2) |
| Cost should not reduce importance | Priority/Scheduling split (§7.5) |
| Intent on findings is god-object temptation | Moved to annotations.yaml (§7.9) |
| Flat finding list lacks root cause | Architecture Layer with clusters + hypotheses (§17) |
| Findings have relationships | Finding DAG with inferred edges (§18) |
| Positive/negative binary is insufficient | Three classes: positive / neutral / negative (§7.7) |
| Confidence + Semantic Verifier are redundant | Consolidated into Verification Pipeline (§3.5) | |
| Verification must include domain-knowledge check | Semantic Verifier component (§3.1) |
| Naming can hide framework semantics (`kOversampleFactor` is misleading) | Out of scope — naming is human concern |

**Cost of mitigation**: per-framework verifier implementation is ~10-50 lines of domain logic. Cost is bounded; benefit is reduced false-positive rate on P0/P1 findings. Implementation deferred to Phase 4+ (not Phase 1).

### 16.2 Confirmed category-level patterns (3/3 validations)

These hypotheses held across all three production audits:

| Pattern | Vortex | GOLDAD | FINALIZER-2 | Status |
|---------|-------:|-------:|------------:|:------:|
| Magic numbers without named constants | 50 | 47 | 60+ | **CONFIRMED** |
| SSOT residue (constant duplicated across files) | 6 | 3 | 5 | **CONFIRMED** |
| Magic string duplication | 7 | 7 | 10 | **CONFIRMED** |
| Top 5 files = ~40% of hacks (Pareto concentration) | ~40% | ~40% | ~40% | **CONFIRMED** (structural, not THE_FINALIZER's discovery) |
| `assert()` vs `jassert` family-specific to C++/JUCE | 0 | 4 | 8 | **CONFIRMED** |
| Foreign-code-embedding scales P0 count | n/a | n/a | +20 P0 | **WORKING HYPOTHESIS** (1 sample) |

### 16.3 Refuted claims

| Claim | Source | Status |
|-------|--------|--------|
| "~6 P0 BLOCKERS is universal baseline" | N=2 validation doc | **REFUTED at N=3** — P0 count tracks foreign-code-embedding, not maturity |
| Fix 9 SilverDAD-vs-GOLDAD `kOversampleFactor` drift | RFC Fix 9 | **REFUTED** — no such drift exists; replaced by declared-vs-actual-exponent misread (see §16.1) |
| Fix 9 `mWritePosition` Sprint 24 incident | RFC Fix 9 | **UNVERIFIABLE** — no references in current repo |

### 16.4 Recommendation engine calibration status

The recommendation engine's sprint grouping logic (5-sprint default) was originally derived from one project's audit (Vortex) and treated as a working hypothesis. After N=3:

- **Magic numbers / SSOT / strings / type hygiene / law enforcement** — the 5-sprint structure maps well to the Vortex/GOLDAD pattern. Promising for JS/TS + C++/JUCE.
- **Foreign-code-embedding** — not represented in the original 5-sprint structure. FINALIZER-2 has 20+ P0 items that don't fit any of the 5 sprints cleanly. **Needs a 6th sprint category or a special-case flag.**

**Recommendation**: add Sprint 0 (or Sprint 6) for `foreign-code-embedding` cleanup. This is a separate category from the existing 5 because it has different fix shape (extract shared library OR remove foreign code) vs the existing sprints (which are all in-codebase refactors).

### 16.5 Implementation impact summary

| Spec section | Original | Post-amendment | Reason |
|--------------|----------|----------------|--------|
| Phase 1 file list (§9.1) | 10 files | 10 files (unchanged) | Verifier registry deferred to Phase 4+ |
| Phase 1 schema (§3.2) | `detectionMethod`, `whitelist`, `requiresLogging` | + `requiresDomainKnowledge`, `semanticDomain` | Mitigates false positives |
| Phase 2 AST rules | — | — | Unchanged; AST independent of semantic verifiers |
| Phase 4 cross-framework packs | Add new pack files | + per-framework verifier under `scripts/finalizer/verifiers/` | Adds per-framework domain knowledge |
| Total implementation cost | 8-11 days (estimate) | 10-13 days (revised) | +2 days for verifier framework + 3 initial verifiers (juce-dsp, react-hooks, netlify-handler) |

### 16.6 Summary of changes from validation findings

1. ✅ §3.1: added Semantic Verifier to core responsibilities
2. ✅ §3.5 (new): `requiresDomainKnowledge` + `semanticDomain` schema fields
3. ✅ §7.7 (new): multi-site consistency is a positive signal
4. ✅ §16 (new): this section — full validation findings record
5. ✅ §15 Decision Log: appended 2026-06-26 entry recording this amendment
6. ✅ Cross-framework validation doc updated to reflect N=3 findings + false positive correction
7. ✅ FINALIZER-2 audit doc updated: Tier 1 count 29 → 27; finding #10 closed as false positive

**Status (v2 → v3)**: spec evolved from "lint classifier with 4D evaluation" to "architect assistant with hypothesis layer + finding DAG." v3 changes (see §17, §18) close the v2 gaps identified during architecture review: false precision in confidence, conflated priority/scheduling, god-object intent field, missing root-cause layer, missing finding relationships, binary positive/negative classification, no learning. Each v3 change is justified by either a v2 method failure (JUCE Oversampling) or an architecture review finding.

---

## 17. Architecture Layer — Cluster → Architecture → Systemic Fix

A flat list of N findings is a **symptom list**. A software architect thinks in **clusters** (groups of findings sharing a root cause) and **systemic fixes** (changes that eliminate many findings at once). Without this layer, THE_FINALIZER is a *lint classifier*. With it, it is an *architect assistant*.

### 17.1 Five-layer hierarchy (v3)

```
Pattern (rule in pack)
   ↓
Finding (instance of pattern at file:line)
   ↓
Cluster (group of findings sharing category + co-location + architecture hypothesis)
   ↓
Architecture (declared hypothesis about why cluster exists)
   ↓
Sprint (recommended sequencing of cluster-level systemic fixes)
```

The first two layers (Pattern, Finding) are v1/v2 functionality. v3 adds **Cluster**, **Architecture**, and the **Sprint** now operates on clusters not findings.

### 17.2 Cluster detection (automated)

Clusters are computed from finding attributes:

- **Same category** — findings matching the same `category` field (e.g. all `magic_number`)
- **Co-location** — findings within N files of each other (default N=5, configurable)
- **Pattern signature** — set of category IDs that co-occur in the cluster

The cluster grouper emits clusters without names. Naming is the next step.

### 17.3 Architecture hypothesis (rule pack declares)

Naming the cluster and proposing a systemic fix is **declared in rule packs**, not computed by the analyzer. The pack author understands the codebase domain and writes hypotheses like:

```json
{
  "clusterRules": [
    {
      "id": "config-drift",
      "signature": ["magic-number", "duplicate-literal", "env-fallback-url"],
      "hypothesis": "Missing central constants module — fixes at one location can eliminate 60%+ of cluster findings",
      "structuralScope": "project",
      "systemicFix": "Create `src/constants.h` consolidating 12 magic-number declarations"
    },
    {
      "id": "react-effect-anti-pattern",
      "signature": ["use-effect-derived-state", "use-effect-empty-deps"],
      "hypothesis": "Team is not familiar with React 18 hooks mental model — training gap, not per-finding fix gap",
      "structuralScope": "project",
      "systemicFix": "Add `docs/HOOKS_GUIDE.md` and code review checklist item"
    },
    {
      "id": "foreign-code-embedding",
      "signature": ["namespace-foreign-product", "import-foreign-source-tree"],
      "hypothesis": "Multiple products share one binary when they should be separate libraries",
      "structuralScope": "external",
      "systemicFix": "Extract foreign product source trees into separate shared libraries with version pinning"
    }
  ]
}
```

When the matcher detects findings matching a `signature`, it emits the corresponding `hypothesis` and `systemicFix` in the Architecture Hypotheses section of the report.

### 17.4 Cluster threshold

A cluster is only emitted in the Architecture Hypotheses section when **at least N findings match** the signature. Default N=3. Smaller clusters appear in the regular findings list but don't get an architectural hypothesis — they're noise rather than systemic.

### 17.5 Why this changes everything

**Without architecture layer:**
> 108 findings across 47 files. Top categories: magic_number (50), duplicate_literal (40), env_fallback_url (18). Recommended: fix in priority order over 8 sprints.

**With architecture layer:**
> 108 findings across 47 files. Detected 7 clusters with declared hypotheses:
> - **Configuration Drift** (88 findings, 81% of total) — Hypothesis: missing central constants module. Single systemic fix: create `src/constants.h`. Estimated impact: eliminates 70+ findings.
> - **Hook Mental Model Gap** (3 findings) — Hypothesis: training gap. Systemic fix: docs + review checklist.
> - **Foreign Code Embedding** (15 findings) — Hypothesis: products should be libraries. Systemic fix: extract to shared libs.
>
> Recommendation: prioritize the 3 systemic fixes over per-finding fixes. Sprint 1: create `constants.h` (eliminates 70+ findings). Sprint 2: extract foreign libraries. Sprint 3: training.

The shift from "fix 108 things" to "make 3 architectural changes" is what makes THE_FINALIZER an *architect assistant*, not a *lint classifier*.

### 17.6 Cluster emission format

```markdown
## Architecture Hypotheses

### Cluster: Configuration Drift
- **Signature**: magic-number, duplicate-literal, env-fallback-url
- **Findings matching**: 88 (across 41 files)
- **Hypothesis**: Missing central constants module — fixes at one location can eliminate 60%+ of cluster findings
- **Systemic Fix**: Create `src/constants.h` consolidating 12 magic-number declarations
- **Structural Scope**: project
- **Estimated Impact**: ~70 findings eliminated by single systemic fix
- **Per-finding fixes still relevant for**: 18 findings (where extraction isn't feasible)

### Cluster: Foreign Code Embedding
- **Signature**: namespace-foreign-product, import-foreign-source-tree
- **Findings matching**: 15
- **Hypothesis**: Multiple products share one binary when they should be separate libraries
- **Systemic Fix**: Extract foreign product source trees into separate shared libraries with version pinning
- **Structural Scope**: external
- **Estimated Impact**: depends on extraction scope
```

### 17.7 Implementation cost (v3)

| Component | Estimated |
|-----------|----------:|
| Cluster grouper | 0.5 day (similarity scoring + co-location check) |
| Cluster-rule loader (extend rule pack schema) | 0.5 day |
| Architecture hypothesizer | 0.25 day (lookup + emit) |
| Report section generator | 0.5 day |
| **Total v3 layer additions** | **~2 days** |

Adds to existing implementation estimate (10-13 days from v2) → **12-15 days** for full v3.

---

## 18. Finding DAG (Directed Acyclic Graph)

Findings are not isolated. They form relationships: a magic-number at one location may *cause* a duplicate-literal at another; both may *share* a root cause. A flat list loses this structure. The finding DAG preserves it.

### 18.1 Edge types

| Type | Semantics | When inferred |
|------|-----------|---------------|
| `causes` | Finding X at file:line directly causes finding Y (X must be fixed first or Y cannot be evaluated) | Rule pack declares (`from`, `to`, optional `condition`) |
| `shares-root-cause` | Findings X and Y share a common architectural cause (e.g. both stem from missing SSOT module) | Cluster co-membership; rule pack declares |
| `compounds` | X and Y together are worse than each alone (fixing only one makes things worse) | Rule pack declares |
| `blocks` | Until X is fixed, Y cannot be meaningfully addressed | Rule pack declares |

### 18.2 Rule pack declaration

```json
{
  "dagRules": [
    {
      "from": "magic-number-rate-limit",
      "to": "duplicate-literal-same-value",
      "edgeType": "shares-root-cause",
      "condition": "same-file"
    },
    {
      "from": "if-false-block",
      "to": "commented-code-block",
      "edgeType": "causes",
      "rationale": "Dead code with disabled lint often precedes commented-out alternatives"
    },
    {
      "from": "use-effect-derived-state",
      "to": "fetch-no-abort-signal",
      "edgeType": "compounds",
      "rationale": "Both signal missing React 18 mental model; fixing one without the other leaves the underlying confusion"
    }
  ]
}
```

### 18.3 Inferred edges (v1 scope, automatic)

Without rule pack declarations, the DAG builder can infer some edges automatically:

- **Cluster co-membership → `shares-root-cause`**: any two findings in the same cluster get a `shares-root-cause` edge between them.
- **Same file + same category → `causes`** (weaker): findings of the same category in the same file are likely to interact.
- **Same file + cascade category → `causes`**: `magic-number` and `duplicate-literal` in the same file get a `causes` edge (the magic-number probably *becomes* the duplicate literal).

### 18.4 v1 scope: inferred edges only

For v1, **only inferred edges are computed**. Declared `dagRules` in rule packs are **forward-compatible** (the schema accepts them) but the runtime only emits inferred edges in v1. Declared edges ship in v2 when telemetry shows which declared edges are actually useful vs. noise.

Reasoning: declared edges are speculative without validation. Inferred edges have at least one observable signal (co-membership, co-location, cascade). Adding declared edges prematurely risks polluting the DAG with author-opinion that may not generalize.

### 18.5 DAG output format

```markdown
## Finding DAG

### Cluster: Configuration Drift (88 findings)

[X1: magic-number@config.ts:42]──shares-root-cause──[Y1: duplicate-literal@config.ts:42]
[X1]──causes──[Z1: env-fallback-url@config.ts:43]
[Y2: magic-number@auth.ts:88]──same-file-causes──[Y3: duplicate-literal@auth.ts:88]

Legend:
  X1, Y1, Z1 etc: finding IDs (auto-assigned per scan)
  edges: declared in clusterRules OR inferred from co-location/cascade rules
```

For large clusters (>20 findings), the DAG is summarized to top-N most-connected nodes rather than fully enumerated.

### 18.6 Implementation cost (v3)

| Component | Estimated |
|-----------|----------:|
| DAG builder | 0.5 day (graph construction + traversal) |
| Inferred edge rules | 0.25 day |
| Report DAG section generator | 0.5 day (with summarization for large clusters) |
| **Total v3 DAG additions** | **~1.25 days** |

Adds to v3 total → **~13.5-16.5 days** for full v3 implementation.

---

## 19. Learning from Telemetry (v2 milestone, deferred)

A static analyzer that runs the same rules repeatedly should learn from history: rules with high false-positive rate should auto-calibrate to lower confidence; rules that consistently surface systemic issues should be promoted. **This is not v1 scope.**

### 19.1 Why deferred

- **Statistical power**: a rule needs minimum ~20 invocations across audits before its false-positive rate can be meaningfully estimated. Below this, calibration is noise.
- **Storage infrastructure**: requires telemetry store (per-project or aggregated).
- **Drift detection**: false-positive rate changes over time as codebases evolve; calibration needs re-runs.
- **Calibration algorithm**: requires validation against multiple project sizes, languages, and maturities.

### 19.2 v2 prerequisite

Before v2 learning is implementable:
- At least 10+ audits per active rule pack across diverse projects
- Telemetry schema documented (which events to capture: invocation count, refuted count, suppressed count, sprint-inclusion count, fix-completion rate)
- Storage backend chosen (local JSONL vs. central DB)
- Calibration algorithm validated against historical audit data

### 19.3 What v2 learning looks like (preview)

A rule with telemetry showing 80% false-positive rate across 50 invocations would auto-recommend `confidence: "D"` default for the rule. The pack author can override this in the rule definition, but the default reflects observed reality rather than author intuition.

This is the **only** way to address the **base-rate problem**: rules like `console.log` will always have low precision in any non-trivial codebase, and author-set confidence will be systematically miscalibrated without telemetry.
