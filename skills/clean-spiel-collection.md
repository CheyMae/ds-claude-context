---
name: clean-spiel-collection
description: >
  Audits and restructures a Figma string variable collection (a "Spiel collection") to be
  complete, deduplicated, and semantically named. Use this skill any time the user wants to
  clean up, audit, reorganize, or create string variables for copy/content in a Figma file —
  including phrases like "clean up the spiels", "audit the copy variables", "create variables
  for unbound text", "merge duplicate string variables", or "organize the content collection."
  Also trigger when the user shares a Figma file and asks to make text variables consistent,
  functional, or production-ready. The skill covers the full pipeline: discovery of unbound
  strings, variable creation, deduplication, functional renaming, binding, and cleanup.
compatibility:
  tools:
    - figma-console:figma_execute        # all reads, writes, deletes
    - figma-console:figma_get_variables  # collection inspection
  figma_plugin: figma-console-mcp (must be open and connected)
---

# clean-spiel-collection

Cleans a Figma string variable collection end-to-end: discovers unbound text, creates missing
variables, merges duplicates, renames everything by usage context, binds all nodes, and deletes
dead variables. Two human review gates protect against destructive mistakes.

---

## Inputs

| Input | How to get it |
|---|---|
| Figma file URL | From the user — extract the file key from the URL |
| Collection name | From the user, or prompt if ambiguous (e.g. "Spiels", "Content") |
| Target page ID | From the URL `node-id` param, or scan all pages if not specified |

---

## Phases at a glance

```
Phase 1 — Discovery        (read-only)
    └── 1a. Walk unbound text nodes outside components
    └── 1b. Cross-ref against existing collection values
    └── 1c. Group candidates, propose new vars + binding gaps
         ↓
    ⛔ HUMAN GATE A — approve proposed variables and names
         ↓
Phase 2 — Inventory        (read-only)
    └── Full collection read + usage-count map (all pages, all nodes)
         ↓
Phase 3 — Propose restructure   (read-only)
    └── Merge table, rename table, unused list
         ↓
    ⛔ HUMAN GATE B — approve before any destructive writes
         ↓
Phase 4 — Execute          (writes)
    └── 4a. Create new variables
    └── 4b. Rename survivors
    └── 4c. Rebind + delete duplicates   (atomic)
    └── 4d. Bind all unbound nodes
    └── 4e. Delete unused variables      (after zero-reference confirm)
         ↓
Phase 5 — Verify           (read-only)
```

---

## Phase 1 — Discovery

### 1a. Walk unbound text nodes

Run a full `figma_execute` walk on the target page. For every text node:
- **Exclude** nodes inside `COMPONENT`, `COMPONENT_SET`, or `INSTANCE` ancestors
- Record: `nodeId`, `characters`, `parentFrameName`, `parentFrameId`, `boundVariables.characters`

Split into two buckets:
- **Bound** → carries into Phase 2 (already has a variable ID)
- **Unbound** → carries into Phase 1b

```js
// Pattern: async walk excluding component interiors
function hasComponentAncestor(node) {
  let n = node.parent;
  while (n && n.type !== 'PAGE') {
    if (['COMPONENT','COMPONENT_SET','INSTANCE'].includes(n.type)) return true;
    n = n.parent;
  }
  return false;
}
```

### 1b. Cross-reference against existing collection

For each unbound string, check whether the collection already has a variable whose resolved
value matches exactly (case-sensitive). Split:
- **Binding gap** — value exists in collection but node is unbound → no new var needed, just bind
- **True candidate** — no matching value anywhere in collection → needs a new variable

### 1c. Group candidates and propose names

For each true candidate, infer a `screen/role` name using the priority stack below.
Collapse identical string values across nodes into a single proposed variable.

#### Naming convention: `screen/role`

**Inferring `screen`** (use first confident signal):

1. Top-level frame name — clean and map to a product flow noun (see Screen Vocabulary below)
2. Sibling/parent layer names + nearby bound variable names in the same frame
3. String content itself — certain strings signal their context (`"Transaction ID"` → `receipt`)
4. Flag for human review if no confident answer

**Screen vocabulary** — map frame names to these canonical tokens:

| Token | Maps from |
|---|---|
| `login` | Sign In, Pre-Auth, Welcome Back |
| `transfer` | Fund Transfer, Send Money, Transfer To |
| `favorites` | Favorites, Saved Recipients |
| `otp` | OTP, One-Time Password, Verify |
| `receipt` | Receipt, Confirmation, Summary |
| `transaction` | Transaction History, Activity Detail |
| `activity` | Activity, History, Recent |
| `notifications` | Notifications, Alerts |
| `wallet` | Wallet, Balance, My Money |
| `brand` | Appears identically across 3+ unrelated screens |
| `footer` | Help, Support, Legal — appears at bottom of multiple screens |
| `actions` | Generic UI verbs: Edit, Remove, Cancel, Confirm |

Add new tokens as needed — derive from the actual product, never force a string into a
token that doesn't fit. Flag it for human review instead.

**Inferring `role`** (from visual position, text style, interaction context):

| Role | When to use |
|---|---|
| `title` | Primary screen or section heading |
| `subtitle` | Supporting copy directly under a title |
| `sectionTitle` | Sub-section heading within a screen |
| `label` | Field or data label ("Transaction ID", "Amount") |
| `value` | Data value paired with a label — never the same var as its label |
| `cta` | Primary action button text |
| `ctaSecondary` | Secondary or ghost button text |
| `link` | Tappable inline or standalone text link |
| `placeholder` | Input field placeholder |
| `emptyState` | Message shown when list/section has no content |
| `status` | State indicator ("Transferred", "Pending", "Failed") |
| `helperText` | Instructional copy under an input or action |
| `errorText` | Validation or error message |
| `timestamp` | Date, time, or period display |
| `amount` | Monetary value |
| `prefix` | Fixed prefix attached to an input ("+63", "PHP") |
| `badge` | Short pill or tag label |
| `bodyText` | Paragraph copy that doesn't fit a tighter role |

**Disambiguation rules:**

- Same string, multiple screens → one variable only if the product team would want them to
  change together. Default to splitting (`favorites/seeAllLink`, `notifications/seeAllLink`).
  Merge only when semantically identical across all contexts.
- Same screen, multiple instances of same role → qualify with context noun, not a number.
  `otp/subtitle` and `otp/resendSubtitle`, never `otp/subtitle1`.
- Numbers in names are a last resort.
- Labels and values always get separate variables even when they appear in the same row.

### Human Gate A — Phase 1 proposal

Present before writing anything. Three sections:

1. **New variables to create** — proposed `screen/role` name, string value, node IDs that will bind to it
2. **Binding gaps** — nodes whose string matches an existing variable; show var name and node IDs
3. **Flagged for review** — strings where screen/role inference was ambiguous; ask for direction

Wait for explicit approval. User may rename, regroup, or remove items before proceeding.

---

## Phase 2 — Inventory

After Gate A approval, read the full collection state (pre-write — new vars don't exist yet).

Run two reads in a single `figma_execute`:

1. **`getLocalVariablesAsync()`** filtered to the target collection → build `byName` and `byId` maps, record resolved values per variable
2. **Full document walk** (all pages, all nodes including component interiors) → for each text node with a `boundVariables.characters` alias, increment a usage counter keyed by variable ID

Result: every variable in the collection has a usage count. Zero = deletion candidate.

> This phase is read-only. No human gate. Output feeds directly into Phase 3.

---

## Phase 3 — Propose restructure

Using the inventory, produce three tables:

**Merge table** — variables with identical resolved values
- Show: survivor name (proposed), victim name (to delete), string value, node count currently bound to victim
- Victim nodes will be rebound to the survivor before deletion

**Rename table** — variables keeping their value, getting a better name
- Show: current name → proposed `screen/role` name
- Flag any that conflict with the naming vocabulary; ask for human input

**Unused list** — variables with zero usage across all pages and component interiors
- Show: name, value, zero-usage confirmation
- Do not include variables from the merge table victim column here (they're handled separately)

Include totals: current count → projected count after restructure.

### Human Gate B — restructure proposal

Present all three tables. Wait for explicit approval. This is the last gate before destructive
writes. User may adjust proposed names, remove items from the unused list, or cancel.

---

## Phase 4 — Execute

Run in strict order. Never reorder steps.

### 4a. Create new variables

One `figma_execute` block. Create all approved Phase 1 new variables in the target collection.
Build a `byName` map after creation to get the new variable IDs for binding steps.

```js
// Pattern
const collection = collections.find(c => c.name === COLLECTION_NAME);
const modeId = collection.defaultModeId;
const newVar = figma.variables.createVariable(name, collection, 'STRING');
newVar.setValueForMode(modeId, value);
```

### 4b. Rename survivors

One `figma_execute` block. Apply all renames from the merge table (survivors) and rename table
before any deletions. This prevents name collisions where a new name matches a victim's old name.

```js
v.name = newName; // direct property set
```

### 4c. Rebind then delete duplicates (atomic)

One `figma_execute` block. Must not be split into two calls.

1. Walk all pages (all nodes including component interiors)
2. For each text node whose `boundVariables.characters.id` matches a victim variable ID,
   call `node.setBoundVariable('characters', survivorVar)` using `getVariableByIdAsync`
3. After walk completes, call `.remove()` on each victim variable

```js
// Pattern: async walk for rebind
async function walk(node) {
  if (node.type === 'TEXT') {
    const alias = node.boundVariables?.characters;
    if (alias?.type === 'VARIABLE_ALIAS' && victimIdToSurvivorId[alias.id]) {
      const survivor = await figma.variables.getVariableByIdAsync(victimIdToSurvivorId[alias.id]);
      if (survivor) node.setBoundVariable('characters', survivor);
    }
  }
  if ('children' in node) for (const c of node.children) await walk(c);
}
for (const page of figma.root.children) await walk(page);
// then remove victims
victims.forEach(v => v.remove());
```

### 4d. Bind all unbound nodes

One `figma_execute` block. Process both:
- Phase 1 binding gaps → bind to the pre-existing variable that matches
- Phase 1 new variables → bind to their newly created variable IDs

```js
const node = await figma.getNodeByIdAsync(nodeId);
node.setBoundVariable('characters', variable);
```

### 4e. Delete unused variables

One `figma_execute` block. Before deleting each variable, re-confirm zero references with
a targeted walk (don't rely on Phase 2 counts — new bindings may have changed things).

```js
// Quick targeted check: scan all pages for this variable ID before removing
```

Only delete after zero-reference confirmed. Log any that now have references and skip them —
surface these to the user after execution completes.

---

## Phase 5 — Verify

One `figma_execute` block. Re-fetch the collection and re-run the full page walk.

Assert all of the following. Report failures with node IDs and variable names — never silently pass:

| Assertion | Pass condition |
|---|---|
| Variable count | Matches expected post-restructure number |
| Unbound text nodes | Zero outside components on target page |
| Dead variables | Zero variables with zero usage in collection |
| Orphaned bindings | No `boundVariables.characters` alias pointing to a deleted ID |

If any assertion fails, report what failed and what to investigate. Do not auto-correct — surface
to the user for a decision.

---

## Output summary format

After Phase 5, present a compact summary:

```
✓ Variables: 68 → 43
✓ Unbound nodes fixed: 8
✓ Duplicates merged: 15
✓ Unused deleted: 10
✓ All assertions passed
```

If any assertion failed, replace the relevant line with the failure details.

---

## Common failure modes

**`getVariableById` throws in async context** — always use `getVariableByIdAsync` inside
`figma_execute` blocks that use async/await.

**Name collision on rename** — rename all survivors before deleting any victims. A victim's
name may be the same as a survivor's new name; deletion before rename causes a conflict.

**Component-interior rebind** — Phase 4c walk must include component interiors even though
Phase 1 discovery excluded them. Rebind scope ≠ discovery scope.

**Stale `byName` map** — rebuild the variable map at the top of each Phase 4 block with a
fresh `getLocalVariablesAsync()` call. Never reuse a map across separate `figma_execute` calls.

**Page not found** — always locate the target page by ID (`figma.root.findChild(p => p.id === id)`)
not by index. Page order can change.
