---
estimated_steps: 5
estimated_files: 3
---

# T03: Implement merge guards, auto-push, facade prefs fix, and validation update

**Slice:** S05 — Enhanced features — merge guards, snapshots, auto-push, rich commits
**Milestone:** M001

## Description

Add pre-merge verification (R012) and auto-push (R014) to `GitServiceImpl`. Fix the critical facade preferences wiring gap in `worktree.ts` so all preference-gated features (snapshots, pre_merge_check, auto_push) actually work when called through the facade. Update `preferences.ts` validation to accept custom string commands for `pre_merge_check`.

These are the "external operations" features — they execute project test commands and push to remotes. The facade fix is the keystone: without it, all preference-gated features from T02 and T03 silently do nothing when called through the worktree.ts facade that auto.ts uses.

## Steps

1. **Add `runPreMergeCheck()` method** to `GitServiceImpl`. Auto-detection logic:
   - Check `this.prefs.pre_merge_check`: if `false`, return immediately (skip). If it's a non-empty string (and not `"auto"`), use that as the custom command.
   - If `"auto"` or `undefined` (default behavior): detect from project files in `this.basePath`:
     - `package.json` with `scripts.test` → `npm test`
     - `package.json` with `scripts.build` → `npm run build` (only if no test script)
     - `Cargo.toml` → `cargo test`
     - `Makefile` with `test:` target → `make test`
     - `pyproject.toml` → `python -m pytest`
   - Execute the detected/configured command via `execSync` with `cwd: this.basePath`, `timeout: 300_000` (5 min), `stdio: ['ignore', 'pipe', 'pipe']`.
   - Return `{ passed: boolean, command: string, output?: string }`. On failure, include stderr in output.
   - If no runner detected in auto mode, return `{ passed: true, command: 'none', output: 'no test runner detected' }` (don't block merge for repos without tests).

2. **Wire pre-merge check into `mergeSliceToMain`** — After snapshot creation and BEFORE `git merge --squash`, call `runPreMergeCheck()`. If it fails, throw an error with the command and output so the caller can report what went wrong. The merge hasn't started yet, so there's nothing to roll back. Important: the check must run AFTER checkout to main branch but BEFORE squash merge — we need to check the slice branch code. Solution: run the check while still on the slice branch (before `switchToMain` in the caller), OR check after squash merge but before commit and reset on failure. The cleanest: run the check on main after `git merge --squash` (tests the merged result), and `git reset --hard HEAD` on failure to undo the squash.

3. **Add auto-push logic** to `mergeSliceToMain` — After successful commit and branch deletion, if `this.prefs.auto_push === true`, run `git push <remote> <mainBranch>` where remote is `this.prefs.remote ?? "origin"`. Use `allowFailure: true` — push failures should `console.error` a warning, not throw. The merge already succeeded locally.

4. **Fix worktree.ts facade `getService()`** — Change `new GitServiceImpl(basePath, {})` to load real preferences. Import `loadEffectiveGSDPreferences` from `preferences.ts`. Call it, extract the `git` field, and pass it to `GitServiceImpl`. Handle the case where prefs loading returns null (no preferences file) — fall back to `{}`. Cache invalidation: the existing cache-by-basePath is fine since prefs don't change mid-session.

5. **Update preferences.ts validation** — Change the `pre_merge_check` validation to accept any non-empty string, not just `boolean | "auto"`. The type already says `boolean | string` in `GitPreferences`, but validation rejects custom strings. Fix: `if (typeof g.pre_merge_check === "boolean" || typeof g.pre_merge_check === "string") { ... }` with string validation requiring non-empty after trim.

## Must-Haves

- [ ] `runPreMergeCheck()` auto-detects test runner from package.json (npm test)
- [ ] Pre-merge check aborts merge when tests fail (before squash merge is committed)
- [ ] Pre-merge check skippable via `pre_merge_check: false` preference
- [ ] Pre-merge check accepts custom string command
- [ ] Auto-push executes `git push` when `auto_push: true`, skips otherwise
- [ ] Auto-push failures warn (don't throw)
- [ ] `worktree.ts` `getService()` loads real preferences (no more hardcoded `{}`)
- [ ] `preferences.ts` accepts custom string for `pre_merge_check`
- [ ] `npm run build` and `npm run test` pass clean (all S05 tests green)

## Verification

- `npm run build` passes
- `npm run test` passes — all existing + all T01 S05 tests green
- `grep -r 'new GitServiceImpl.*{}' src/resources/extensions/gsd/worktree.ts` returns 0 matches
- `grep 'pre_merge_check === "auto"' src/resources/extensions/gsd/preferences.ts` returns 0 matches (replaced with broader string check)

## Observability Impact

- Signals added/changed: `console.error` for pre-merge check failures (includes command + stderr), push failures (includes remote + error), and "no test runner detected" info
- How a future agent inspects this: Pre-merge check result includes command name and output. Push failure includes remote URL.
- Failure state exposed: Pre-merge check failure throws with structured error including command, exit code context, and stderr snippet

## Inputs

- `src/resources/extensions/gsd/git-service.ts` — T02 output with snapshot, rich commits, remote fetch already implemented
- `src/resources/extensions/gsd/worktree.ts` — current facade with `getService()` using `{}`
- `src/resources/extensions/gsd/preferences.ts` — current validation rejecting custom string for `pre_merge_check`
- `src/resources/extensions/gsd/tests/git-service.test.ts` — T01 tests defining expected behavior for merge guards, auto-push, facade prefs

## Expected Output

- `src/resources/extensions/gsd/git-service.ts` — `runPreMergeCheck()` method, auto-push in `mergeSliceToMain`, pre-merge check wired into merge flow
- `src/resources/extensions/gsd/worktree.ts` — `getService()` loads real preferences via `loadEffectiveGSDPreferences()`
- `src/resources/extensions/gsd/preferences.ts` — `pre_merge_check` validation accepts custom string commands
