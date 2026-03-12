# S05: Enhanced features — merge guards, snapshots, auto-push, rich commits

**Goal:** GitServiceImpl gains five enhanced features: pre-merge verification, snapshot refs, auto-push, rich squash commit messages, and remote fetch before branching. All preference-gated features work end-to-end through the worktree.ts facade.

**Demo:** Unit tests pass proving: (1) `git for-each-ref refs/gsd/snapshots/` shows snapshot ref created before merge, (2) pre-merge check aborts merge on test failure, (3) `git log --oneline -1` on main after merge shows task list in commit body, (4) `git push` called when auto_push enabled, (5) `git fetch` called before new branch creation when remote exists. `npm run build` and `npm run test` pass.

## Must-Haves

- `createSnapshot(label)` creates `refs/gsd/snapshots/<label>/<timestamp>` ref, gated on `prefs.snapshots !== false` (default: on)
- `runPreMergeCheck()` auto-detects test runner from `package.json`/`Cargo.toml`/`Makefile`/`pyproject.toml`, runs it, returns pass/fail. Gated on `prefs.pre_merge_check` (`"auto"` default, `false` to skip, custom string command)
- `mergeSliceToMain` calls snapshot → pre-merge check → squash merge → rich commit → delete branch → auto-push (in that order)
- Rich commit message includes task list from `git log --oneline main..branch` and branch name for forensics
- Multi-line commit messages use `git commit -F -` with stdin pipe instead of `JSON.stringify()` with `-m`
- Auto-push after merge when `prefs.auto_push === true`, best-effort (warn on failure, don't throw)
- Remote fetch (`git fetch --prune`) before new branch creation in `ensureSliceBranch` when remote exists
- `worktree.ts` `getService()` loads real preferences via `loadEffectiveGSDPreferences()` instead of `{}`
- `preferences.ts` validation updated to accept custom string commands for `pre_merge_check` (not just `boolean | "auto"`)
- All features have unit tests in `git-service.test.ts`

## Proof Level

- This slice proves: contract
- Real runtime required: no (temp git repos in unit tests)
- Human/UAT required: no — all features are deterministic git operations verifiable by unit tests

## Verification

- `npm run build` passes
- `npm run test` passes — specifically the new test sections:
  - `createSnapshot` — ref exists at correct path, gated by prefs
  - `runPreMergeCheck` — detects runner, passes/fails correctly, custom command works
  - `mergeSliceToMain` with enhanced flow — snapshot created, rich commit body present
  - Auto-push — push executed when enabled, skipped when disabled, warn on failure
  - Remote fetch — fetch called before branching when remote exists
  - Facade prefs — `getService()` loads real preferences
- `grep -r 'new GitServiceImpl.*{}' src/resources/extensions/gsd/worktree.ts` returns 0 matches (facade fix verified)

## Observability / Diagnostics

- Runtime signals: `console.error` warnings for push failures, fetch failures, and pre-merge check detection misses. These are operational warnings, not structured logs — appropriate for a CLI tool.
- Inspection surfaces: `git for-each-ref refs/gsd/snapshots/` to list all snapshot refs. `git log -1 --format=%B` to inspect rich commit body.
- Failure visibility: Pre-merge check failures include the command that was run and its stderr output. Push failures include the remote and error message. Fetch failures are warnings only.
- Redaction constraints: None — no secrets involved in git operations.

## Integration Closure

- Upstream surfaces consumed: `git-service.ts` (GitServiceImpl, all existing methods), `worktree.ts` (facade getService), `preferences.ts` (loadEffectiveGSDPreferences, GitPreferences validation)
- New wiring introduced in this slice: facade prefs fix (worktree.ts `getService()` → `loadEffectiveGSDPreferences()`), five new methods/behaviors in GitServiceImpl
- What remains before the milestone is truly usable end-to-end: S06 (cleanup/archive of design input files, final doc consistency check)

## Tasks

- [x] **T01: Write failing tests for all S05 features** `est:45m`
  - Why: Establishes the red-green verification contract for all five features before implementation. Tests define exact expected behavior.
  - Files: `src/resources/extensions/gsd/tests/git-service.test.ts`
  - Do: Add test sections for createSnapshot, runPreMergeCheck, rich commit messages, auto-push, remote fetch, and facade prefs loading. Tests use temp git repos following existing `initTempRepo()` pattern. All new tests should fail initially (methods don't exist yet).
  - Verify: `npm run build` passes (tests are valid TS). `npm run test` reports new test failures (expected — methods not implemented).
  - Done when: All new test assertions exist and the test file compiles. Tests cover: snapshot ref creation + prefs gating, pre-merge check detection + execution + abort, rich commit body format, auto-push execution + failure handling, remote fetch before branching, facade loads real prefs.

- [x] **T02: Implement snapshot refs, rich commits, remote fetch, and commit message fix** `est:45m`
  - Why: Delivers R013 (snapshots), R015 (rich commits), R016 (remote fetch), and fixes the multi-line commit message fragility. These are pure git operations with no external process execution.
  - Files: `src/resources/extensions/gsd/git-service.ts`
  - Do: (1) Add `createSnapshot(label)` method using `git update-ref`. (2) Add rich commit message builder that collects `git log --oneline` from branch. (3) Switch `mergeSliceToMain` commit from `git commit -m` with `JSON.stringify` to `git commit -F -` with stdin pipe for multi-line support. (4) Add remote fetch in `ensureSliceBranch` before branch creation. (5) Wire snapshot + rich commits into `mergeSliceToMain` flow.
  - Verify: `npm run build` passes. `npm run test` — snapshot, rich commit, and remote fetch tests pass.
  - Done when: `createSnapshot` creates verifiable refs. `mergeSliceToMain` produces rich commit messages with task list and branch name. `ensureSliceBranch` fetches when remote exists. Related T01 tests go green.

- [x] **T03: Implement merge guards, auto-push, facade prefs fix, and validation update** `est:45m`
  - Why: Delivers R012 (merge guards), R014 (auto-push), fixes the facade prefs wiring gap (R004 support), and corrects preference validation for custom pre_merge_check commands. These are the preference-gated features that need the facade fix to work at runtime.
  - Files: `src/resources/extensions/gsd/git-service.ts`, `src/resources/extensions/gsd/worktree.ts`, `src/resources/extensions/gsd/preferences.ts`
  - Do: (1) Add `runPreMergeCheck()` method that auto-detects test runner from project files. (2) Wire pre-merge check into `mergeSliceToMain` before squash merge. (3) Add auto-push logic after successful merge in `mergeSliceToMain`. (4) Fix `worktree.ts` `getService()` to call `loadEffectiveGSDPreferences()` instead of `{}`. (5) Update `preferences.ts` validation to accept any non-empty string for `pre_merge_check` (not just `boolean | "auto"`).
  - Verify: `npm run build` passes. `npm run test` — all S05 tests pass (0 failures). `grep -r 'new GitServiceImpl.*{}' src/resources/extensions/gsd/worktree.ts` returns 0 matches.
  - Done when: Pre-merge check auto-detects and runs. Auto-push pushes on success, warns on failure. Facade passes real prefs. All T01 tests go green. `npm run build` and `npm run test` pass clean.

## Files Likely Touched

- `src/resources/extensions/gsd/git-service.ts`
- `src/resources/extensions/gsd/worktree.ts`
- `src/resources/extensions/gsd/preferences.ts`
- `src/resources/extensions/gsd/tests/git-service.test.ts`
