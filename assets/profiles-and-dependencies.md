# Profiles, Dependencies, Governance, Sizing & Scaling

How the profile and mode reshape the change taxonomy, how dependencies are inferred and validated, how governance and size are assigned, and how derivation stays correct past 50 changes.

---

## 1. Profile inference

Resolve the profile from the KB before slicing (detail in `kb-input-contract.md` §6):

| Signal | Profile |
| --- | --- |
| UI flows (07) + RBAC (03) + entities (04) | `web_app` |
| Service auth, contract-heavy 04, no UI | `api` |
| Commands/flags in 06, no RBAC, config schema in 04 | `cli` |
| UI + offline/sync notes | `mobile` |
| Per-tenant isolation in 04, plan limits in 05 | `saas_multi_tenant` |
| Public API surface in 04, no 03/07-UI | `library_sdk` |
| DAG/lineage in 07, data contracts in 04, no stories | `data_pipeline` |

Ambiguous ⇒ default `web_app`, note the assumption.

---

## 2. Change taxonomy by profile **and mode**

The unit of a change differs per profile; **mode decides whether setup changes exist at all.**

### Mode gate (apply first)
- **Greenfield** (system to be built): include `C-01` foundation-setup and the scaffold/core-models spine below.
- **Brownfield** (chronicle Mode C — code already exists): **drop foundation-setup, scaffold, CI-init, initial migrations.** Changes are **extensions, refactors, and gap-fills**: new Post-MVP features, missing tests/docs flagged in node 10, refactors implied by decisions (09) or open questions. `C-01` is the first real extension. Order them by the same dependency hierarchy (§3), but the "infra first" rule rarely fires.

### `web_app` / `mobile` (greenfield spine)
- `C-01` foundation-setup (scaffold, CI, env, health check).
- `C-02` core-models (shared entities, mixins, repositories, first migration).
- `C-03` auth (if 03 exists): login/refresh/logout, RBAC.
- One change per **domain epic** from 06 (US-NNN grouped). Public/read endpoints → admin → dashboards.
- Integrations (payments, webhooks) late. Visual restyle last.

### `api`
- Same spine, **no frontend changes**. Emphasis on contracts (04); auth = service auth.

### `cli`
- `C-01` foundation (arg parser, config loader, exit-code contract).
- One change per **command**/cohesive group from 06 (flags, IO, exit codes).
- Shared config/IO schema (04) before the commands using it. No auth phase.

### `library_sdk`
- `C-01` foundation (build, types, test harness, semver/release setup).
- **Public API surface (04) gates everything** — the type/symbol contract first.
- One change per **usage recipe / module** from 06. Semver decisions (09) gate breaking vs additive. No auth, no frontend.

### `data_pipeline`
- `C-01` foundation (orchestration scaffold, connections, IO contracts).
- One change per **stage/job** from 06; **ordering follows the DAG in 07** (upstream before downstream). Data contracts (04) before consuming stages. Backfill/monitoring late.

### `saas_multi_tenant`
- `web_app` spine **plus** an early tenancy change (isolation model) before domain features, and per-plan limit enforcement (05) where rules require it.

### 2.5 Monorepo / multi-subsystem
When the KB describes more than one deployable (e.g. Go API + React web + a CLI):
- Treat each subsystem as its own taxonomy with its own profile.
- Prefix change IDs by subsystem (`API-C-01`, `WEB-C-01`) **or** emit one `changes/{subsystem}/` tree per subsystem.
- Model **cross-subsystem edges explicitly** (e.g. `WEB-C-04` depends on `API-C-04` because the page consumes that endpoint). These cross edges are the whole reason not to flatten.
- The root `CHANGES.md` carries a top-level tree showing subsystem subtrees and the cross edges; per-subsystem detail goes to its `changes/` subtree. Never collapse multiple profiles into one mislabeled graph.

---

## 3. Dependency hierarchy

Apply in order; profile/mode overrides win.

1. **Infra first** — `C-01` depends on nothing (greenfield only).
2. **Core models/contracts before features.**
3. **Auth before protected resources** (profiles with 03 only).
4. **Referenced entity before referencing entity** (`categories` → `products`).
5. **Producer before coupled consumer** (endpoint before caller; backend before frontend; cross-subsystem in §2.5).
6. **External integrations / payments / webhooks late.**
7. **Admin / dashboards / reporting late.**
8. **Visual refactors last.**
9. **MVP before Post-MVP** — `[Post-MVP]` never blocks `[MVP]`.

**Profile/mode overrides:**
- `data_pipeline`: the DAG (07) is the dependency graph — follow it literally.
- `library_sdk`: public API surface (04) + semver (09) gate everything; rules 3, 5-7 rarely apply.
- `cli`: rules 3, 5 (frontend), 6, 7 rarely apply.
- **brownfield**: rules 1-2 rarely fire (infra exists); ordering is dominated by 4, 5, 9 and by refactor prerequisites in 09.

> Cite evidence for non-obvious edges: `Dependencies: C-03 (DD-04: postgres before cache)`. Every edge must be **justifiable from the KB**, not inferred from a guess — at scale this means reading the relevant unit files (§6).

---

## 4. Governance levels

| Level | When |
| --- | --- |
| **BAJO** | Scaffolding, simple CRUD, pages/commands without critical logic, config. |
| **MEDIO** | Stateful flows, sessions, state machines, non-critical events, pipeline stages with idempotency. |
| **ALTO** | Notifications, role management, event gateway, observability, public API surface (library). |
| **CRITICO** | Auth, payments, security data (PII, audit), core models everything references, tenancy isolation, data contracts with SLA. |

A change touching node 12 (security/compliance) or carrying PII/payments rules is at least ALTO, usually CRITICO.

---

## 5. Sizing (replaces fabricated time estimates)

Do **not** assert hours. Derive a coarse size from real signals:

| Size | Heuristic |
| --- | --- |
| **S** | 1 unit (one US/command/recipe/stage), ≤1 new entity, no cross-cutting rule. |
| **M** | 2-4 units, a couple of entities, or one integration touchpoint. |
| **L** | 5+ units, several entities, or a flow spanning many components. |

Flag every **L** change as a **split candidate** — a change bundling 5+ units is probably not atomic. Record the size in the per-change block; never translate it to hours (forkmap has no basis for that).

---

## 6. Phase-by-phase derivation & scaling to 50+

A flat single-pass derivation of 50 changes degrades: the model can't hold 50 nodes' dependencies at once, and a flat `CHANGES.md` becomes unscannable.

### Derive phase by phase (correctness)
- Process one phase at a time: identify its changes, their dependencies (to earlier phases **or to earlier changes within the same phase** — never to a later phase), size them, and verify locally (no forward refs, no dangling deps) before the next phase.
- **At scale (>20 changes), descend into unit files** (not just folder READMEs) to justify each dependency edge. A README-only edge is a guess; the per-unit detail is what proves entity/endpoint coupling.
- This mirrors chronicle's "small context, verify each unit, stop before degrading" discipline.

### Output layout (scale rule)
- **≤ 20 changes** → single `CHANGES.md`, full detail inline.
- **> 20 changes** → split:

```
CHANGES.md                        ← master index (root)
changes/
├── FASE-00-cimientos.md
├── FASE-01-auth.md
└── ...                           ← full per-change blocks, grouped by phase
```

### Master index holds (>20)
Header + "how to use", dependency tree (ASCII), gates, critical path, agent plan as a **wave schedule**, and a per-phase summary table (`ID | name | status | size | 1-line scope | gov | deps | → detail file`), linking each `changes/FASE-NN-*.md`.

### Validation still runs on the merged graph
Phase-by-phase derivation does not skip the global Validation gate (SKILL.md §Validation): after all phases, the full graph is checked acyclic, ID-consistent, phase-monotonic, and coverage-complete before writing.

> Always report in the closing output whether detail was promoted to `changes/` and how many files were created. Never split silently.
