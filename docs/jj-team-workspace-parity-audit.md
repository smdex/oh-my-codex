# JJ Team Workspace Feature-Parity Audit

**Audit date:** 2026-07-28  
**Audited revision:** `a2740ad601974fa9cfa9133c8a42d44585830eba`  
**Project version:** `0.20.3`  
**Local JJ version:** `jj 0.43.0`

## Executive summary

JJ workspace provisioning is implemented, but JJ support is **not feature-complete and is not at parity with Git worktrees**.

The provisioning layer correctly detects JJ repositories, creates isolated JJ workspaces, reuses matching workspaces, checks JJ workspace dirtiness, and forgets workspaces during the normal removal path. The broader team lifecycle remains Git-oriented, however. A pure JJ repository cannot enable team workspace isolation through the main runtime, while a colocated JJ repository reaches Git-only checkpoint, integration, synchronization, shutdown, and stale-cleanup paths.

The practical result is that the current feature should be described as **JJ workspace provisioning support**, not end-to-end JJ team workspace support.

## Scope and methodology

This audit examined the team workspace lifecycle from planning through cleanup:

1. VCS detection and workspace planning.
2. Workspace creation and reuse.
3. Scale-up rollback after partial failure.
4. Worker checkpointing and leader integration.
5. Conflict handling and cross-worker synchronization.
6. Shutdown classification, preservation, reporting, and merge behavior.
7. Stale-team discovery and cleanup.
8. Persisted workspace metadata.
9. JJ-specific automated coverage.
10. Compatibility with current JJ behavior.

Primary files reviewed:

- `src/team/worktree.ts`
- `src/team/runtime.ts`
- `src/team/scaling.ts`
- `src/team/current-task-baseline.ts`
- `src/team/__tests__/worktree.test.ts`

## Parity matrix

| Capability | Git | JJ | Assessment |
|---|---:|---:|---|
| Detect repository type | Yes | Yes | Implemented; JJ takes precedence in colocated repositories |
| Plan worker workspace | Yes | Yes | Implemented |
| Create worker workspace | Yes | Yes | Implemented with `jj workspace add --revision` |
| Reuse matching workspace | Yes | Yes | Implemented with root/name validation |
| Reject dirty reuse | Yes | Yes | Implemented at provisioning level |
| Normal workspace removal | Yes | Yes | Implemented with `jj workspace forget` |
| Force workspace removal | Yes | Yes | Implemented with `jj workspace forget` |
| Team startup in a pure JJ repository | Yes | No | Runtime disables workspace mode when Git is unavailable |
| Scale-up failure rollback | Yes | Partial | Recovery proof is Git-only; a new JJ workspace can leak |
| Automatic worker checkpoint | Yes | No | Uses Git status/add/commit |
| Leader integration | Yes | No | Uses Git ancestry, merge, and cherry-pick |
| Conflict handling | Yes | No | Uses Git conflict inspection and abort operations |
| Cross-worker synchronization | Yes | No | Uses Git rebase |
| Shutdown dirty detection | Yes | No | Uses Git status |
| Shutdown preservation/integration | Yes | No | Uses Git commit, merge, diff, and revision queries |
| Stale-team discovery | Yes | Partial | Repository root and dirty detection are Git-only |
| Explicit VCS-aware persisted metadata | Git-shaped | No | Workspace/change identity is not modeled |
| End-to-end runtime tests | Yes | No | Only a provisioning happy path covers JJ |

## Findings

### P0 — Pure JJ teams silently lose workspace isolation

`resolveEffectiveTeamWorktreeMode()` enables requested team worktrees only when the leader directory is recognized as a Git repository:

- `src/team/runtime.ts:1762`

Its effective behavior is:

```ts
if (!isGitRepository(leaderCwd)) {
  return { enabled: false };
}
```

This conflicts with the generic provisioning layer, where `detectVcs()` probes JJ before Git:

- `src/team/worktree.ts:139`
- `src/team/worktree.ts:505`

A non-colocated JJ repository is therefore supported by `planWorktreeTarget()` and `ensureWorktree()`, but the main team runtime disables worktree mode before those functions can provide isolation. Workers fall back to the shared leader directory.

**Impact**

- The primary pure-JJ use case cannot use isolated team workspaces.
- The fallback is silent, so operators can believe isolation is active when it is not.
- Parallel workers can modify the same working copy despite requesting workspaces.

**Required direction**

Make effective mode resolution VCS-aware. It should accept any repository type supported by the provisioning layer and fail explicitly when requested isolation cannot be provided rather than silently disabling it.

---

### P0 — Worker checkpointing and leader integration are Git-only

The ongoing integration path models worker output as Git commits and branches:

- Commit range enumeration: `src/team/runtime.ts:998`
- Worker/leader integration: `src/team/runtime.ts:1157`
- Cross-worker rebase phase follows the integration phase in the same subsystem.

The implementation relies on Git concepts and operations including:

- `rev-parse`
- commit ranges
- ancestry checks
- `status --porcelain`
- `add -A`
- `commit`
- `merge`
- `cherry-pick`
- conflict-path inspection
- `merge --abort`
- `cherry-pick --abort`
- `rebase`
- `rebase --abort`

JJ workspaces do not represent worker branches. Each workspace has a working-copy change and JJ-managed change identity. Provisioning a JJ workspace does not make Git branch-based integration valid, even when the repository is colocated with Git.

**Impact**

- JJ worker changes cannot be reliably discovered or checkpointed by the team monitor.
- The leader cannot reliably integrate worker output.
- Conflict reporting and recovery do not follow JJ semantics.
- Cross-worker synchronization cannot update JJ workers safely.
- In colocated repositories, direct Git operations can move Git refs without modeling the JJ working-copy changes that users and workers see.

**Required direction**

Introduce a VCS lifecycle abstraction rather than invoking Git directly from runtime orchestration. JJ integration must operate on explicit workspace/change identities. The exact operation should be selected deliberately from JJ-native mechanisms such as `jj squash`, `jj rebase`, or a change-selection workflow; it must not assume that a JJ workspace corresponds to a Git branch.

---

### P0 — Shutdown cannot safely preserve or integrate JJ worker work

Shutdown reporting and merge preparation are Git-specific:

- Per-worker shutdown report: `src/team/runtime.ts:1548`
- Batch report preparation: `src/team/runtime.ts:1654`
- Dirty-worker classification: `src/team/runtime.ts:1715`

These paths use Git status, staging, commits, revision lookup, merge-base checks, merges, merge aborts, and Git diff output. Shutdown classification also decides whether the clean fast path is safe using Git status.

**Impact**

- JJ workspace changes may be misclassified as clean.
- Worker output may not be checkpointed before removal.
- The leader may not receive worker changes.
- Shutdown reports can omit or misrepresent JJ changes.
- Cleanup can proceed based on an incorrect preservation decision.

**Required direction**

Dispatch shutdown classification, checkpointing, integration, and report generation through the same VCS abstraction as the live integration loop. For JJ, preserve explicit change IDs and produce reports from JJ diffs/status rather than Git staging state.

---

### P1 — Scale-up failure recovery can orphan a newly created JJ workspace

When workspace creation succeeds but a later scale-up step fails, `recoverCreatedWorktreeAfterEnsureFailure()` attempts to prove that the new workspace belongs to the operation:

- `src/team/scaling.ts:275`

The proof uses only:

- `git rev-parse --git-common-dir`
- `git symbolic-ref -q HEAD`
- `git rev-parse HEAD`

A non-colocated JJ workspace cannot satisfy those checks. The function returns `null`, so the created workspace is not added to the collection passed to the otherwise JJ-aware rollback function.

**Impact**

- Failed scale-up can leave a workspace directory behind.
- The JJ workspace remains registered and can block later provisioning.
- Team state and filesystem artifacts can diverge.

**Required direction**

Dispatch recovery according to `plan.vcs`. For JJ, prove identity using the same repository root and workspace-name checks used by normal reuse, then return an `EnsureWorktreeResult` carrying `vcs: 'jj'` so rollback calls `jj workspace forget`.

---

### P1 — Stale-team discovery and dirty detection are Git-only

Stale-team cleanup resolves the repository root with Git before searching worker paths:

- `src/team/runtime.ts:5516`
- Git root query: `src/team/runtime.ts:5530`

If `git rev-parse --show-toplevel` fails, cleanup returns immediately. Dirty detection then calls `isWorktreeDirty()`:

- `src/team/runtime.ts:5547`
- `src/team/worktree.ts:189`

`isWorktreeDirty()` is Git-specific. A separate JJ-aware dirty helper exists in the provisioning layer, but stale cleanup does not dispatch to it.

**Impact**

- Pure JJ stale teams are not discovered or cleaned.
- JJ workspaces can remain registered indefinitely.
- Colocated repositories can display a dirty-work summary based on Git rather than JJ working-copy semantics.
- Confirmation prompts can understate work that will be removed.

**Required direction**

Use generic VCS/root detection and expose a public workspace-status abstraction. Persisted worker workspace paths should be preferred over reconstructing paths from a Git root and worker count.

---

### P2 — JJ compatibility requires a recent version but the requirement is implicit

JJ workspace discovery formats workspace records with a template containing `root`:

- JJ workspace listing is implemented near `src/team/worktree.ts:264`.

The effective command is:

```sh
jj workspace list -T 'name ++ "|" ++ root ++ "\n"'
```

`WorkspaceRef.root()` is a recent JJ template capability. The external audit identified JJ 0.40.0 as the release introducing it. The audited environment uses JJ 0.43.0, so the local happy path does not expose compatibility failures.

**Impact**

Older JJ installations can create or use repositories but fail during workspace identity, reuse, or cleanup operations with a template parse error.

**Required direction**

Choose one of these approaches:

1. Declare and validate a minimum supported JJ version at startup, currently `jj >= 0.40.0` if the template remains.
2. Replace the template dependency with a backward-compatible workspace discovery strategy.

Version errors should identify the installed and required versions and offer an actionable upgrade message.

---

### P2 — Persisted lifecycle state is Git-shaped

`CurrentTaskBaselineEntry` stores `branch_name`, optional `base_ref`, and PR metadata:

- `src/team/current-task-baseline.ts:8`
- Branch availability guard: `src/team/current-task-baseline.ts:115`

During JJ provisioning, a branch-like name is useful as a scheduler collision key, but no corresponding JJ branch or bookmark is created. JJ returns `createdBranch: false` while the broader state model remains branch-oriented.

**Impact**

- State fields imply VCS semantics they do not possess.
- Restart/recovery code lacks a persisted JJ workspace name and working-copy change ID.
- Future integration logic has no authoritative identity for selecting the worker change.

**Required direction**

Add an explicit VCS discriminator and VCS-specific identity fields. A compatible model could retain the existing scheduler key while distinguishing it from a Git branch:

```ts
interface WorkspaceIdentity {
  vcs: 'git' | 'jj';
  workspacePath: string;
  schedulerKey: string;
  git?: {
    branchName: string | null;
    baseRef: string;
  };
  jj?: {
    workspaceName: string;
    workingCopyChangeId: string;
    baseChangeId: string;
  };
}
```

The final schema should follow existing migration and compatibility conventions rather than replacing persisted fields without a migration.

---

### P2 — JJ automated coverage is limited to provisioning happy path

The sole focused JJ scenario is:

- `src/team/__tests__/worktree.test.ts:129`

It verifies that a non-colocated JJ repository can:

1. Plan a team workspace.
2. Create it.
3. Avoid registering it as a Git worktree.
4. Reuse it idempotently.

That test validates the provisioning foundation but not runtime parity.

**Missing coverage**

- Dirty reuse rejection.
- `allowDirtyReuse` behavior.
- Detached launch workspace creation/reuse.
- Existing directory conflicts.
- Mismatched workspace name/root rejection.
- Normal rollback invoking `jj workspace forget`.
- Force removal invoking `jj workspace forget`.
- Failure injected after successful creation during scale-up.
- Team startup mode resolution in a pure JJ repository.
- Live worker checkpointing and leader integration.
- JJ conflict behavior.
- Cross-worker synchronization.
- Shutdown classification and preservation.
- Stale-team cleanup in pure and colocated JJ repositories.
- Minimum-version validation or compatibility fallback.

**Required direction**

Add non-colocated JJ integration tests for every lifecycle path. Colocated cases should be supplemental tests because they can conceal direct Git assumptions that fail in pure JJ repositories.

## Correctly implemented foundation

The following behavior is sound and should be retained behind a generalized lifecycle interface.

### JJ is selected before Git

`detectVcs()` probes `jj root` before `git rev-parse --show-toplevel`:

- `src/team/worktree.ts:139`

This correctly gives JJ semantics precedence in colocated repositories.

### Workspace planning and creation are JJ-native

`planWorktreeTarget()` records `vcs: 'jj'`, and `ensureWorktree()` creates the workspace with `jj workspace add --revision`:

- `src/team/worktree.ts:505`
- `src/team/worktree.ts:529`

The base revision comes from JJ rather than a synthesized Git branch.

### Workspace reuse validates identity

The provisioning layer checks the expected workspace root and name before reuse and checks JJ workspace cleanliness unless dirty reuse is allowed.

This prevents a pre-existing unrelated directory from being silently adopted.

### Removal follows JJ workspace lifecycle

Both rollback and forced removal have JJ-aware behavior:

- `src/team/worktree.ts:728`
- `src/team/worktree.ts:789`

They call `jj workspace forget` before deleting the workspace path.

### Provisioning test uses a pure JJ repository

The current test creates and reuses a non-colocated JJ workspace and confirms it is not a Git worktree:

- `src/team/__tests__/worktree.test.ts:129`

This is the correct baseline environment for future parity tests.

## Recommended architecture

### 1. Centralize VCS detection and repository identity

Promote the current private detection behavior into a reusable service used by runtime, scaling, shutdown, and cleanup. The result should include the repository type and canonical root.

```ts
type RepositoryIdentity =
  | { vcs: 'git'; repoRoot: string; commonDir: string }
  | { vcs: 'jj'; repoRoot: string; storeRoot: string };
```

All lifecycle paths should consume this identity instead of probing Git independently.

### 2. Persist explicit workspace identity

Worker metadata should include enough information to prove and recover a workspace after partial failure or process restart:

- VCS kind.
- Canonical repository root.
- Workspace path.
- JJ workspace name or Git branch/detached identity.
- Worker base identity.
- Current JJ working-copy change ID where applicable.

Do not infer JJ identity from a branch-shaped scheduler name.

### 3. Define lifecycle operations by behavior, not Git commands

A focused interface could expose:

```ts
interface WorkspaceLifecycle {
  inspectWorkspace(identity: WorkspaceIdentity): WorkspaceStatus;
  checkpointWorker(identity: WorkspaceIdentity): CheckpointResult;
  integrateWorker(input: IntegrationInput): IntegrationResult;
  synchronizeWorker(input: SynchronizationInput): SynchronizationResult;
  createShutdownReport(identity: WorkspaceIdentity): ShutdownReport;
  removeWorkspace(identity: WorkspaceIdentity): Promise<void>;
}
```

The Git implementation can preserve current behavior. The JJ implementation should use explicit change IDs and JJ-native commands.

### 4. Make integration idempotent

Persist the last successfully integrated worker change/commit. A monitor retry must distinguish:

- No new worker output.
- New output ready to integrate.
- Output already integrated.
- Divergence requiring rebase or user intervention.
- Conflicted output.

This is especially important for JJ because change IDs remain stable while commit IDs can evolve after rewrites.

### 5. Fail closed when isolation or preservation cannot be guaranteed

Requested workspace isolation should never silently degrade to a shared working directory. Shutdown should never remove a workspace when its status cannot be determined or its changes cannot be preserved.

## Implementation sequence

### Phase 1 — Close lifecycle safety gaps

1. Make effective worktree-mode resolution recognize pure JJ repositories.
2. Make scale-up recovery dispatch by VCS and recover created JJ workspaces.
3. Make stale-team root/status discovery VCS-aware.
4. Add regression tests for all three paths.

These changes eliminate silent isolation loss and orphan cleanup issues without yet claiming integration parity.

### Phase 2 — Introduce VCS-aware state and interfaces

1. Add VCS identity to worker workspace metadata.
2. Persist JJ workspace name, base change ID, and working-copy change ID.
3. Extract Git lifecycle operations from `runtime.ts` behind a behavior-oriented interface.
4. Preserve compatibility with existing state files.

### Phase 3 — Implement JJ-native runtime integration

1. Detect worker change status.
2. Checkpoint or stabilize worker output without inventing Git branches.
3. Integrate a selected worker change into the leader.
4. Report conflicts using JJ conflict state.
5. Synchronize inactive workers onto the new leader state.
6. Record idempotent integration state.

The choice among `jj squash`, `jj rebase`, or another explicit flow should be validated against desired authorship, history shape, conflict behavior, and worker concurrency semantics before implementation.

### Phase 4 — Implement JJ-native shutdown

1. Determine dirty/pending JJ work accurately.
2. Preserve worker changes before cleanup.
3. Integrate or report non-integrated changes.
4. Generate JJ-native diff metadata.
5. Refuse destructive cleanup when preservation cannot be proven.

### Phase 5 — Complete compatibility and coverage

1. Enforce or eliminate the JJ 0.40.0 template requirement.
2. Add pure-JJ lifecycle integration tests.
3. Add colocated-JJ regression tests.
4. Add injected-failure tests around every mutation boundary.
5. Document supported JJ versions and lifecycle semantics.

## Acceptance criteria for declaring parity

JJ team workspace support should not be described as feature-complete until all of the following are true:

- [ ] Requested team workspaces are enabled in a pure JJ repository.
- [ ] Workers never silently fall back to the shared leader directory.
- [ ] A newly created JJ workspace is cleaned after every later startup/scale-up failure.
- [ ] Worker changes are detected using JJ semantics.
- [ ] Worker output can be integrated into the leader idempotently.
- [ ] Conflicts are surfaced and recoverable without direct Git conflict commands.
- [ ] Inactive workers can be synchronized to the new leader state.
- [ ] Shutdown preserves or explicitly reports every JJ worker change.
- [ ] Stale JJ teams are discoverable and removable in pure JJ repositories.
- [ ] Persisted state carries explicit JJ workspace/change identity.
- [ ] The supported JJ version is validated or older compatible behavior is implemented.
- [ ] Pure-JJ tests cover creation, reuse, dirty state, integration, conflict, shutdown, rollback, and stale cleanup.
- [ ] Colocated-JJ tests verify that no Git-only path bypasses JJ semantics.

## Verification notes

The source audit was performed against a clean working tree at revision `a2740ad601974fa9cfa9133c8a42d44585830eba` using `jj 0.43.0`.

A prior focused verification reported that the TypeScript build and worktree suite passed with **20/20 tests**, which supports the conclusion that provisioning itself works. During final report preparation, the repository build completed its TypeScript phase, but an attempted ad hoc test invocation referenced a nonexistent source-level runner, `scripts/run-test-files.mjs`. The actual runner is built from `src/scripts/run-test-files.ts` to `dist/scripts/run-test-files.js`; therefore that invocation provides no additional test evidence and does not indicate a product failure.

No product source code was modified as part of this audit.

## Final assessment

**Status: incomplete, with P0 parity blockers.**

JJ support currently provides a credible workspace-provisioning foundation, including correct JJ-first detection and JJ-aware removal. It does not yet provide a safe end-to-end team lifecycle. Pure JJ repositories lose requested isolation, and all consequential post-provisioning paths—checkpointing, integration, synchronization, shutdown, and parts of cleanup—retain Git assumptions.

Until the P0 findings are resolved, the supported capability should be documented narrowly as **experimental JJ workspace provisioning**, not Git-equivalent JJ team workspaces.
