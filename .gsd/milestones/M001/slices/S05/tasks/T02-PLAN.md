---
estimated_steps: 5
estimated_files: 1
---

# T02: Implement snapshot refs, rich commits, remote fetch, and commit message fix

**Slice:** S05 — Enhanced features — merge guards, snapshots, auto-push, rich commits
**Milestone:** M001

## Description

Add three features to `GitServiceImpl` in `git-service.ts`: hidden snapshot refs before merges (R013), rich squash commit messages with task lists (R015), and remote fetch before branching (R016). Also fix the multi-line commit message fragility by switching from `git commit -m` with `JSON.stringify()` to `git commit -F -` with stdin pipe.

These are "pure git" features — no external process execution (test runners, push to remotes). They modify `mergeSliceToMain()` and `ensureSliceBranch()`.

## Steps

1. **Add `createSnapshot(label)` method** to `GitServiceImpl`. Uses `git update-ref refs/gsd/snapshots/<label>/<YYYYMMDD-HHmmss> HEAD`. Gated on `this.prefs.snapshots !== false` (default: on — undefined counts as enabled). Label should have `/` replaced or preserved as-is since git ref paths handle `/` natively.

2. **Switch commit helper to stdin pipe** — Replace the `git commit -m JSON.stringify(message)` pattern with a helper that writes to `git commit -F -` via stdin. This is necessary for multi-line rich commit messages. Use `execSync` with `input` option on `stdio: ['pipe', 'pipe', 'pipe']`. Apply this to all commit calls in the class (the `commit()` method and `mergeSliceToMain()`).

3. **Add rich commit message builder** — In `mergeSliceToMain`, after squash merge and before commit, collect `git log --oneline <main>..<branch>` to get branch commit subjects. Build message body:
   ```
   type(scope): title
   
   Tasks:
   - commit subject 1
   - commit subject 2
   
   Branch: gsd/M001/S01
   ```
   Handle edge case where branch has many commits (cap at ~20 entries with "..." truncation).

4. **Add remote fetch in `ensureSliceBranch`** — Before creating a new branch (inside the `!this.branchExists(branch)` block), check if a remote exists via `git remote`. If so, run `git fetch --prune <remote>` (using `this.prefs.remote ?? "origin"`). Use `allowFailure: true` and `console.error` on failure (fetch is best-effort). After fetch, check if local main is behind remote via `git rev-list --count HEAD..@{upstream}` with `allowFailure` (upstream may not be set).

5. **Wire snapshot + rich commit into `mergeSliceToMain`** flow. New order: save branch ref before switching → switch to main → snapshot (before squash) → `git merge --squash` → build rich commit message → `git commit -F -` → delete branch. The snapshot captures the slice branch HEAD before it's deleted.

## Must-Haves

- [ ] `createSnapshot(label)` creates refs visible via `git for-each-ref refs/gsd/snapshots/`
- [ ] Snapshot creation gated on `prefs.snapshots !== false` (default on)
- [ ] Rich commit body includes task list from branch commits and `Branch:` line
- [ ] Multi-line commit messages work correctly (no quoting/escaping issues)
- [ ] Remote fetch runs before new branch creation when remote exists
- [ ] Remote fetch is best-effort (warns, doesn't throw)
- [ ] All existing tests still pass (no regressions from commit message change)

## Verification

- `npm run build` passes
- `npm run test` — snapshot, rich commit, remote fetch, and commit message tests from T01 pass
- Existing merge tests still pass (commit message format change is backward-compatible because body is additive)

## Observability Impact

- Signals added/changed: `console.error` warnings for fetch failures and behind-remote detection
- How a future agent inspects this: `git for-each-ref refs/gsd/snapshots/` lists snapshot refs. `git log -1 --format=%B` shows rich commit body.
- Failure state exposed: Fetch failure warning includes remote name and error detail

## Inputs

- `src/resources/extensions/gsd/git-service.ts` — current GitServiceImpl with `mergeSliceToMain`, `ensureSliceBranch`, `commit` methods
- `src/resources/extensions/gsd/tests/git-service.test.ts` — T01 tests defining expected behavior

## Expected Output

- `src/resources/extensions/gsd/git-service.ts` — GitServiceImpl gains `createSnapshot(label)` method, rich commit builder, stdin-pipe commit helper, remote fetch logic. `mergeSliceToMain` uses new flow. `ensureSliceBranch` fetches before branching.
