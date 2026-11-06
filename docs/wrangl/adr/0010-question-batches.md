# ADR 0010: Question batches: answer an agent's question list one by one from the Inbox

- Status: **Proposed, not built.** Nothing here exists in the code. The owner reads and decides before it merges.
- Task: LqArqs
- Builds on: [ADR 0006 (action state machine)](0006-action-state-machine.md) for the card states; migration 00036 (`tool_error`, the pattern for adding a question kind or column)
- Touches (future work): `stacktools/tools.go`, `controller/runs/{tools,questions}.go`, `runner/{agentsock,work,daemon}.go`, `controller/{plans,deliverables,taskreview}`, `db/queries/question.sql`, `web/inbox.go`, `cli/question.go`, migrations 00037+

Design only; no application code changes. Paths are under `cmd/wrangl/internal/`.

## 1. Context

Task hpCqdq had a spec with eight blocking questions (Q1 to Q8), and its spec and plan approvals lived inside spec/plan documents. The agent folded everything into one free-text `ask_question`. The owner could not answer Q1 to Q8 separately, could not see recommended defaults as choices, and the task blocked.

### 1.1 What the code does today (verified)

1. **`ask_question` takes exactly one question.** Schema in `stacktools/tools.go`: `prompt`, `detail_md`, `allow_free_text`, `options[{label,value,description}]`. `controller/runs/tools.go` (`case "ask_question", SystemQuestionTool`) creates one `question` row of kind `agent` (or `system` for the daemon-only `_system_question`) through `runs.NewQuestion`, which also sets the run `needs_input`. There is no way to send a list, no "recommended" marker, no grouping column.
2. **Nothing parses a spec.** There is no code that reads a spec or plan document for questions. `task_complete` (`Stack.AgentComplete`) takes a sqid and a summary and moves the task to `in_review`. Questions that live in a document stay in the document.
3. **Review items exist, but only for outputs.** `plan_review` is created by `plans.ProposeSplit` (the `propose_split` tool of a plan run; options Accept, Edit, Reject; payload is the proposal). `deliverable_review` is created by `deliverables.OnRunFinished` after a work or revise run. `task_review` is created by `taskreview` (a periodic sync that opens "Ready for review" / "Run failed" cards and closes them when the task moves on). None of them is raised from a spec the agent wrote; "approve the spec" and "approve the plan" are not explicit items.
4. **The block is an in-memory channel in the runner.** `Worker.ToolFor` (`runner/agentsock.go`) sends the tool call to the controller (30 s request timeout), reads `question_id` from the result, registers a channel in `waiters`, and blocks until `DeliverAnswer`, the tool context ends, or the run ends. The agent's MCP call is open the whole time.
5. **Answers are delivered once, to a live waiter.** `runs.Answer` marks the row answered, then (for kinds `agent` and `system`) `sendAnswer` publishes `runner.<id>.answer`. The daemon calls `worker.DeliverAnswer`; if no waiter exists it only logs "answer for a question nobody is waiting on" (`runner/daemon.go`). `sendAnswer` also returns silently when the run is terminal.
6. **Late answers are lost, three ways.**
   - The run ended: `waiters.cancelRun` and the controller's `CancelRunQuestions` (`kind IN ('agent','system','tool_error')`) cancel the question, so a later `Answer` gets `Conflict: question N is cancelled`. The owner's text is not stored anywhere.
   - The tool call ended first (agent turn timeout, hook cancel, MCP client timeout, runner restart): the `ctx.Done()` branch returns an error but does not remove the waiter. The question stays `open`, the owner can still answer, `Answer` succeeds and marks it answered, and the delivery lands in a channel nobody reads (or the waiter is gone after a restart): logged and dropped. The row says "answered", the agent never saw it.
   - Retry: the agent that sees the error and asks again creates a second card for the same thing; the first stays open.
7. **The stopped-twice question** (`runner/work.go`, after one nudge) is a `system` question with fixed options retry / complete / fail plus free text. A typed reply resumes the agent with `ownerReplyPrompt`. It is created by the daemon, blocks the daemon's own loop, and has the same lost-answer behaviour.
8. **Feature tasks** (`plans.QueueWork`) get a plan run first when they have no subtasks. The plan run's only exit is `propose_split` (ends the run, raises `plan_review`). A plan run can call `ask_question` (it is in `readTools`), but nothing groups several of those.

### 1.2 What the owner wants

An agent with N blocking questions raises them at once. Each shows in the Inbox as its own card with options and the recommended default pre-marked, under one task header. The owner answers them one by one, in any order, and the agent gets all answers as one batch. Approval of a spec and of a plan are explicit review items. Timeouts and late answers do not lose anything.

## 2. Options

### Option A: a batch tool, `ask_questions`

A new agent tool taking `questions: [{key, prompt, detail_md, options[{label,value,description}], recommended, allow_free_text}]`. The controller creates one `question_batch` row and one `question` row per entry (kind `agent`, new nullable `batch_id` and `key`, `recommended` flag on the option). The call blocks until the batch is complete or the owner chooses "answer later" (section 3.4).

- Pros: explicit and validated at call time (duplicate keys, empty options). Works for any task kind, no document needed. Small change to the existing blocking path.
- Cons: the agent must call a tool instead of writing the questions where it already writes them (the spec). The spec and the batch can disagree.

### Option B: a machine-readable block in the spec, parsed by the controller

The agent writes a fenced block (`wrangl-questions` YAML/JSON) in the spec or plan document; the controller parses it on `task_complete` or on a plan/spec submission and creates the cards.

- Pros: one source of truth; the spec reads well and the cards come from it.
- Cons: the controller must find the document (repo file in a worktree the controller does not see, or a `submit_deliverable` report) and parse it after the fact, so errors surface when the agent has already moved on. A bad block is silent unless every submit validates it. Couples question handling to a document convention and to `task_complete`, which today has no document.

### Option C: keep one question, make the owner's free text richer

Render a long free-text question with a per-line answer form, no schema.

- Pros: no data model change.
- Cons: no options, no recommended default, no per-item state, no partial answers. This is today's failure with nicer typography. Rejected.

### Option D (hybrid, chosen): A as the primitive, B as a validated convenience

The primitive is the batch (Option A). A spec document may *carry* the machine-readable block, but the agent submits it through the same tool (`ask_questions` accepts `from_block_md`, a string containing the block, parsed with the same validator). The controller never reads repo files. This keeps one code path and one validator; the block syntax is only a way to fill the arguments from the spec text the agent already has.

## 3. Decision

Adopt Option D.

### 3.1 Data model (migration 00037+)

- New table `question_batch(id, task_id, run_id, state, created_at, closed_at)`; `state` is `open | complete | partial_closed | cancelled`.
- `question` gains `batch_id` (nullable FK), `batch_key` (agent-chosen, unique within the batch, e.g. `Q3`), `recommended_value` (nullable). The `kind` CHECK is not widened; batch members are kind `agent`. This avoids a second `writable_schema` migration like 00036.
- Answers stay in the existing columns (`answer_option_id`, `answer_text`). A question answered by accepting the default is stored as that option, plus `answer_text` empty; whether it was the recommended one is derivable.

### 3.2 The tool

`ask_questions({questions[], block_md?, blocking: true|false})`. Validation: 1 to 12 questions; unique `key`; each with a prompt and either options or `allow_free_text`; `recommended` must be one of the option values. Returns `{batch_id, question_ids[]}` immediately if `blocking` is false, otherwise blocks (3.4). `ask_question` stays, unchanged, for the single-question case. `Allowed()` puts `ask_questions` where `ask_question` is (`readTools`); the owner's own agents (`user` kind) do not get it.

### 3.3 Inbox presentation

One group per batch, headed by the task (sqid, title, "Q answered 3 of 8"). Each member is a card with its options, the recommended one marked and preselected, a free-text box where allowed, and a skip-this-one state. Buttons on the header: **Accept all recommended** (answers every still-open member with its default, with confirmation), and **Send answers** when the batch is not auto-sent. The CLI gets `wrangl question answer <id>` as today plus `wrangl question batch <batch_id>` to list members. Each member answer is stored immediately (so nothing is lost on a refresh); the agent is only told when the batch closes (3.4).

### 3.4 Delivery as one batch, and what "closed" means

- A batch closes when every member is answered, or when the owner presses **Send answers** with some left (those get `skipped`: "no answer; use your judgment and say so"), or when the run ends (section 3.5).
- Delivery is one message `{batch_id, answers: [{key, value, text, source: owner|default|skipped}]}` and the tool result carries the same structure. Members are never delivered one at a time.
- Answers are stored first, delivery second, and delivery is idempotent. See 3.5.

### 3.5 Timeouts and late answers (fixes the lost-answer bug)

The root cause is that "answered" and "delivered" are one event. Decision: split them.

1. Add `delivered_at` to `question_batch` (and to single `agent`/`system` questions in the same migration). An answer is *recorded* when the owner clicks; it is *delivered* when the runner acknowledges it to a live waiter.
2. `waiters` removes its entry when the tool context ends (fixing the leak) and the daemon acks a delivery with a reply; the controller leaves `delivered_at` NULL on no ack.
3. **Run still alive, call gone** (MCP timeout, restart): the controller re-sends on the next runner heartbeat/enrol, and, if the agent is idle or the process was restarted, injects the answers as the first part of the next turn prompt ("The owner answered your earlier questions: ..."), the same mechanism as `ownerReplyPrompt`. The agent re-ask is not needed.
4. **Run ended** (including cancelled, failed, stopped): the batch becomes `partial_closed`, not `cancelled`. Open members stay open, because questions about a task outlive one run. `CancelRunQuestions` no longer includes batch members. Answering them later is allowed, and the answers are attached to the task: shown in its header and prepended to the prompt of the task's next run ("Answers from the owner to your earlier questions"). `Answer` therefore never returns `Conflict: cancelled` for a batch member.
5. **Owner never answers**: no automatic timeout and no auto-default (a silent default on a blocking question is a decision the owner did not make). The existing stuck-run watchdog still applies; for a run waiting on a batch it raises nothing new, since the batch cards are already in the Inbox. Optionally, a per-project setting `auto_accept_recommended_after` can be added later; it is out of scope here.
6. The same split fixes the single `ask_question` path. That is a bug fix in its own right and ships first (Stage 1).

### 3.6 Spec and plan approvals as explicit items

Add two explicit review requests, using existing kinds so no CHECK change is needed:

- `submit_spec` is not a new tool. A spec is a `submit_deliverable` (kind `report_md`) whose `purpose` is `spec`; when a spec deliverable is submitted, `deliverable_review` is raised with options **Approve spec / Request changes** (text), grouped under the same task header as the batch. The agent is told approval is pending and ends or continues.
- The plan: the existing `plan_review` (Accept / Edit / Reject) is the approval. A feature task's plan run may call `ask_questions` before `propose_split`, and the owner's answers and the Accept both arrive as items under one task header.
- Ordering rule: a spec or plan approval card is created *after* its questions are all closed, and shows the answers in its detail, so the owner does not approve a document whose open questions are still unresolved. An approval card never gates the batch, and a batch never gates an approval except through this ordering.

### 3.7 Interaction with the stopped-twice system question

It stays a single `system` question (it is the daemon's, not the agent's) and is not folded into a batch. Two changes: (a) it gets `delivered_at` and the same re-delivery (3.5), so a restart does not lose a "retry with this text" reply; (b) if the run has an open batch when the agent stops twice, the card text says so ("waiting on N unanswered questions") and the options are only offered after the batch is closed or skipped, so the agent is not nagged for stopping while it is legitimately blocked on the owner.

### 3.8 Interaction with feature-task planning

A plan run keeps `propose_split` as its only exit and `plan_review` as its approval. It may first run `ask_questions` (blocking). The proposal's detail lists the answers it was based on. A `Reject` with feedback re-queues the plan run as today; unanswered batch members from the first plan run stay open and are shown again, not re-asked, because they hang on the task (3.5.4).

## 4. Consequences

- The owner can answer Q1 to Q8 independently, accepting defaults in one click, and the agent gets them together. The hpCqdq failure mode has a direct path.
- Lost answers become impossible for batches and are fixed for single questions: record first, deliver at least once, idempotently. The cost is a re-delivery path and a new prompt-prefix injection, both to be tested with a killed runner and an expired MCP call.
- A cancelled or failed run no longer erases a question list. Questions on a task now outlive runs, so the Inbox can hold older groups; the header shows the run they were asked in, and the task's close/cancel path cancels them.
- Agents can still dump everything into one free-text question. `ask_questions` is advisory text in the tool description plus a prompt hint in `nudgePrompt`; no enforcement is attempted.
- Cost of the choice over B alone: the spec text and the submitted batch can differ. Mitigation: the `from_block_md` convenience, and the spec approval card showing the answers.
- No auto-defaults means a task can wait forever. That is a visible state (cards in the Inbox), not a silent one.

## 5. Stages and task breakdown

**Stage 1: stop losing answers (small, independent, ships first).**
1. Runner: drop the waiter on tool context end; ack deliveries. (`runner/agentsock.go`, `daemon.go`)
2. Migration 00037: `delivered_at` on `question`. Controller: set on ack; re-send unacked answers on runner reconnect. (`runs/questions.go`)
3. Late-answer tests: run ended, MCP call expired, runner restarted.

**Stage 2: batches.**
4. Migration 00038: `question_batch`, `question.batch_id/batch_key/recommended_value`.
5. `ask_questions` tool: schema, validator (shared with `from_block_md`), controller handler, `Allowed()` entry. (`stacktools`, `runs/tools.go`)
6. Batch close and delivery, `partial_closed` on run end, the exclusion from `CancelRunQuestions`, answers prepended to the next run's prompt.

**Stage 3: Inbox and CLI.**
7. Grouped cards, recommended preselected, Accept all recommended, Send answers, progress. (`web/inbox.go`, template)
8. `wrangl question batch`, `answer` for members. (`cli/question.go`)

**Stage 4: approvals and the system question.**
9. Spec deliverable purpose and its `deliverable_review` wording; ordering rule after the batch closes. (`deliverables`)
10. Plan-run guidance and `plan_review` detail listing the answers. (`plans`)
11. Stopped-twice card: waiting-on-batch text, deferred options, delivery ack. (`runner/work.go`)

**Stage 5: docs.** Update the guide's tasks/Inbox chapter and `agents.md` (tool list, nudge wording).

## 6. Open questions for the owner

1. Should "Accept all recommended" exist at all, or only per-card defaults? (Proposed: exists, with a confirmation.)
2. Should batch members outliving the run be the default (3.5.4), or should a failed run's questions be cancelled as today when the owner chose "Fail the run"? (Proposed: outlive, cancelled with the task.)
3. Limit of 12 questions per batch: right size?
4. Is a spec a `deliverable_review` with a purpose, or does it deserve its own kind? (Proposed: purpose on the existing kind, to avoid another CHECK migration.)
