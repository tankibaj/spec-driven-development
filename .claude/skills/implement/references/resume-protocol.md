# Resume Protocol

Procedure for resuming an interrupted implementation session. Follow this when `status.yaml` shows a WP with `status: in_progress` and a `last_checkpoint`.

---

## When to Use

- Session was interrupted (timeout, crash, human ended session)
- Agent is resuming work on a WP that was previously started
- `status.yaml` has `phase_4.WP-XXX.status: in_progress` with a `last_checkpoint`

---

## Procedure

### 1. Read State

```
1. Read status.yaml — find the WP's last_checkpoint and status
2. Read the WP in full — treat it as a fresh spec (it may have been updated since last session)
3. Check status.yaml blockers — if there are unresolved blockers for this WP, STOP and report
```

### 2. Verify Code State

```
1. Navigate to the workspace
2. Check which branch is active — should be the feature branch for this WP
   IF on wrong branch:
     → Switch to the correct branch per .claude/rules/branching-strategy.md
     → If branch doesn't exist, the previous session may not have committed. Start fresh.

3. Check git log on the feature branch:
   → List commits since branch creation
   → These show what was already implemented and committed

4. Check for uncommitted changes:
   IF uncommitted changes exist:
     → Review them. If they look complete and correct, commit them.
     → If they look broken or partial, stash or discard them.
     → When in doubt, discard and re-implement from the last committed checkpoint.
```

### 3. Re-run Tests

```
1. Run existing tests (if any exist from previous session)

IF all tests pass:
  → Code state is verified. Continue from last_checkpoint.

IF some tests fail:
  → The interruption may have left broken code.
  → Check if the failures are in the last checkpoint category (partially completed).
  → If so: re-implement that checkpoint category from scratch.
  → If failures are in earlier categories (previously passing):
    → Something changed. Investigate before continuing.
    → If the WP was updated, the old code may no longer be correct.
    → Re-implement from the failing category forward.

IF no tests exist yet:
  → The previous session hadn't reached the tests checkpoint.
  → Continue from last_checkpoint (which should be before tests).
```

### 4. Continue Implementation

```
1. Identify the last completed checkpoint category (from last_checkpoint)
2. Map it to the checkpoint taxonomy:
   BE: scaffold → models → routes → integration → tests → dod
   FE: scaffold → state → components → integration → tests → dod

3. Skip all categories before and including the last checkpoint
4. Resume from the NEXT category in the taxonomy

EXCEPTION: If last_checkpoint indicates partial completion
  (e.g., "routes — 3/5 endpoints implemented"):
  → Resume within that category, implementing the remaining items
```

### 5. Update Status

After resuming:
- Update `last_checkpoint` as you complete each new category (normal checkpoint behavior)
- If you had to re-implement a category, note it in the checkpoint: `"routes — re-implemented after resume, all 5 endpoints done"`

---

## Edge Cases

### Branch exists but no commits

The previous session created the branch but crashed before committing anything. Treat this as a fresh start — begin from the first checkpoint category.

### WP was updated since last session

The human may have updated the WP to resolve a blocker. Re-read the WP in full. Check if the changes affect already-completed work:
- If changes are in a future checkpoint category → no impact, continue normally
- If changes affect an already-completed category → re-implement that category and everything after it

### Multiple blockers resolved at once

If `status.yaml` had multiple blockers and the human resolved all of them:
1. Clear all resolved blockers
2. Re-read the WP
3. Resume from `last_checkpoint`
4. The fixes may affect multiple categories — check each one

### status.yaml shows `done` but human asks to resume

The WP was already marked complete. Ask the human what specifically needs to change — this is outside the normal resume flow and may require a scope discussion.
