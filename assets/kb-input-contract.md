# KB Input Contract — how forkmap reads a `chronicle` knowledge base

forkmap consumes the knowledge base produced by the **`chronicle`** skill. This is the contract between the two: what chronicle writes, and how forkmap reads it **without assuming a fixed shape**.

> chronicle is standalone and writes **no external state file**. Everything forkmap needs is on disk under `knowledge-base/`. Discovery is filesystem-only.

---

## 1. The index is the entry point

chronicle always writes `knowledge-base/README.md` with a **node index**:

```markdown
# {Project} — Knowledge Base

## Node index
| Node | Type | Content |
| [01_vision_y_objetivos.md](01_vision_y_objetivos.md) | file | ... |
| [04_modelos-apis/](04_modelos-apis/README.md) | folder | ... |
| 03_actores_y_roles | omitted (profile: cli) | — |

## Quick start for devs
...

## Executive summary
[2-3 sentences]
```

Read this **first** and treat it as authoritative:
- The **Type** column says whether each node is a `file` or a `folder`. Resolve the real path from the link, never from a hardcoded name.
- A node marked **omitted** is not a gap — the profile deactivated it. Don't flag it or read it.
- The **executive summary** seeds the project name and a one-line framing for the `CHANGES.md` header.

> Node **filenames are not guaranteed** to be canonical Spanish names — chronicle's Q-language can change naming. Always follow the index links; never assume `04_modelo_de_datos.md` exists by that exact name.

---

## 2. The canonical slots (what each node gives the plan)

chronicle uses **10 numbered slots**: core (always present) + variable (by profile).

| # | Node (web_app form) | Kind | What forkmap extracts | Read |
| --- | --- | --- | --- | --- |
| 01 | vision & objectives | map | project name, MVP scope (`Alcance v{X.Y}` / `Fuera de alcance`) | always (core) |
| 02 | description / stack | map | `system_type` + mode signals, stack | always (core) |
| 03 | actors & roles (RBAC) | map | auth + RBAC changes | if present |
| 04 | data / API surface / config / contracts | collection | entities → creation order; library: public symbols; pipeline: data contracts | if present |
| 05 | business rules | collection | cross-cutting rules (RN-…), MVP tags | if present |
| 06 | features / commands / recipes / stages | collection | **the unit of each change** (US-NNN / commands / recipes / jobs), MVP tags | if present |
| 07 | flows / call sequences / execution / DAG | collection | atomic vs composite; pipeline order | if present |
| 08 | architecture | map | patterns → infra prerequisites; env vars | if present |
| 09 | decisions (ADR) | collection | constraints that gate ordering (DD-NN), assumptions (SU-NN) | if present |
| 10 | open questions | backlog | uncertain deps → flag changes ([DISCOVERY], IN headings) | always skim (core) |

**Core = 01, 02, 09, 10 (always). Variable = 03-08 (by profile).** This matters: forkmap must **read a node only if the index lists it**. Do not "always read 04/06" — they are variable. The **change-unit node** (06 for most profiles, or 04 public surface for `library_sdk`) must exist, or there is nothing to slice into changes → abort (see SKILL.md pre-checks).

Optional extras (`11_`, `12_seguridad_compliance`, `1X_tenancy`) complement, never replace — read when present for security/tenancy-flavored changes.

---

## 3. Reading folder (collection) nodes

When the index marks a node a **folder**, it keeps the numeric prefix and has its own `README.md` (the map) plus unit files:

```
knowledge-base/
├── 04_modelos-apis/
│   ├── README.md            ← index + GLOBAL ERD   ← read for ordering
│   ├── modelos/{entity}.md
│   └── contratos-api/{domain}.md
├── 05_reglas-de-negocio/{domain}.md (+ README)     ← RN-{DOMAIN}-NN
├── 06_funcionalidades/{epic}.md (+ README)         ← US-NNN
├── 07_flujos-principales/{flow}.md (+ README)
└── 09_decisiones/DD-NN-{title}.md (+ README w/ SU-NN)
```

Token discipline:
- **Always read the folder `README.md`** — it holds the map/ERD/domain index needed for ordering and slicing.
- **Descend into unit files** when a change's scope, or a **dependency edge**, needs the detail. At scale (>20 changes) descending is required to justify edges — an edge asserted from a README alone is a guess (see `profiles-and-dependencies.md` §6).

---

## 4. Traceability codes (the contract for Scope & Read-before)

chronicle assigns codes. forkmap **cites them** so the downstream OpenSpec change is traceable back to the KB:

| Code | Source | Stability | Meaning |
| --- | --- | --- | --- |
| `US-NNN` | 06 features | stable, cited | user story (the unit of a feature change) |
| `RN-{DOMAIN}-NN` | 05 rules | stable, cited | business rule |
| `DD-NN` | 09 decisions | stable, cited | architecture decision — may gate ordering |
| `SU-NN` | 09 README | stable, cited | assumption (risk if false) |
| `IN-NN` | 10 | **section heading, not a stable cited code** | a detected inconsistency — reference it loosely (`⚠ see node 10 IN-01`), don't treat it like US/RN |
| `[DISCOVERY]` | 10 | marker | a field chronicle couldn't infer → change has uncertain deps |

When a change touches a story, cite it: `Scope: ... (US-012, RN-AUTH-01)`. When node 10 flags an inconsistency/discovery affecting a change, note it in that change's block (`⚠ depends on resolving node 10 IN-03`). A real contradiction can corrupt the dependency inference itself — if core nodes contradict each other on something structural (e.g. whether auth exists), surface it and degrade rather than guessing.

---

## 5. MVP / Post-MVP coverage

chronicle tags items in 05/06 with `[MVP]`, `[Post-MVP]`, `[v2]` — explicitly so a roadmap can be derived (chronicle `conventions.md` §6). **But chronicle leaves them unset in Mode A/C** (`conventions.md`: the code carries no roadmap intent; the tag is the user's or it goes to node 10). So forkmap must **measure, not assume**:

1. While reading 05/06, count feature items **tagged** vs **untagged**.
2. Apply tags as signal:
   - `[MVP]` → on the **critical path**, scheduled early.
   - `[Post-MVP]` / `[v2]` → later phases; never blocks an MVP change.
3. **Coverage rule** (do NOT silently treat all-untagged as all-MVP):
   - **High coverage** (≥ ~60% of feature items tagged) → derive a real MVP critical path; untagged items default to MVP (unless node 01's `Alcance`/`Fuera de alcance` places them otherwise — then honor node 01).
   - **Low coverage** (< ~60% tagged, or zero tags — the common Mode A/C case) → the critical path reflects **all** changes, not a real MVP boundary, **unless** node 01 carries an explicit scope boundary you can use instead. Say so explicitly in the closing output and recommend setting `[MVP]` in node 06 or `Alcance v{X.Y}` in node 01.
4. Node 01 sets the boundary at vision level; item tags refine it. If node 01 has an `Alcance`/`Fuera de alcance` boundary, use it even when item tags are missing.

Carry the coverage ratio into the closing output's **MVP coverage** line.

---

## 6. Profile inference & monorepo

The profile decides the change taxonomy (`profiles-and-dependencies.md`). Infer from, in order:
1. Node 02 stack + description ("CLI", "REST API", "library/SDK", "pipeline/ETL", "multi-tenant").
2. Which variable nodes the index marks active (no 03/UI-07 + API-surface 04 ⇒ `library_sdk`; a DAG in 07 ⇒ `data_pipeline`).
3. Node 01 vision framing.

Ambiguous ⇒ default `web_app` (broadest), note the assumption. A wrong profile degrades naming/taxonomy, never the correctness of edges derived from 04/07.

**Monorepo / multi-subsystem**: if node 02/08 describes **more than one** deployable (e.g. Go API + React web + a CLISDK), the KB covers several subsystems with possibly different profiles. Detect this from the stack table and directory structure (08). Handling in `profiles-and-dependencies.md` §2.5 — do not flatten them into one mislabeled graph.

---

## 7. Mode — greenfield vs brownfield

A KB can describe a system **to be built** or one that **already exists** (chronicle Mode C reverse-documents existing code). The change taxonomy and whether `C-01` is a scaffold depend on it. Detect mode from:

1. **Code presence**: source files / manifests beyond the KB at the project root ⇒ likely brownfield.
2. **chronicle signals**: provenance citations dominated by `[code · …]` (vs `[user]`/`[doc]`) indicate reverse-documented existing code ⇒ brownfield. `[user]`/`[doc]`-dominated with no code ⇒ greenfield.
3. **Node 01 framing**: "we will build" vs "the system does".

- **Greenfield** → build roadmap: `C-01` foundation-setup, scaffold, CI, initial migrations, then domain changes.
- **Brownfield** → **extension/refactor roadmap**: NO foundation-setup, NO scaffold. Changes are derived from gaps, Post-MVP items, and refactors. `C-01` is the first real extension, not project setup.

Ambiguous ⇒ prefer brownfield if any code exists at root; note the assumption in the closing output. See `profiles-and-dependencies.md` §2 for the per-mode taxonomy.

---

## 8. Language

Detect the KB's language from the index and node content; write `CHANGES.md` in **that** language (chronicle asks it once via Q-language). The section names in the template are reference labels — translate to match the KB.

> Caveat: chronicle calls out bilingual/Spanglish KBs as ambiguous. If the KB mixes languages, pick the dominant one for structural labels, keep cited content verbatim, and note the ambiguity in the closing output rather than guessing silently.

---

## 9. No-index fallback (degraded, last resort)

If `knowledge-base/README.md` is missing but the directory has files:
1. Glob `knowledge-base/**`, map entries to slots by numeric prefix (`0N_*` → slot N; `0N_*/` → folder node).
2. Infer the profile from filenames present (no `03_*` ⇒ likely cli/library/pipeline).
3. Proceed, and **warn in the closing output** that discovery was inferred without an index — recommend running `chronicle` to (re)generate `README.md`.

The index path is always preferred; the fallback exists only so a partial KB still produces a plan.
