# Lifecycle — idempotent re-runs, KB drift, OpenSpec sync

Loaded **only** when `CHANGES.md` already exists (re-run) or `openspec/changes/` has content (sync). On a first generation into a clean repo, skip this asset.

---

## 1. Idempotent re-run (non-destructive merge)

Re-generating must **never** blow away progress or human edits. When `CHANGES.md` already exists:

1. **Read the existing `CHANGES.md`** (and `changes/*` at scale) before deriving anything.
2. **Harvest state to preserve**:
   - Every `[x]` checkbox (completed changes) — keyed by change ID and name.
   - Human edits inside change blocks: added bullets, notes, adjusted Scope/Read-before, manual dependency tweaks. Treat prose you didn't generate as authoritative.
   - Manual phase regroupings or renamed changes.
3. **Derive fresh** from the KB as usual, then **merge, not overwrite**:
   - A change that still exists in the new derivation → keep its preserved `[x]` and human edits; update only the machine-derived fields that genuinely changed (and note what changed).
   - A new change (KB grew) → add it, `[ ]`, in the right phase.
   - A change that no longer maps to any KB unit → **do not silently delete**. Mark it `⚠ orphaned (no KB unit)` and list it in the closing output for the user to confirm removal.
4. **Stable IDs**: keep existing `C-NN` IDs for changes that persist; assign new IDs only to genuinely new changes. Renumbering a live roadmap breaks in-flight references.

> Rule: a re-run is a **merge** of (fresh KB derivation) over (existing human-curated file), with the existing file winning on anything a human touched. Same KB + no edits ⇒ byte-stable output.

---

## 2. KB ↔ CHANGES drift

When re-running, detect and report drift instead of silently diverging:

| Drift | Detect | Report |
| --- | --- | --- |
| **New KB units** | a US/command/recipe/stage with no mapped change | "N new KB units → added changes: …" |
| **Orphaned changes** | a change whose KB units no longer exist | "N changes no longer backed by the KB: … (confirm removal)" |
| **MVP boundary moved** | tag coverage / `Alcance` changed since last run | "MVP boundary changed → critical path recomputed" |
| **Dependency shift** | a decision (DD-NN) or entity relation changed an edge | "Dependency changed: C-07 now needs C-04 (DD-09)" |

Optional cheap fast-path: if a KB fingerprint (e.g. a hash of node 04/05/06/07 + the index) is unchanged since the last run, the roadmap is current — say so and skip re-derivation. forkmap does not own chronicle's ledger; compute its own lightweight fingerprint if it wants this optimization, or just re-derive and merge (§1).

---

## 3. OpenSpec archive sync

The document tells humans to verify dependencies are in `openspec/changes/archive/` — so forkmap should do it too, not just instruct. And it must **decide state from the actual listing, never from assumption**: claiming a change is (or isn't) archived without having listed `archive/` is the exact defect this step guards against.

On every generation/re-run:

1. **List both directories and treat the listing as the evidence** — do **not** assert any change's state without it. Produce the real entries (glob each dir). An empty directory is a result (`→ []`), not a reason to skip the listing:

   ```
   openspec/changes/         → [assign-notifications]            (in-flight)
   openspec/changes/archive/ → [foundation-setup, core-models]  (done)
   ```

   A change's state is decided **only** by which listing its name appears in — `archive/` must be globbed explicitly, including its subdirectories.

2. Match each listed entry to a `C-NN` by name (the `/opsx:propose C-NN-{name}` convention aligns names). Tolerate minor naming differences; match on the kebab name after the `C-NN-` prefix.

3. Reflect real state, **driven by the step-1 listing** (not memory):
   - Name in `archive/` → mark `[x]` and annotate `(archived)`. **Archived wins**: it is done regardless of anything also present in `changes/`.
   - Name in `changes/` only (proposed, not archived) → annotate `(in progress)`, keep `[ ]`.
   - In the KB plan but in **neither** listing → `[ ]` as normal.
   - In a listing but **not** in the derived plan → list under "untracked OpenSpec changes" in the closing output (possible drift or manual work).

4. Recompute the **first recommended change** as the earliest `[ ]` whose dependencies all appear in the `archive/` listing.

> The step-1 listing is an **artifact, not a mental note** — it is the evidence the `[x]` states derive from. Asserting "C-01 is/isn't archived" from memory, without globbing `archive/`, is precisely the bug a real sync prevents.

This makes the roadmap reflect reality on every run, and closes the asymmetry between what the document tells the human and what the skill itself checks.
