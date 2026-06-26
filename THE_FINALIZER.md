# THE_FINALIZER — Production Codebase Audit Agent

**Part of:** GABRIEL (`GABRIEL.md`)
**Status:** Design specification
**Scope:** Pattern-driven hack/boilerplate detection · read-only

---

## 1. Purpose

Scans any codebase → produces markdown report distinguishing:

- **HACKS** — anti-patterns to remove
- **BOILERPLATE** — clean patterns to replicate
- **RECOMMENDATIONS** — sequenced, prioritised, file:line-cited

**Non-goals:** Not a code fixer · not a CI gate · not a replacement for ESLint/typecheckers.

---

## 2. Architecture

```
THE_FINALIZER
├── Core (framework-agnostic)
│   ├── File discovery · Rule loader · Pattern matcher
│   ├── Verification pipeline · Findings aggregator
│   └── Report generator · Sprint grouping engine
├── Rule Packs (JSON — project-specific)
│   └── packs/{netlify-functions,react-typescript,node-cjs,cpp-juce}.json
└── Configuration (finalizer.config.json)
```

### 2.1 Core Responsibilities

| Component | Responsibility |
|-----------|----------------|
| **Pattern matcher** | Regex for `detectionMethod: "regex"`; skip `ast-pending` |
| **Verification pipeline** | Pattern strength → domain knowledge → consistency → confidence bucket |
| **Priority calculator** | `severity_rank × confidence_weight` |
| **Cluster grouper** | Co-location + category similarity → root-cause clusters |
| **DAG builder** | Inferred edges between findings |

### 2.2 Verification Pipeline

Every candidate finding runs through 4 stages:

1. **Pattern strength** — clean match? offset drift? outputs initial bucket
2. **Domain knowledge** — `semanticDomain` declared? route to verifier → confirmed/refuted/indeterminate
3. **Multi-site consistency** — `ssot_violation` with same name+value = boilerplate, not drift
4. **Confidence assignment** — final bucket A/B/C/D

**Buckets:**
| Bucket | Meaning |
|--------|---------|
| **A** (auto-confirm) | Clean match, no ambiguity |
| **B** (quick review) | Clean match, heuristic category |
| **C** (manual review) | Domain knowledge required |
| **D** (suppress) | Likely false positive |

**Mandatory:** P0 at C or D → `humanReviewRequired: true`.

---

## 3. Evaluation Framework

### 3.1 Three Dimensions

| Dimension | Question | Type |
|-----------|----------|------|
| **Severity** | If correct, how bad is impact? | P0/P1/P2/P3 |
| **Confidence** | Is this a real issue? | A/B/C/D |
| **Structural Scope** | Where does the fix live? | local/module/project/external |

**Priority** = `severity_rank × confidence_weight` (importance, not time)
**Scheduling** = `Priority × scope_delay_factor` (timing, not importance)

| Severity rank | P0=4, P1=3, P2=2, P3=1 |
|----------------|--------------------------|
| Confidence weight | A=1.0, B=0.9, C=0.6, D=0.2 |
| Scope delay | local=1.0, module=1.0, project=0.6, external=0.3 |

### 3.2 Severity Tiers

| Tier | Meaning | Example |
|------|---------|---------|
| **P0** | Ship-blocker | `if (false)` block, silent `catch {}` |
| **P1** | Correctness | Unprotected `JSON.parse`, hardcoded URL |
| **P2** | Convention | Magic numbers, repeated literals |
| **P3** | Style | `console.log` residue, `TODO` comments |

### 3.3 Finding Classification

| Class | Meaning | Action |
|-------|---------|--------|
| `positive` | Exemplary code | "Boilerplate" section |
| `neutral` | Context-dependent | "Neutral Observations" — no sprint planning |
| `negative` | Hack/anti-pattern | "Hacks" section + sprint planning |

---

## 4. Rule Pack Schema

```json
{
  "packName": "example",
  "appliesTo": { "extensions": [".ts", ".tsx"], "pathPrefixes": ["src/"] },
  "negative": [
    {
      "id": "console-log-prod",
      "category": "debug_in_prod",
      "severity": "P3",
      "detectionMethod": "regex",
      "pattern": "^\\s*console\\.(log|debug)\\(",
      "scope": "anywhere",
      "whitelist": { "patterns": [] },
      "rationale": "Bare console bypasses logger SSOT"
    }
  ],
  "positive": [
    { "id": "logger-shared", "pattern": "logger\\.(info|warn|error)", "scope": "anywhere" }
  ],
  "neutral": [
    { "id": "uses-std-variant", "pattern": "\\bstd::variant\\b", "scope": "anywhere" }
  ],
  "dagRules": [{ "from": "...", "to": "...", "edgeType": "shares-root-cause" }],
  "clusterRules": [{ "id": "...", "signature": ["..."], "hypothesis": "...", "structuralScope": "project" }]
}
```

### 4.1 Detection Method

| Value | Matcher behaviour |
|-------|------------------|
| `"regex"` | Evaluated every scan |
| `"ast-pending"` | Skipped; listed in `--stats` as deferred |
| `"ast"` | Evaluated when `astEngine` configured |

---

## 5. Pattern Catalogue

### 5.1 HACK patterns

| Rule ID | Category | Pattern sketch | Severity |
|---------|----------|---------------|----------|
| `console-log-prod` | debug_in_prod | `console\.(log\|debug)\(` | P3 |
| `catch-empty-silent` | silent_swallow | `catch\s*(\(\s*\))?\s*\{\s*\}` | P0 |
| `json-parse-unprotected` | unprotected_json | `JSON\.parse\(` | P1 |
| `if-false-block` | dead_code | `if\s*\(\s*false\s*\)\s*\{` | P0 |
| `magic-number` | magic_number | bare numeric literals in non-constant context | P2 |
| `env-fallback-url` | env_fallback | `process\.env\.\w+\s*\|\|\s*["']https?://` | P2 |
| `as-any-cast` | type_anti_pattern | `as\s+any\b` | P2 |
| `fetch-no-abort` | no_abort | `fetch\([^)]*\)` without `signal:` | P1 |
| `eval-constructor` | dangerous_code | `eval\(` \| `new Function\(` | P0 |
| `literal-duplication` | literal_duplication | Same value in ≥3 files without shared constant | P1 |
| `use-effect-derived-state` | law_6_violation | useEffect for derived state (AST rule) | P2 |

### 5.2 BOILERPLATE patterns

| Rule ID | Pattern | Scope |
|---------|---------|-------|
| `logger-shared` | `logger\.(info\|warn\|error\|debug)` | any |
| `abort-signal-fetch` | `AbortSignal.timeout\|controller.signal` | any |
| `cleanup-useeffect` | `return () =>` in useEffect | React |
| `usememo-derived` | `useMemo` for derived values | React |
| `upper-snake-const` | `const [A-Z_]+ =` at module top | any |

---

## 6. Output Format

```markdown
# Production Codebase Audit — [date]

## Scope Stats
[file count, finding counts by class, confidence histogram]

## Architecture Hypotheses
[cluster → hypothesis → systemic fix → estimated impact]

## Negative Findings — Hacks

### P0 BLOCKERS
| File:Line | Rule | Bucket | Scope | Priority |
|----------|------|--------|-------|---------:|
| ...      | ...  | ...    | ...   | ...      |

### P0 Requiring Human Review
| File:Line | Rule | Bucket | Reason |
|----------|------|--------|--------|
| ...      | ...  | ...    | ...    |

### P1 HIGH · P2 MEDIUM · P3 LOW

## Neutral Observations · Positive Patterns · Finding DAG

## Summary
[counts, distribution, top files, top clusters]

## Recommendations — Sequenced Sprints
[sprint plan by priority + scheduling]
```

---

## 7. CLI

```bash
npm run finalizer -- --include "src/**" --severity P0,P1
npm run finalizer -- --hacks-only --output report.md
npm run finalizer -- --diff baseline.md
```

---

## 8. Cluster & DAG

### 8.1 Architecture Layer

**Five-layer hierarchy:** Pattern → Finding → Cluster → Architecture → Sprint

Clusters detected by co-location + category signature. Hypotheses declared in rule packs.

```
Cluster: Configuration Drift (88 findings)
  Hypothesis: Missing central constants module
  Systemic Fix: Create src/constants.h
  Estimated Impact: ~70 findings eliminated
```

### 8.2 Finding DAG

**Edge types:** `causes` · `shares-root-cause` · `compounds` · `blocks`

Inferred edges (v1): cluster co-membership → `shares-root-cause`

### 8.3 Sprint Taxonomy

| Sprint | Inclusion |
|--------|-----------|
| Sprint 1 | P0 + {A,B} confidence |
| Sprint 1-Review | P0 + {C,D} confidence |
| Sprint 2 | `literal_duplication` category |
| Sprint 3 | `magic_number` category |
| Sprint 4 | `type_anti_pattern` category |
| Sprint 5 | `law_*_violation` category |

---

## 9. Annotations (External)

`annotations.yaml` — human-controlled, NOT produced by THE_FINALIZER:

```yaml
annotations:
  - findingId: "if-false-block@Maintenance.tsx:433"
    intent: intentional
    source: engineer:indraqadarsih
    rationale: "Benchmark branch kept for A/B comparison"
```

---

**End of THE_FINALIZER** · Spec v1.0
