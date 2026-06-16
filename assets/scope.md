# Scope — partial generation contract

Loaded **only** when the request scopes a subset of the KB (partial generation). A full-coverage run never loads this asset.

Partial generation produces a **runnable** map for the requested subset and **parks** the rest as future work. It never invents scope and never silently drops KB units.

> Scope is taken from the **request**, not by interrogating the user; closure is resolved from **mode**, not by asking. forkmap picks the sensible default and **reports what it assumed** in the closing output. This stays fire-and-forget.

---

## Resolve scope (in order)

### 1. Selection set `S`

Map the named subset to KB change-units **through the index** (a `US-NNN`, an epic/file in node 06, a command group, a whole subsystem). Discover-don't-assume applies: resolve from the index, never guess.

If the named scope matches no KB unit → **abort cleanly**, don't fabricate it:

> *"No encontré '{X}' como unidad en node 06. Unidades disponibles: {list}."*

### 2. Dependency closure of `S`

Derive `S`'s dependencies with the normal hierarchy (`profiles-and-dependencies.md` §3). Every dep pointing **outside** `S` is an out-of-scope dependency; resolve it by **mode**:

| Mode | An out-of-scope dependency becomes |
| --- | --- |
| **greenfield** | a **real change pulled into the map** — `S` is not runnable without its spine (foundation, core-models, auth as needed). A partial greenfield map = `S` + its minimal down-closure. |
| **brownfield** | a **precondition, not a change** — it already exists. Emit `Precondición: {C-NN or KB ref} (ya existe)`; generate **no block** for it. |

Closure is **downward only** (what `S` needs to run). Do **not** pull in changes that depend *on* `S` (its consumers) — those are parked.

### 3. Park the rest

Every KB change-unit not in `S` and not in its closure goes to a **`Futuras implementaciones`** section:

- Listed: KB unit + name + 1-line + why parked.
- **Not detailed, not deleted.**
- **No committed `C-NN`** — a parked item receives an ID only when a later run promotes it (append-only, so live IDs never renumber — `lifecycle.md` §1).

Parked block format:

```markdown
## Futuras implementaciones

> Out of the current scope. Promote with a re-run when ready — IDs assigned on promotion.

- `wishlist` (US-031) — guardar productos para después. Parked: out of requested scope.
- `admin-reports` (US-040..042) — panel de métricas. Parked: depends on `S` (consumer), not a precondition.
```

### 4. Scope overrides MVP

An explicit subset wins over `[MVP]`/`[Post-MVP]` tags for this map. The critical path is computed over `S` + closure, **not** the KB's MVP boundary. State it in the closing output.

---

## Validate the subgraph

The scoped result still runs the full Validation gate (SKILL.md §Validation), on `S` + closure:

1. **Acyclic** — topological sort of `S` + closure succeeds.
2. **ID integrity** — every dep references an in-map `C-NN` **or** a declared `Precondición` — **never a dangling out-of-scope ID**.
3. **Phase-monotonic** — no change depends on a later phase.
4. **Critical-path consistency** — the path is a real chain within `S` + closure.
5. **Coverage** — every unit in `S` maps to exactly one change; parked units are **listed, not orphaned**.

A scoped map with a dangling out-of-scope dependency is the exact defect this gate prevents.

---

## Closing output (partial)

The `Scope:` line in the closing output reports what was assumed:

```
**Scope**: partial — {requested}; pulled {X} closure change(s) / {Y} precondition(s); **{P} unit(s) parked** to Futuras implementaciones; scope overrode MVP tags.
```

Mode ambiguous ⇒ prefer brownfield if code exists, and say so.

---

## Token note

Partial scope is also the **cheapest** path: read only `S`'s node(s) and the closure's units, never the whole KB. Scope discipline and token discipline pull the same way.
