---
estimated_steps: 6
estimated_files: 1
---

# T01: Write failing tests for all S05 features

**Slice:** S05 — Enhanced features — merge guards, snapshots, auto-push, rich commits
**Milestone:** M001

## Description

Add comprehensive test sections to `git-service.test.ts` for all five S05 features: snapshot refs, pre-merge check (merge guards), rich squash commit messages, auto-push, and remote fetch before branching. Also add a test for the facade prefs loading fix.

Tests follow the existing pattern: `initTempRepo()` creates disposable git repos, `GitServiceImpl` is instantiated with controlled prefs, assertions verify git state. All tests should fail initially because the methods don't exist yet — this establishes the red-green contract.

## Steps

1. Add `createSnapshot` test section: create a GitServiceImpl with `{ snapshots: true }`, call `createSnapshot("gsd/M001/S01")` on a repo with commits, verify ref exists via `git for-each-ref refs/gsd/snapshots/`. Add a second test with `{ snapshots: false }` confirming no ref is created.

2. Add `runPreMergeCheck` test section: create a temp repo with a `package.json` containing `"test": "node -e 'process.exit(0)'"`, verify `runPreMergeCheck()` returns success. Create another with `"test": "node -e 'process.exit(1)'"`, verify it returns failure. Test with `pre_merge_check: false` (skipped). Test with custom command string `pre_merge_check: "node -e 'process.exit(0)'"`.

3. Add rich commit message test section: create a repo with a slice branch that has 2-3 commits, merge via `mergeSliceToMain`, inspect `git log -1 --format=%B` on main to verify the body includes a task list with commit subjects and a `Branch:` line.

4. Add auto-push test section: create a temp repo with a local remote (bare repo as remote), set `{ auto_push: true }`, merge a slice, verify the remote's main has the merge commit. Add a second test with `{ auto_push: false }` (or omitted) confirming no push occurs.

5. Add remote fetch test section: create a temp repo with a local bare remote, add a commit to the remote, call `ensureSliceBranch`, verify no crash (fetch runs). Test without a remote configured — verify no error.

6. Add facade prefs test section: verify that `getService()` in worktree.ts would load prefs (this may need to test via `mergeSliceToMain` from worktree.ts facade — if prefs.snapshots is set in a preferences file, snapshots should be created). Alternatively, test by importing `getService` behavior indirectly — the simplest approach is testing that the worktree facade's merge creates a snapshot when prefs file has `snapshots: true`.

## Must-Haves

- [ ] Snapshot ref creation test (prefs enabled + disabled)
- [ ] Pre-merge check detection and execution test (pass, fail, disabled, custom command)
- [ ] Rich commit message format test (task list + branch line in body)
- [ ] Auto-push test (enabled → pushes, disabled → no push)
- [ ] Remote fetch before branching test (with and without remote)
- [ ] All tests compile (`npm run build` passes)

## Verification

- `npm run build` passes (test file is valid TypeScript)
- `npm run test` runs the test file — new tests fail with expected errors (methods don't exist or behavior doesn't match yet)
- No existing tests break

## Observability Impact

- Signals added/changed: None — these are test-time assertions
- How a future agent inspects this: `npm run test` output shows pass/fail counts and specific failure messages
- Failure state exposed: Test assertion messages identify exactly which feature and scenario failed

## Inputs

- `src/resources/extensions/gsd/tests/git-service.test.ts` — existing test file with helpers (`initTempRepo`, `assert`, `assertEq`, `createFile`, `run`)
- `src/resources/extensions/gsd/git-service.ts` — current GitServiceImpl API (methods to be added in T02/T03)

## Expected Output

- `src/resources/extensions/gsd/tests/git-service.test.ts` — extended with 6 new test sections covering all S05 features. Tests compile but fail (red phase of red-green cycle).
