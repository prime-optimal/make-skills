
# Blueprint Pull-Test-Push

## Overview

Make.com scenarios mutate in the UI without any automatic versioning. Pushing a stale blueprint silently overwrites live changes, and a misidentified scenario ID corrupts the wrong automation entirely. This skill enforces a safe pull-analyze-modify-push-commit loop that prevents data loss and keeps your local repo in sync with what is actually running.

## When to Use

Use any time you touch a Make.com scenario:
- "Pull the blueprint for X"
- "Let's update the scenario" or "add a module to the scenario"
- Debugging a scenario that is behaving unexpectedly
- Reviewing mapper values or router logic
- Before AND after any structural change to a scenario

**Use this ESPECIALLY when:**
- You have not touched the scenario recently (UI may have drifted)
- Multiple people have access to the Make workspace
- You are about to push — always pull first

**Do not skip when:**
- The change "seems small" (one missed pull can still lose hours of UI work)
- You are fixing a broken scenario under time pressure

## Instructions

### Step 1 — Verify version control status

Before touching anything, confirm the blueprint is tracked in git:

```bash
git status | grep blueprints/
```

If the file is untracked or has uncommitted local changes, **stop**. Commit or stash those changes first. Do NOT proceed until the working tree is clean for that file.

### Step 2 — Pull the latest blueprint from Make

Use the justfile recipe to fetch the live version:

```bash
just make-pull <scenario-name>
```

This overwrites your local copy with what is currently deployed. If the recipe does not exist yet, check the justfile for the correct target name — never invoke the Make API directly without the justfile wrapper.

### Step 3 — Confirm the scenario ID mapping

Open the justfile and locate the mapping that links the filename to a numeric Make scenario ID. **Never assume the ID from the filename alone.** The justfile mapping is the source of truth. Note the ID before making any edits.

### Step 4 — Analyze the blueprint JSON

Read the downloaded JSON and assess:
- **Module IDs**: Each module has a unique integer ID. Note the highest ID in use — new modules must use an ID higher than this.
- **Router routes**: Routes are evaluated top-to-bottom (not in parallel). Order matters.
- **Mapper values**: Check `{{...}}` expressions for correctness and stale field references.
- **Connections**: Verify each module references a valid, active connection ID.

### Step 5 — Check for orphaned modules

Orphaned or detached modules (not connected to any route) inflate file size and can cause unexpected behavior. If the file is unusually large, scan for modules with no incoming or outgoing edges and remove them before making your changes.

### Step 6 — Make targeted changes

Edit the blueprint JSON directly or via tooling:
- Add/modify modules using the highest available ID + 1.
- Update mapper `{{...}}` values surgically — change only the fields you intend to change.
- For **AITable link field writes**, always use an HTTP module with `PATCH` to the Fusion API. Do NOT use the native AITable updateRecords module — it cannot write link fields.

### Step 7 — Set HTTP module authentication

If your change involves any HTTP module, verify its `auth` property is set to `"none"`. The Make UI defaults to prompting for authentication; pushing a blueprint with a missing or wrong auth setting breaks the scenario silently.

### Step 8 — Push the blueprint with auto-healing

```bash
just make-push <scenario-name>
```

Auto-healing resolves minor schema drift. If the push fails, read the error before retrying — do not guess and re-push.

### Step 9 — Test the scenario

Trigger a real execution:
- **Webhook-triggered**: fire the webhook with a representative payload.
- **Scheduled/manual**: click "Run once" in the Make UI.

Watch the execution log in Make. Confirm every module shows a green checkmark and the expected data flowed through.

### Step 10 — Verify the result in AITable

Open the target table in AITable and confirm the record was created or updated as expected. For link fields specifically, confirm the linked record IDs resolved correctly — not just that the field appears non-empty.

### Step 11 — Commit the updated blueprint

```bash
git add blueprints/<scenario-name>.json
git commit -m "chore: update <scenario-name> blueprint — <brief description>"
```

Do not skip this step. An uncommitted blueprint is a blueprint you will lose the next time someone else pulls.

## Best Practices

| Do | Don't |
|----|-------|
| Pull before every push | Push from a stale local file |
| Verify scenario ID in the justfile | Guess the ID from the filename |
| Use HTTP PATCH to Fusion API for link fields | Use native AITable updateRecords for link fields |
| Set HTTP module auth to "none" explicitly | Leave auth at Make's default prompt |
| Commit after every successful push | Leave blueprints uncommitted "for now" |
| Check active vs. archived status before editing | Update an archived scenario by mistake |

## Common Pitfalls

| Problem | Solution |
|---------|----------|
| Push overwrites live UI changes | Always `just make-pull` before any push |
| Wrong scenario updated | Confirm ID in justfile before pushing |
| Link field write silently fails | Switch to HTTP PATCH against Fusion API |
| HTTP module breaks after push | Set `auth` to `"none"` on every HTTP module |
| Mapper expressions reference deleted AITable fields | Re-inspect `expect` schemas after any AITable field change; update mappers to match |
| Blueprint file is unexpectedly large | Search for orphaned/detached modules and remove them |
| Scenario runs but nothing lands in AITable | Verify link field IDs, not just the surface value |

## Key Constraints

- **Routes resolve top-to-bottom.** In Make routers, route order determines which branch runs. Parallel evaluation is not supported.
- **Native AITable updateRecords cannot write link fields.** Always use HTTP PATCH to the Fusion API.
- **Always commit blueprints after a successful push.** Git is the only version history Make.com does not provide.
- **Never push without pulling first.** Even a 5-minute gap is enough for a UI change to be lost.
- **Scenario ID mapping lives in the justfile.** It is the single source of truth.
