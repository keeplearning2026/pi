# GitHub Supervisor Agent

Status: Draft

## 1. Purpose

This document defines a GitHub-native development automation loop that combines:

- ChatGPT as the high-reasoning supervisor;
- a local orchestrator as the deterministic control plane;
- Claude Code as the local implementation agent;
- GitHub as the authoritative task, source, review, and state bus;
- GitHub Actions as an independent verification layer;
- optional Git bundles as immutable repository snapshots and connector fallbacks.

The system is an agent system, but not a single autonomous coding agent. It is a supervised multi-agent workflow with explicit authority boundaries:

- ChatGPT decides architecture, writes executable plans, reviews candidate implementations, and returns constrained decisions.
- Claude Code reads and edits the local repository, responds to compiler feedback, and implements approved plans.
- The local orchestrator owns state transitions, worktrees, command execution, evidence collection, Git operations, retries, and recovery.
- GitHub stores all authoritative task inputs and outputs.

The first user interaction creates or approves the initial plan. After that, the system should continue automatically until one of these terminal conditions occurs:

- the candidate is approved;
- the plan is invalid and requires replanning;
- a human decision is required;
- a safety or retry limit is reached.

## 2. Non-goals

The first implementation does not:

- let ChatGPT edit source files directly;
- let Claude Code decide architecture or task completion;
- merge directly into a protected branch without repository policy checks;
- trust natural-language summaries as execution evidence;
- scrape ChatGPT response text from the browser;
- depend on ChatGPT scheduled tasks;
- require a model API for the supervisor path;
- place Git bundles in normal source history;
- modify Pi core agent semantics unless a later implementation proves that a reusable primitive belongs there.

## 3. Design principles

### 3.1 GitHub is the system of record

All persistent workflow facts must be reconstructable from GitHub:

- task definition;
- approved plan and plan revision;
- baseline commit;
- candidate commits;
- pull request diff;
- execution evidence;
- CI results;
- supervisor reviews;
- state labels;
- correction rounds;
- final approval.

The local orchestrator may keep a cache, but GitHub remains authoritative after restart.

### 3.2 Models do not own workflow state

ChatGPT and Claude Code may propose state changes, but only the local orchestrator applies or accepts state transitions after validating protocol fields and current GitHub state.

### 3.3 Every decision is commit-bound

A supervisor decision is valid only for the exact candidate commit it reviewed.

If the pull request head changes after review, the old decision is stale and must not approve the new head.

### 3.4 Deterministic operations stay outside the model loop

The local orchestrator directly performs:

- Git status and baseline checks;
- worktree creation and cleanup;
- allowed-file checks;
- required command execution;
- timeout handling;
- exit-code collection;
- diff checks;
- evidence serialization;
- branch and pull request operations;
- state transition validation.

Models should not spend tokens deciding how to run fixed commands.

### 3.5 Repository content is untrusted input

Issue text, pull request text, source comments, logs, test output, and generated evidence may contain prompt-injection instructions.

The supervisor must treat them as task data, not authority. Only the project supervisor instructions and the authenticated machine manifest define allowed actions.

## 4. Component responsibilities

## 4.1 ChatGPT supervisor

The ChatGPT supervisor has two modes.

### Plan mode

The user starts the initial task manually. ChatGPT:

1. reads the repository through the GitHub connector and, when supplied, a Git bundle;
2. confirms the baseline commit;
3. traces the relevant call graph and state ownership;
4. separates confirmed facts, inferences, and unknowns;
5. identifies the root cause or implementation gap;
6. selects one implementation design;
7. writes a multi-step executable plan with decision criteria;
8. creates or updates the GitHub task issue;
9. applies the `agent-state:plan-ready` label.

### Review mode

A browser trigger submits a short instruction to a fixed ChatGPT project conversation. ChatGPT:

1. finds one `agent-state:review-ready` task;
2. claims it by transitioning the label to `agent-state:reviewing`;
3. reads the approved plan, pull request, diff, candidate commit, CI status, and evidence;
4. reviews the actual implementation rather than the executor summary;
5. submits a commit-bound GitHub pull request review;
6. sets exactly one next-state label.

The allowed supervisor decisions are:

- `APPROVE`;
- `CORRECT`;
- `REPLAN`;
- `HUMAN_REQUIRED`.

ChatGPT must not create source commits, update refs, merge pull requests, or delete files in the automatic supervisor loop.

## 4.2 Local orchestrator

The local orchestrator is the workflow authority and deterministic runner. It:

- polls or receives GitHub task state;
- parses and validates machine markers;
- enforces legal state transitions;
- validates baseline and candidate commits;
- creates isolated Git worktrees;
- starts and resumes Claude Code sessions;
- enforces file-scope and command policies;
- runs required verification independently of Claude;
- collects logs and hashes artifacts;
- commits and pushes candidate branches;
- creates or updates pull requests;
- publishes execution evidence;
- triggers the ChatGPT browser conversation;
- consumes structured supervisor reviews;
- resumes corrections automatically;
- applies retry limits and circuit breakers;
- restores state after process restart.

The orchestrator must never infer approval from prose. It only accepts a valid machine marker attached to a review of the current pull request head.

## 4.3 Claude Code executor

Claude Code implements the approved plan in a local worktree. It:

- reads the plan and repository rules;
- reads the files and symbols required by the plan;
- edits only allowed paths;
- writes the specified tests;
- handles local syntax, import, type, and ownership adjustments;
- runs targeted commands when useful for iterative implementation;
- reports plan deviations and unresolved blockers.

Claude Code does not:

- alter the selected architecture;
- weaken tests;
- expand scope without an allowed fallback;
- push to protected branches;
- merge pull requests;
- declare the task approved;
- change state labels directly.

## 4.4 GitHub

GitHub provides:

- issue-based task manifests;
- plan history;
- state labels;
- candidate branches;
- pull request diffs;
- reviews and inline findings;
- CI and artifact links;
- immutable commit identities;
- an auditable workflow history.

## 4.5 GitHub Actions

GitHub Actions is an independent verifier, not a reasoning agent. It runs required build, test, lint, packaging, and platform-matrix checks.

A ChatGPT `APPROVE` decision does not override required status checks.

## 4.6 Browser trigger

Browser automation only triggers the fixed ChatGPT supervisor conversation.

It:

- opens or activates the configured ChatGPT project conversation;
- locates the message input using DOM or accessibility selectors;
- submits a short fixed instruction;
- does not parse or scrape ChatGPT output;
- waits for GitHub state changes instead.

The trigger instruction should remain stable, for example:

> Process the next `agent-state:review-ready` task using the project supervisor protocol. Review the authoritative GitHub plan, pull request, diff, CI, and execution evidence, then write the structured decision back to GitHub.

## 5. Pi reuse boundary

Pi provides useful primitives, but this workflow should initially remain outside Pi core.

Reusable Pi concepts include:

- agent session and tool-call loops;
- RPC-based coding-agent processes;
- persistent session identifiers;
- process lifecycle and unexpected-exit handling;
- supervisor-managed live instances;
- restart recovery;
- event subscribers;
- provider abstraction.

The current `OrchestratorSupervisor` manages Pi process instances and session metadata. It should not be expanded directly into a GitHub development workflow state machine. The responsibilities differ:

- Pi instance status describes process lifecycle;
- development task status describes repository workflow lifecycle.

The first implementation should therefore use one of these boundaries:

1. a standalone repository consuming Pi packages;
2. an experimental package adjacent to `packages/orchestrator`;
3. an extension that starts or communicates with Pi coding-agent sessions.

A later refactor may extract reusable supervisor primitives after at least one complete implementation proves the interface.

## 6. State model

Every managed task has exactly one primary state label.

### 6.1 Primary state labels

- `agent-state:plan-ready`
- `agent-state:claimed`
- `agent-state:executing`
- `agent-state:local-verifying`
- `agent-state:review-ready`
- `agent-state:reviewing`
- `agent-state:correction-ready`
- `agent-state:approved`
- `agent-state:replan-required`
- `agent-state:human-required`
- `agent-state:failed`
- `agent-state:completed`

Optional non-state labels:

- `agent-task:managed`
- `agent-risk:low`
- `agent-risk:medium`
- `agent-risk:high`

### 6.2 Legal transitions

Normal execution:

    PLAN_READY
      -> CLAIMED
      -> EXECUTING
      -> LOCAL_VERIFYING
      -> REVIEW_READY
      -> REVIEWING

Review outcomes:

    REVIEWING -> APPROVED -> COMPLETED
    REVIEWING -> CORRECTION_READY -> EXECUTING
    REVIEWING -> REPLAN_REQUIRED
    REVIEWING -> HUMAN_REQUIRED
    REVIEWING -> FAILED

Recovery transitions:

    CLAIMED -> PLAN_READY
        only when the claim lease expires before local changes begin

    EXECUTING -> FAILED
    LOCAL_VERIFYING -> FAILED
        only after retry policy is exhausted or a mandatory stop condition occurs

No other transition is legal.

### 6.3 Transition invariants

Before applying a transition, the orchestrator verifies:

- exactly one current state label exists;
- the transition is legal;
- the task ID and repository match;
- the plan revision matches the issue manifest;
- the baseline commit still exists;
- the current pull request head matches the candidate commit where required;
- the request ID has not already been consumed;
- correction and review limits have not been exceeded.

## 7. GitHub protocol

Machine-readable data is embedded in HTML comments so the GitHub page remains readable while the orchestrator can parse a strict payload.

Natural-language text is descriptive. Hidden JSON is authoritative.

## 7.1 Plan marker

Marker name:

    DEV_AGENT_PLAN_V1

Required fields:

```json
{
  "protocol_version": 1,
  "task_id": "CCR-142",
  "repository": "keeplearning2026/commandcode-router",
  "baseline_sha": "abc1234",
  "target_branch": "main",
  "plan_revision": 1,
  "risk": "high",
  "allowed_files": [
    "src/example.ts",
    "test/example.test.ts"
  ],
  "forbidden_files": [
    "package-lock.json"
  ],
  "required_verification": [
    {
      "id": "target-test",
      "argv": ["npm", "test", "--", "target-test"],
      "expected_exit_code": 0
    }
  ]
}
```

The human-readable issue body contains the complete executable plan, including root cause, selected solution, rejected alternatives, ordered implementation steps, decision rules, allowed fallbacks, mandatory stop conditions, expected diff shape, and evidence requirements.

## 7.2 Evidence marker

Marker name:

    DEV_AGENT_EVIDENCE_V1

Required fields:

```json
{
  "protocol_version": 1,
  "task_id": "CCR-142",
  "request_id": "CCR-142:def5678:1",
  "baseline_sha": "abc1234",
  "candidate_sha": "def5678",
  "plan_revision": 1,
  "execution_round": 1,
  "changed_files": [
    "src/example.ts",
    "test/example.test.ts"
  ],
  "plan_deviations": [],
  "verification": [
    {
      "id": "target-test",
      "exit_code": 0,
      "duration_ms": 18342,
      "timed_out": false,
      "cancelled": false,
      "log_sha256": "...",
      "artifact": "..."
    }
  ],
  "unverified": []
}
```

The orchestrator generates this payload from observed commands and Git state. Claude does not author authoritative exit codes or commit identities.

## 7.3 Supervisor review marker

Marker name:

    CHATGPT_SUPERVISOR_REVIEW_V1

Required fields:

```json
{
  "protocol_version": 1,
  "task_id": "CCR-142",
  "request_id": "CCR-142:def5678:1",
  "reviewed_sha": "def5678",
  "plan_revision": 1,
  "review_round": 1,
  "decision": "CORRECT"
}
```

For `CORRECT`, the review body must include:

- concrete findings tied to files and symbols;
- the unchanged design contract;
- allowed files for the correction;
- ordered correction actions;
- forbidden approaches;
- required verification;
- success and stop criteria.

For `APPROVE`, the review body must state:

- the reviewed candidate commit;
- whether all required evidence was available;
- whether CI was complete;
- any remaining non-blocking risk.

For `REPLAN` and `HUMAN_REQUIRED`, the review body must explain why automatic correction is unsafe.

## 8. Claiming and concurrency

Only one orchestrator instance may own a task execution lease.

A claim record must include:

- task ID;
- owner instance ID;
- claim nonce;
- claim timestamp;
- lease expiry;
- plan revision.

The claim may be stored in an issue comment marker or a dedicated state file on an agent metadata branch.

A second orchestrator must not execute the task while a valid lease exists.

The reviewer uses the state-label transition from `review-ready` to `reviewing` as the review claim. A request ID makes repeated browser triggers idempotent.

## 9. Execution lifecycle

## 9.1 Initial planning

1. The user starts a plan request in ChatGPT.
2. ChatGPT reads the repository and optional baseline Git bundle.
3. ChatGPT writes the task issue and `DEV_AGENT_PLAN_V1` marker.
4. ChatGPT applies `agent-task:managed` and `agent-state:plan-ready`.

## 9.2 Local execution

1. The orchestrator finds a `plan-ready` issue.
2. It validates the marker and current baseline.
3. It claims the task.
4. It creates an isolated worktree and `agent/<task-id>` branch.
5. It starts Claude Code with the approved plan and executor rules.
6. Claude performs the implementation.
7. The orchestrator runs required checks independently.
8. The orchestrator rejects forbidden or unexpected file changes.
9. It creates a candidate commit and pushes the task branch.
10. It creates or updates a draft pull request.
11. It publishes the evidence marker and human-readable summary.
12. It transitions to `review-ready`.
13. It invokes the browser trigger.

## 9.3 Supervisor review

1. ChatGPT claims one `review-ready` task.
2. It confirms the pull request head and candidate SHA.
3. It reads the original plan and all later plan revisions.
4. It reads the full diff and relevant source context.
5. It checks the evidence and GitHub Actions.
6. It evaluates implementation fidelity, test quality, scope, invariants, and remaining risk.
7. It posts one commit-bound pull request review.
8. It applies exactly one next-state label.

## 9.4 Correction loop

1. The orchestrator reads a valid `CORRECT` review.
2. It verifies that `reviewed_sha` equals the pull request head being corrected.
3. It resumes the Claude session or starts a correction session using the review body.
4. It applies the correction under the original plan design.
5. It reruns all required checks.
6. It pushes a new candidate commit.
7. It publishes new evidence with a new request ID.
8. It transitions back to `review-ready`.

## 9.5 Completion

A task may become `completed` only when:

- the latest supervisor review is `APPROVE`;
- `reviewed_sha` equals the current pull request head;
- all required GitHub checks are successful;
- branch protection requirements are satisfied;
- the final diff contains no forbidden files or untracked artifacts;
- repository policy allows the chosen merge or handoff action.

Automatic merge is an optional later feature and must be risk-gated.

## 10. Failure classification

The orchestrator classifies failures before deciding whether to retry.

### 10.1 Local implementation failure

Examples:

- compiler failure introduced by the candidate;
- target test failure;
- forbidden path modification;
- Claude stop condition;
- invalid executor output.

Action:

- retry only within configured local limits;
- otherwise set `failed` and publish evidence.

### 10.2 Plan invalidation

Examples:

- baseline no longer matches;
- target symbol or responsibility has moved;
- the root cause cannot be reproduced;
- the change requires a public API or dependency decision excluded by the plan;
- the selected design cannot be implemented within allowed scope.

Action:

- set `replan-required`;
- stop Claude execution;
- notify the user.

### 10.3 Environment failure

Examples:

- missing SDK;
- unavailable external service;
- expired credentials;
- target platform unavailable;
- GitHub outage.

Action:

- preserve logs;
- do not describe the code as verified;
- retry only when policy permits;
- otherwise set `human-required` or `failed`.

### 10.4 Supervisor failure

Examples:

- GitHub connector cannot read required files;
- review is not written back;
- malformed review marker;
- stale reviewed SHA;
- browser trigger fails repeatedly.

Action:

- leave the task recoverable in `review-ready` or `human-required`;
- never infer approval.

## 11. Circuit breakers

Initial defaults:

- maximum correction rounds: 3;
- maximum review rounds: 4;
- maximum candidate commits per task: 8;
- maximum automatic elapsed time: 24 hours;
- maximum one active task per repository in the MVP;
- maximum changed files: plan-defined;
- maximum diff size: plan-defined.

Exceeding a limit transitions to `human-required`.

## 12. Security controls

### 12.1 GitHub permissions

The automatic ChatGPT supervisor should normally be limited to:

- read repository, issue, pull request, commit, and Actions data;
- create or update issue comments;
- add or remove task labels;
- submit pull request reviews;
- reply to review threads;
- rerun explicitly classified transient CI failures.

It should not automatically:

- create or update source files;
- create commits or trees;
- move refs;
- delete files;
- merge pull requests;
- enable auto-merge;
- change pull request base branches.

The local orchestrator owns branch, commit, push, and pull request creation.

### 12.2 Protected branches

The orchestrator must reject direct work on protected branches. Task branches use a configured prefix such as `agent/`.

Force push is prohibited.

### 12.3 Allowed-file enforcement

The orchestrator compares the actual changed-file set against the plan. Unexpected changes block publication unless the plan contains an explicit fallback permitting them.

### 12.4 Command policy

Commands execute as argument arrays, not shell-concatenated strings, unless a task explicitly requires a shell script.

The orchestrator applies timeouts, captures stdout and stderr separately, and records cancellation state.

### 12.5 Secret handling

Evidence must redact secrets before publication. Full logs containing credentials stay local or in access-controlled artifacts.

## 13. Git bundle support

Git bundles are optional immutable snapshots, not the workflow state bus.

### 13.1 Baseline bundle

A baseline bundle can be uploaded during initial ChatGPT planning when connector search is insufficient.

The supervisor must clone it into a temporary workspace and verify that its head matches the plan baseline.

### 13.2 Candidate bundle

A candidate bundle can be generated after committing local changes when:

- the GitHub connector cannot see the latest commit;
- a full-repository search is needed;
- deep history inspection is required;
- an immutable offline review snapshot is desired.

The candidate bundle head must equal the pull request candidate SHA.

### 13.3 Storage

Bundles must not be committed to the normal source branch. They belong in local task artifacts, restricted workflow artifacts, release assets, or a dedicated artifact store.

## 14. Persistence and recovery

The local orchestrator maintains a local cache for efficiency, but reconstructs authoritative state from GitHub after restart.

Local task state may include:

```json
{
  "task_id": "CCR-142",
  "repository": "keeplearning2026/commandcode-router",
  "issue_number": 142,
  "pull_request_number": 216,
  "plan_revision": 1,
  "baseline_sha": "abc1234",
  "candidate_sha": "def5678",
  "state": "REVIEW_READY",
  "execution_round": 1,
  "review_round": 0,
  "claude_session_id": "...",
  "worktree": "D:/agent-worktrees/CCR-142"
}
```

On startup, the orchestrator reconciles local state with:

- issue labels and plan marker;
- pull request state and head SHA;
- latest evidence marker;
- latest valid supervisor review;
- active GitHub checks.

Conflicts stop automatic execution rather than selecting one side silently.

## 15. Proposed implementation boundary

The MVP should be built as a standalone TypeScript application that may consume Pi packages. It should not initially modify Pi core.

Suggested modules:

    src/domain/task.ts
    src/domain/state-machine.ts
    src/protocol/markers.ts
    src/protocol/schemas.ts
    src/github/client.ts
    src/github/task-store.ts
    src/workspace/worktree.ts
    src/executor/claude.ts
    src/verification/runner.ts
    src/evidence/collector.ts
    src/browser/chatgpt-trigger.ts
    src/orchestrator.ts
    src/cli.ts

Pi integration can begin through the coding-agent RPC or CLI boundary. Direct imports from experimental Pi orchestrator internals should be avoided until their interface is intentionally stabilized.

## 16. MVP milestones

### M0: Protocol and state machine

Implement:

- plan, evidence, and review schemas;
- marker parser and serializer;
- legal transition table;
- SHA and revision validation;
- idempotency checks;
- unit tests for valid and invalid transitions.

No Claude or browser integration.

### M1: GitHub task store

Implement:

- issue discovery;
- label transitions;
- plan parsing;
- pull request lookup;
- evidence publication;
- supervisor review consumption;
- restart reconciliation.

### M2: Mock executor vertical slice

Implement a mock executor that:

- creates a worktree and task branch;
- makes one deterministic test change in a smoke-test repository;
- creates a candidate commit and draft pull request;
- publishes evidence;
- consumes a manually submitted supervisor review.

This milestone proves the GitHub-only control loop.

### M3: Claude Code executor

Implement:

- non-interactive Claude invocation;
- session ID persistence and resume;
- executor rules injection;
- bounded turns and timeout;
- structured executor summary;
- stop-condition handling.

### M4: Deterministic verification

Implement:

- command execution from plan arrays;
- stdout and stderr artifact capture;
- exit-code and duration recording;
- changed-file and diff checks;
- evidence generation independent of Claude claims.

### M5: ChatGPT browser trigger

Implement:

- fixed project conversation configuration;
- DOM or accessibility-based input selection;
- stable trigger submission;
- no response scraping;
- retry and browser-session health checks.

### M6: Automatic correction loop

Implement:

- structured `CORRECT` review parsing;
- Claude session resume;
- new request IDs and evidence rounds;
- stale-review rejection;
- circuit breakers.

### M7: Git bundle support

Implement:

- full baseline and candidate bundle creation;
- bundle verification;
- artifact hashing;
- optional browser attachment workflow;
- candidate-head equality checks.

### M8: Concurrency and multi-repository support

Implement:

- per-repository execution locks;
- bounded worker pools;
- claim leases;
- fair task ordering;
- repository-specific policies.

## 17. MVP acceptance criteria

The MVP is accepted only when a smoke-test repository can complete this sequence:

1. A human-approved plan exists in a GitHub issue.
2. The orchestrator claims the issue and creates an isolated task branch.
3. The mock executor makes a deterministic change.
4. The orchestrator independently verifies the change.
5. A draft pull request and evidence marker are created.
6. ChatGPT or a human submits a valid commit-bound review marker.
7. The orchestrator consumes `CORRECT` and performs one correction round, or consumes `APPROVE` and moves to completion.
8. A stale review for an older SHA is rejected.
9. Restarting the orchestrator reconstructs the current task state from GitHub.
10. No direct changes are made to the protected branch.

## 18. Open implementation questions

These questions remain implementation choices and must be resolved before M1 or M3, not left to the executor at runtime:

- whether the standalone implementation uses the GitHub REST API, `gh`, or a hybrid;
- whether claim leases live in comments, issue body metadata, or a metadata branch;
- whether Claude is invoked through CLI, Pi RPC, or a dedicated adapter;
- whether task artifacts are local-only or uploaded to GitHub Actions;
- whether browser automation uses Playwright, Windows UI Automation, or another accessibility-aware driver;
- which GitHub write actions are enabled for unattended ChatGPT use;
- whether low-risk approved tasks may auto-merge after branch protection checks.

The first implementation should prefer the smallest reversible choice and preserve the protocol boundary so these components remain replaceable.
