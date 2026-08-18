# CodeRabbit Format Reference

Copy-ready templates for producing a CodeRabbit-replica review. Use these exactly; adjust only the content.

## 1. Badge Taxonomy

Every finding gets three badges, combined on one line. Category → severity → effort:

| Badge | Values | Emoji |
|---|---|---|
| **Category** | 🎯 Functional Correctness · 🩺 Stability & Availability · 🔒 Security · ⚡ Performance · 📐 Maintainability & Code Quality · 🧪 Tests | — |
| **Severity** | 🔴 Critical · 🟠 Major · 🟡 Minor · 🔵 Trivial | — |
| **Effort** | ⚡ Quick win · 🔨 Medium · 🏗️ Heavy lift | — |

Severity guidance:
- **Critical** — data loss, security hole, crashes in production path, breaks the build.
- **Major** — definite bug or correctness issue, resource leak, race, missing error handling that will bite.
- **Minor** — edge case not handled, minor inefficiency, maintainability smell with real impact.
- **Trivial** — style, naming, nitpicks. These go in the **Nitpicks** group, not as separate inline comments.

---

## 2. Walkthrough Comment (issue comment on the PR)

Posted as an **issue comment** (not part of the review) via `add_issue_comment`. Structure:

```markdown
## Walkthrough
> [!NOTE]
> This is a solid PR. The refactor of `TaskStore` into a dedicated persistence layer with typed row-mapping is clean and well-tested. The new `OrchestratorService` state machine is easy to follow, and the heartbeat-staleness detection covers the needs-input flow correctly. Main concern: the scheduler only runs immediately at boot — tasks created mid-flight wait up to 3 minutes.

<details>
<summary><strong>Changes</strong></summary>

| Layer / File(s) | Summary |
|---|---|
| `lib/src/models/task.dart` | Adds `TaskStatus` enum, state-machine transition table, and heartbeat-staleness helpers. |
| `lib/src/store/task_store.dart` | SQLite-backed persistence with typed row mapping and transition-guarded updates. |
| `lib/src/scheduler/task_scheduler.dart` | `OrchestratorService` — create/cancel/pause/resume/message, stale-heartbeat detection, queued-task assignment. |
| `lib/src/adapters/` | Pluggable `WorkerAdapter` contract + `FullPower` (PR) and `Deliverable` (patch/tarball) implementations. |
| `lib/src/api/server.dart` | Shelf router: `/health`, `/tasks` CRUD + control endpoints, artifact serving. |
| `test/` | 47 tests across models, store, scheduler, adapters, and API. |

</details>

### Estimated Review Effort: 3 (30 minutes)

| Category of Change | Complexity |
|---|---|
| Architecture & state machine | 🔨 Medium |
| SQLite persistence layer | 🔨 Medium |
| HTTP API surface | ⚡ Low |

### Pre-Merge Checks

| Check | Status |
|---|---|
| `dart analyze` | ✅ Clean (1 warning — unused import, see comment) |
| `dart test` | ✅ 47/47 passing |
| Server boots + `/health` | ✅ 200 OK |
| `.dart_tool/` tracked in git | ❌ Should be removed + gitignored |
| `.gitignore` present | ❌ Missing |

> **Note for reviewers:** the sandbox-absolute paths inside the committed `.dart_tool/package_config.json` will break any local tooling that reads it. Remove before merge.

<details>
<summary>🧸 Poetry</summary>

```
🐰 Hop, hop, hop through the code review trail,
   Tasks queue and run without a fail.
   Heartbeats tick, staleness caught,
   This orchestrator's worth the thought! 🥕
```
</details>
```

---

## 3. Inline Review Comment (one per actionable finding)

Added to the **pending review** via `add_comment_to_pending_review`, anchored to exact diff lines (`subjectType: LINE`, `side: RIGHT`, `startLine`/`line`). Template:

```markdown
🧪 Tests · 🟠 Major · ⚡ Quick win

**Missing test coverage for the invalid-transition path**

The state machine guards `queued → done` (correct), but there's no test asserting `TaskStore.updateStatus` *rejects* an invalid transition end-to-end through the service. `test/src/store/task_store_test.dart` covers `updateStatus` returning `false`, but the service-level path (`cancelTask` on a `done` task → 409) is untested.

The API returns 409 for an uncancellable task, but nothing exercises that branch — a future refactor could silently allow it.

<details>
<summary>Proposed fix</summary>

In `test/src/api/api_test.dart`, add a case where the task is already `done`, then:

```dart
final resp = await call(api, 'POST', '/tasks/${doneTask.taskId}/cancel');
expect(resp.statusCode, equals(409));
```

</details>

🤖 **Prompt for AI Agents**

```
In test/src/api/api_test.dart, add a test: create a task, mark it done
(store.updateStatus(taskId, TaskStatus.done)), then POST /tasks/<id>/cancel
and assert the response is 409 (not 200). Run `dart test` to confirm.
```
```

Optional **committable suggestion** (only when the range exactly covers the replaced lines — verify against real file contents before posting):

````markdown
🧪 Tests · 🟠 Major · ⚡ Quick win

**Missing test coverage for the invalid-transition path**

The API returns 409 for an uncancellable task, but nothing exercises that branch.

```suggestion
  test('POST /tasks/:id/cancel on done task returns 409', () async {
    final t = service.createTask(workerType: 'mock', spec: 'go', repo: 'a/b');
    store.updateStatus(t.taskId, TaskStatus.done);
    final resp = await call(api, 'POST', '/tasks/${t.taskId}/cancel');
    expect(resp.statusCode, equals(409));
  });
```
````

---

## 4. Review Status Summary (submitted review body)

Submitted via `pull_request_review_write` (`submit_pending`, `event: COMMENT`). Structure:

```markdown
### Actionable comments posted: 3

- 🎯 Functional Correctness · 🟠 Major · 🔨 Medium — `lib/src/scheduler/task_scheduler.dart:87` — `cancelTask` on a `paused` task loses the pause state
- 🩺 Stability & Availability · 🟠 Major · ⚡ Quick win — `lib/src/store/task_store.dart:104` — `_decodeResult` swallows empty-string values
- 🧪 Tests · 🟠 Major · ⚡ Quick win — `test/src/api/api_test.dart` — missing 409-coverage for cancel-on-done

### Nitpicks

> 🔵 Trivial — `lib/src/models/task.dart:1` — unused `dart:convert` import; remove it.
>
> 🔵 Trivial — `bin/server.dart` — unused `_eventBus` field; consider wiring it to a logs endpoint or removing.

### Prompt for all review comments

🤖 **Prompt for AI Agents**

```
1. lib/src/scheduler/task_scheduler.dart:87 — when cancelling a paused task,
   keep the paused status instead of overwriting with cancelled...
2. lib/src/store/task_store.dart:104 — _decodeResult: skip empty values...
3. test/src/api/api_test.dart — add cancel-on-done 409 test...
```

### Verdict

> The implementation is well-structured and the state machine is sound — the core is ready to merge once the committed `.dart_tool/` is removed, the unused import is dropped, and the two correctness nits (pause-cancel, empty-result decode) are addressed. 🐰
```

---

## 5. Clean-Diff Celebration

If there are no actionable findings, still post a short review — don't invent issues:

```markdown
### Actionable comments posted: 0

🎉 No actionable comments — this diff is clean.

### Verdict

> Well-structured change. The state machine, tests, and API surface all check out. Thanks for the care here. 🐰
```

---

## 6. Prompt for AI Agents (canonical form)

Self-contained, so a fresh agent can act on it without the diff context:

```markdown
🤖 **Prompt for AI Agents**

```
In <file>:<lines>, <what's wrong> because <impact>. Fix by <concrete change>.
Verify with <command>.
```
```

One prompt per inline comment (self-contained), plus one combined prompt in the review summary listing all comments.

---

## 7. Ordering Rules

- Inline comments: **Critical → Major → Minor**; Trivial items go in the **Nitpicks** group of the review summary only.
- Walkthrough summary table: order by layer (models → store → scheduler → adapters → API → tests).
- Review effort: 1 (≤10 min) → 2 (15) → 3 (30) → 4 (45) → 5 (60+), with per-category complexity chips.
