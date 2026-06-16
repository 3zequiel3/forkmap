# CHANGES Template — the exact structure forkmap writes

Variants:
- **By scale**: A — single file (≤20 changes) · B — master + `changes/` (>20).
- **By mode**: greenfield (build) · brownfield (extend/refactor — different header, no foundation-setup).

Adapt names/content to the domain and **write in the KB's language** (labels below are reference). Respect order and the per-change field contract.

---

## Variant A — single file (`CHANGES.md`), greenfield

```markdown
# CHANGES — Implementation Sequence

> Canonical change index for **{Project}** (profile: `{system_type}`, mode: greenfield).
> Each change is one atomic vertical slice. Size is coarse (S/M/L), not an hour estimate.
> **Read this file before running any `/opsx:propose`.**

---

## How to use this document

1. Pick the change to implement (its dependencies must be in `openspec/changes/archive/`).
2. Read the KB docs listed under "Read-before".
3. Run `/opsx:propose <change-name>`.
4. When done, archive it: `/opsx:archive <change-name>`.
5. Check the box `[x]` here.

---

## Dependency tree

```
C-01 foundation-setup
  └── C-02 core-models
        └── C-03 auth                          ← unblocks everything else
              ├── C-04 menu-catalog
              │     └── C-05 allergens
              ├── C-07 ingredients             ← parallel with C-04
              └── C-08 dashboard-shell
                    └── C-09 dashboard-pages   ← + C-04 + C-05
```

### Parallelism by gate

```
GATE 0: none → C-01
GATE 1: C-01 ✓ → C-02
GATE 2: C-02 ✓ → C-03
GATE 3: C-03 ✓                     ← FIRST FORK
  → C-04 menu-catalog   [Agent A]
  → C-07 ingredients    [Agent B]
  → C-08 dashboard-shell[Agent C]
GATE 4: C-04 ✓ → C-05 allergens [Agent B]
GATE 5: C-08+C-04+C-05 ✓ → C-09 dashboard-pages [Agent C]
```

### Critical path (5 changes — MVP minimum)

```
C-01 → C-02 → C-03 → C-04 → C-09
```

### Execution plan ({A} agents)

```
Step │ Agent A (Backend Core) │ Agent B (Backend Aux) │ Agent C (Frontend)
─────┼────────────────────────┼───────────────────────┼────────────────────
  1  │ C-01 foundation-setup  │          —            │          —
  2  │ C-02 core-models       │          —            │          —
  3  │ C-03 auth              │          —            │          —
  4  │ C-04 menu-catalog      │ C-07 ingredients      │ C-08 dashboard-shell
  5  │ C-05 allergens (B)     │          —            │ C-09 dashboard-pages
```

---

## FASE 0 — Foundations

### [C-01] `foundation-setup`
- **Status**: `[ ]` pending
- **Size**: S
- **Scope**: full scaffold + base infra
  - Directory structure, health check `/api/health`, migrations initialized, shared settings/logger/db
  - `.env.example` per sub-project; sensitive vars via `${VAR}`; CI parallel jobs
- **Dependencies**: none
- **Governance**: BAJO
- **Read-before**:
  - `knowledge-base/01_vision_y_objetivos.md`
  - `knowledge-base/02_descripcion_general.md` §Stack
  - `knowledge-base/08_arquitectura_propuesta.md` §Directory structure

---

### [C-03] `auth`
- **Status**: `[ ]` pending
- **Size**: M
- **Scope**: JWT auth + RBAC `[MVP]`
  - `POST /api/auth/login` — JWT access + refresh, rate limit 5/60s per IP+email (US-012, RN-AUTH-01)
  - `POST /api/auth/refresh` — rotation + blacklist (RN-AUTH-02); `require_role()`/`require_admin()`
  - Migration 002; Tests: valid login, expired token, rate limit, refresh rotation
- **Dependencies**: C-02
- **Governance**: CRITICO
- **Read-before**:
  - `knowledge-base/03_actores_y_roles.md`
  - `knowledge-base/05_reglas-de-negocio/auth.md` §RN-AUTH-* (or `05_reglas_de_negocio.md` if file)
  - `knowledge-base/07_flujos-principales/login.md`

---

### [C-{NN}] `{change-name}`
- **Status**: `[ ]` pending  *(or `[x]` archived (path) / `(in progress)` when matched in OpenSpec — see `lifecycle.md` §3)*
- **Size**: S | M | L  *(flag L as split candidate + suggest the split)*
- **Scope**: operational bullets citing KB codes (US-NNN, RN-{DOMAIN}-NN) and MVP tags
- **Dependencies**: none | `C-NN` | `C-NN, C-MM`
- **Governance**: BAJO | MEDIO | ALTO | CRITICO
- **Read-before**: `knowledge-base/0X_...` §{section or code}

---

## FASE {M} — {semantic name}
(continue until every change-unit in the KB is covered exactly once)
```

> **Path note**: resolve every `Read-before` path from the KB index. For folder nodes, point to the unit file (`05_reglas-de-negocio/auth.md`) or its README, not a non-existent flat file.

> **Tree note (DAG ≠ tree)**: the dependency graph is a DAG, but ASCII is a tree. When a change has **multiple parents**, draw it once under its **primary** parent and annotate the secondary edges inline (`← also dep. C-04`). Don't duplicate the node's subtree. The authoritative dependency list is each change's `Dependencies` field; the tree is a visual aid.

---

## Brownfield header variant

When mode = brownfield, swap the header and drop foundation-setup:

```markdown
# CHANGES — Extension & Refactor Plan

> Change index for **{Project}** (profile: `{system_type}`, mode: brownfield — system already exists).
> Changes are extensions, refactors, and gap-fills. No scaffold — the project is built.
> **Read this file before running any `/opsx:propose`.**
```

`C-01` is the first real extension (e.g. a Post-MVP feature or a refactor from node 09), never `foundation-setup`.

---

## Variant B — master index (`CHANGES.md`, >20 changes)

```markdown
# CHANGES — Implementation Sequence

> Canonical change index for **{Project}** (profile: `{system_type}`, mode: `{mode}`).
> {N} changes across {M} phases. Per-phase detail in [`changes/`](changes/).

## How to use this document
(same 5 steps as Variant A)

## Dependency tree
(ASCII tree — IDs only; for monorepo, subsystem subtrees + cross edges)

### Parallelism by gate
(GATEs with agent assignment)

### Critical path ({K} changes — MVP)
(linear arrow chain)

### Execution plan ({A} agents) — wave schedule
```
Wave 1: C-01
Wave 2: C-02
Wave 3 (FORK): C-04 [A] · C-07 [B] · C-08 [C]
...
```

## Phases overview
| ID | Change | Status | Size | Scope (1 line) | Gov | Deps | Detail |
|----|--------|--------|------|----------------|-----|------|--------|
| C-01 | foundation-setup | `[ ]` | S | scaffold + CI + env | BAJO | — | [FASE 0](changes/FASE-00-cimientos.md) |
| C-03 | auth | `[ ]` | M | JWT + RBAC (US-012) | CRITICO | C-02 | [FASE 1](changes/FASE-01-auth.md) |
```

Each `changes/FASE-NN-{slug}.md` holds the full change blocks (Status, Size, Scope, Dependencies, Governance, Read-before) for that phase only.

---

## Close-out validation checklist

Before returning, verify — **graph integrity first (blocking)**:

- [ ] **Acyclic**: a topological sort of the dependency graph succeeds (no cycles).
- [ ] **ID integrity**: every dependency references an existing `C-NN`.
- [ ] **Phase-monotonic**: no change depends on a change in a later phase.
- [ ] **Critical-path consistency**: the path is a real chain and ends on an MVP-indispensable change.
- [ ] **Coverage**: every KB change-unit (US/command/recipe/stage) maps to exactly one change; orphans reported.

Then format/content:

- [ ] Header matches the mode (Implementation Sequence vs Extension & Refactor Plan) and states profile + mode, in the KB's language.
- [ ] Brownfield: no `foundation-setup`/scaffold change.
- [ ] "How to use" has the 5 steps.
- [ ] Dependency tree uses ASCII art (`└──`, `│`); monorepo shows subsystem subtrees.
- [ ] ≥1 GATE per parallelism fork; execution plan present (table ≤20, wave schedule >20).
- [ ] Each change has all fields: Status, **Size**, Scope, Dependencies, Governance, Read-before.
- [ ] Every **L** change flagged as a split candidate.
- [ ] Read-before: 3-5 KB paths resolved from the index (folder-aware), with section/code.
- [ ] Scope bullets are operational and cite KB codes.
- [ ] Changes grouped into semantic phases; profile-appropriate taxonomy.
- [ ] MVP-coverage note present when tag coverage is low.
- [ ] At scale: `changes/` files created/linked; split reported in the closing output.
```
