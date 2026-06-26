# GABRIEL — Governed Agent-Based Risk Identification & Evidence Loop

**Status:** Canonical · **Date:** 2026-06-27 · **Members:** 3 active, 1 pending
**Framework Version:** 1.0 · **Score:** 9.8/10

**GABRIEL** is an independent, evidence-driven audit framework composed of specialized out-of-band agents that identify project risks and report verified findings to the ARCHITECT. GABRIEL never modifies artifacts, approves releases, or makes engineering decisions.

> **"Loop" means continuous audit cycle** — GABRIEL runs at phase transitions to collect fresh evidence. It is not an execution loop that acts on the project. GABRIEL observes, reports, and stops.

---

## Philosophy

Independent Out-of-Band auditors. Never participate in design, never modify code, never approve releases. Establish facts from different viewpoints, hand decisions to ARCHITECT.

**Shared contract:** Out-of-Band · stateless · read-only · evidence-backed · never auto-remediate · never gate · never approve · hand to ARCHITECT.

---

## Global Definitions

### Evidence Hierarchy

Every finding must cite evidence. Tier determines weight.

| Tier | Source | Example |
|---|---|---|
| **1 — Direct** | Filesystem, git, binary inspection | `SPEC.md:42`, `git log --oneline` |
| **2 — Build** | Build logs, CI logs, compile artifacts | `cmake --build output` |
| **3 — Visual** | Screenshots, recordings, UI captures | DAW screenshot |
| **4 — Verbal** | Conversation, user reports, logs | crash report from user |
| **5 — Assumption** | Inference without source | Never acceptable for findings |

Evidence tier goes into every finding's `Evidence:` field.

### Confidence Scoring

| Level | Meaning |
|---|---|
| **High** | Direct evidence (Tier 1), multiple corroborating sources |
| **Medium** | Partial evidence (Tier 2–3), some inference required |
| **Low** | Inference only (Tier 4), single source, plausible alternative exists |

### Severity Normalization

All auditors map to a global severity scale. Cross-auditor comparison is meaningful.

| Global | SCAFFOLD | RELEASE | PREMORTEM | BUGHUNTER |
|---|---|---|---|---|
| **Blocking / Critical** | Blocking | Blocking | Critical | Critical |
| **Should-Fix / Major** | Should-Fix | Should Release-Fix | Major | Major |
| **Note / Minor** | Note | Note | Minor | Minor |
| **Cosmetic** | — | — | Cosmetic | Cosmetic |
| **Unknown** | — | — | Unknown | Unknown |

### Conflict Resolution

When multiple auditors produce findings on the same subject:

- **ARCHITECT resolves.** Auditors never negotiate, vote, or defer to each other.
- No cross-auditor consensus mechanism.
- ARCHITECT reads both reports, decides.
- This preserves independence.

### Audit Identity

Every report includes this header:

```
Audit ID:      <UUID or timestamp-based>
Target:        <project path>
Date:          <ISO 8601>
Tool Version:  <auditor name + version if applicable>
Framework Ver: <GABRIEL version>
Phase:         <determined phase>
```

Enables cross-time comparison.

### Reproducibility Standard

Every finding includes:

```
Observed:  <what is actually there>
Expected:  <what should be true>
Evidence:  <file:line + quote | path + hash | screenshot>
```

Observed ≠ Expected is the finding. Evidence justifies Observed.

---

## Phase FSM

Artifact presence determines phase. Linear progression, no skips forward.

```
Scaffold ──► Development ──► Pre-release ──► Release ──► Post-build ──► Ready to Ship ──► Published
```

### Entry Conditions

| Phase | Entry condition |
|---|---|
| **Scaffold** | `SPEC.md` missing |
| **Development** | `SPEC.md` exists, `CHANGELOG.md` missing |
| **Pre-release** | `CHANGELOG.md` exists, no Git tag matching latest version |
| **Release** | Git tag == latest CHANGELOG version |
| **Post-build** | Release + `build/` contains distributables |
| **Ready to Ship** | Post-build + `deploy/` exists + artifacts signed/notarized |
| **Published** | deploy/ published + release notes + registry/website updated |

### Regression Rule

Later artifacts removed → phase regresses automatically.

- CHANGELOG deleted → Development
- SPEC.md deleted → Scaffold
- Git tag removed → Pre-release
- Signed artifacts removed → Post-build
- deploy/ deleted → Release

No manual phase reset. Artifacts drive state.

### Non-Standard Locations

GABRIEL scans common paths. Marker satisfied if found anywhere:

- `CHANGELOG.md` · `CHANGELOG` · `deploy/CHANGELOG.md`
- `build/` · `build_osx_ninja/` · `build_win/`
- `deploy/` · `staging_release/`

### UNKNOWN Propagation

- UNKNOWN never upgrades phase. (Cannot confirm tag → cannot claim Release.)
- UNKNOWN may delay certainty. (Note the gap, proceed with available evidence.)
- UNKNOWN is never interpreted as PASS. (Absence of confirmation ≠ confirmation of absence.)

---

## GABRIEL Meta-Audit

Entry point. Determines phase + completeness + applicable auditors + phase gaps.

### When to Invoke

- First contact with a new project
- Unsure about project state
- Transitioning between phases

### Meta Audit Exit Status

| Status | Meaning |
|---|---|
| **COMPLETE** | All phase markers readable, phase determined with High confidence |
| **PARTIAL** | Some markers UNKNOWN, phase determined with Medium confidence |
| **INCOMPLETE** | Too many markers blocked, phase cannot be determined |

### Format

```
PROJECT AUDIT — <name>
Audit ID:   <UUID>
Target:     <path>
Date:       <ISO 8601>
Phase:      <determined phase>  [confidence: High | Medium | Low]

## Phase Determination
  SPEC.md:      <exists | missing | UNKNOWN>  (at: <path>)
  CHANGELOG.md: <exists | missing | UNKNOWN>  (at: <path>)
  Git tag:      <tag | none | UNKNOWN>  (matches: <yes | no | UNKNOWN>)
  build/:       <exists | missing>  (distributables: <yes | no>)
  deploy/:      <exists | missing>  (signed: <yes | no | UNKNOWN>)

## Phase Completeness
  Required for <phase>:
    [x] <artifact> — present
    [ ] <artifact> — MISSING

  Missing for next-phase (<next>):
    [ ] <artifact>

## Applicable Auditors
  @scaffold_checker    — <applicable | not applicable>
  @blackbox_bughunter  — <applicable | not applicable>
  @release_auditor     — <applicable | not applicable>
  @premortem           — <applicable | not applicable>

## Phase Gaps
  GABRIEL-GAP-001 | <gap>
    Evidence:   <file:line or "not found">
    Expected:   <what should be true>
    Impact:     <why it matters>
    Evidence Tier: <1-5>
    Confidence: <High | Medium | Low>
```

### Auditor Phase Coverage

| Phase | SCAFFOLD | BUGHUNTER | RELEASE | PREMORTEM |
|---|---|---|---|---|
| Scaffold | limited | — | — | — |
| Development | ✅ | ✅ | — | — |
| Pre-release | ✅ | ✅ | — | ✅ |
| Release | ✅ | ✅ | — | ✅ |
| Post-build | — | — | ✅ | ✅ |
| Ready to Ship | — | — | ✅ | — |
| Published | — | — | — | — |

---

## Independence Principle

No GABRIEL member reads another member's findings. Each reports only to ARCHITECT. ARCHITECT is sole synthesis point and decision authority. This separates GABRIEL from pipeline agents.

---

## Authority Boundary

| Member | Question | Perspective |
|---|---|---|
| **SCAFFOLD_CHECKER** | Is the implementation coherent with canonical artifacts? | Inside-out |
| **BLACKBOX_BUGHUNTER** | Can the product be broken by an informed user? | Outside-in |
| **RELEASE_AUDITOR** | Is the release artifact operationally complete? | Artifact |
| **PREMORTEM** | What could go wrong if we ship this? | Failure-mode |

---

## Invocation

```
@gabriel              <target>   — meta-audit (phase + completeness)
@scaffold_checker    <target>   — inside-out coherency
@blackbox_bughunter  <target>   — hack/boilerplate pattern audit
@release_auditor     <target>   — artifact completeness
@premortem           <target>   — failure-mode analysis
```

### Machine-Readable Output (optional)

Every report written to disk should also produce a companion JSON:

```json
{
  "auditId": "<UUID>",
  "target": "<path>",
  "date": "<ISO 8601>",
  "phase": "<phase>",
  "phaseConfidence": "<High | Medium | Low>",
  "metaStatus": "<COMPLETE | PARTIAL | INCOMPLETE>",
  "auditors": [
    {
      "name": "scaffold_checker",
      "exitStatus": "PASS | PASS WITH NOTES | REVIEW REQUIRED | INCOMPLETE",
      "blocking": 0,
      "shouldFix": 2,
      "note": 4,
      "findings": [
        {
          "id": "F2-SA-001",
          "severity": "Should-Fix",
          "globalSeverity": "Should-Fix",
          "category": "magic-number",
          "impact": "Maintainability",
          "evidenceTier": 1,
          "confidence": "High",
          "file": "<path>",
          "line": 42,
          "observed": "<...>",
          "expected": "<...>",
          "evidence": "<...>",
          "source": "JRENG §5"
        }
      ]
    }
  ],
  "gabrielGaps": [],
  "conflicts": []
}
```

Human report always primary. JSON is CI/optical-archive companion.

---

# I. SCAFFOLD_CHECKER

**Spec:** `scaffold_checker.md`

Authority: MANIFESTO · ARCHITECTURE · RFC · PLAN · SPEC · Mockups · Repository
Never trusts: conversation history · sprint logs · memory · assumptions
Output: Docs↔Code coherency · Structural · Build-sequence · Mockup · BLESSED
Exit Status: PASS · PASS WITH NOTES · REVIEW REQUIRED · INCOMPLETE

---

# II. BLACKBOX_BUGHUNTER

**Spec:** `THE_FINALIZER.md` (from Vortex-realm-JWT-2 production codebase)

Produces: HACKS · BOILERPLATE · RECOMMENDATIONS — sequenced, prioritised, file:line-cited.
Output: per `THE_FINALIZER.md` §6 audit report structure.

---

# III. RELEASE_AUDITOR

## Philosophy

A build that compiles is not releasable. Zero bugs is not releasable. Release is ready when every required artifact exists, is consistent, and is fit for distribution.

## Artifact Completeness Principle

**Absence is a release defect.** Binary correct, no release notes → finding. Installer correct, checksum missing → finding.

## Scope

Packaged binaries · installers · release bundles · version metadata · changelog · release notes · migration guides · user docs · licensing · manifests · checksums · signatures · presets · factory content.

## Forbidden

Source code quality · implementation style · architectural decisions · RFC/PLAN correctness · bug severity.

## Core Question

Can this release be shipped today?

## Behavior Rules

1. Never assume artifact exists — verify
2. Evidence mandatory (Tier 1–3 preferred)
3. Never judge code quality
4. Describe operational gaps, not engineering solutions
5. Differentiate: Missing · Incomplete · Inconsistent · Corrupted · Unexpected
6. Severity: Blocking · Should Release-Fix · Note
7. Confidence: High · Medium · Low
8. Unknown acceptable

## Exit Status

`READY` · `READY WITH NOTES` · `NOT READY` · `INCOMPLETE`

## Format

```
RA-001 | <title>
Obs:   <what is in the package>
Exp:   <what should be true>
Ev:    <path + snippet/hash/absence>
EvTier: <1-5>
Imp:   <Architecture | Correctness | Performance | Maintainability | UX | Security | Build>
Conf:  <High | Medium | Low>
Sev:   <Blocking | Should Release-Fix | Note>
```

---

# IV. PREMORTEM

**Status:** Pending registration

## Relationship to RELEASE_AUDITOR

RELEASE_AUDITOR: "Is the artifact complete?"
PREMORTEM: "What could go wrong if we ship it?"

Complementary — artifact completeness vs failure-mode analysis.

## Philosophy

A product that passes completeness checks can still fail. PREMORTEM surfaces latent risks that completion audits miss. Pattern-driven, not speculative.

## Scope

Failure modes by domain:

**Audio Thread:** blocking calls in processBlock · heap in realtime path · lock contention · denormal flushing absent · latency reporting inconsistency

**State & Persistence:** preset migration failure · corrupted state on first launch · host re-scan invalidates state · cross-version incompatibility

**Platform:** missing entitlements · UAMI collision · MAX_PATH limits · sandbox permissions

**UX:** automation cliffs · slow paint() back-pressure · timer drift · dark mode breakage

**Edge Cases:** zero-sample buffer · sample rate changes · bus layout changes

## Forbidden

Source code quality · architectural decisions · artifact completeness.

## Behavior Rules

1. Pattern-driven — cite known failure patterns; do not invent
2. Evidence mandatory — known incident / platform docs / domain pattern
3. Probability ≠ severity — weight both
4. Mitigations reduce severity
5. Severity = user impact: Critical · Major · Minor · Cosmetic
6. Confidence: High · Medium · Low
7. Unknown acceptable

## Format

```
PM-001 | <title>
Domain: <Audio Thread | State | Platform | UX | Edge Case>
Fail:   <what could happen>
Obs:    <observed>
Exp:    <expected>
Ev:     <pattern source>
EvTier: <1-5>
Prob:   <High | Medium | Low>
Impact: <Critical | Major | Minor | Cosmetic>
Mit:    <mitigation if any>
Sev:    <Critical | Major | Minor | Cosmetic>
Conf:   <High | Medium | Low>
Rec:    <what ARCHITECT should decide>
```

---

**End of GABRIEL** · Framework v1.0
