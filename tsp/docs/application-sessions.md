# Worker session lifecycle

Before using TSP tools, obtain an application session for this individual worker.
Call `tsp_session_register` (canonical `tsp.session.register`) with
`payload={"name":"<short worker name>"}`. Collaborators see this name on
the canvas beside the node you work on, so name the task in a few words
(`Roster`, `Billing fix`), not the harness or project (`Claude Code`,
`TSP Project Roster`), so parallel workers stay distinguishable. Keep
its private `session_handle` and `resume_handle` in this worker's private runtime
state. Each parallel worker registers separately even when OAuth credentials are
shared. A workflow `agent_session_id` from `tsp.workflow.record_start` is a
different concept and cannot substitute for either handle.

Include `session_handle` in the ordinary `payload` arguments of every subsequent
tool, including explicit-plan calls and `tsp_context_get` / `tsp_context_set` /
`tsp_context_clear`. Use the returned immutable `session_id` for public
attribution. Never expose either private handle in plan fields, notes, logs,
commits or messages to other workers.

Renew using `tsp_session_renew` by the server-provided `renew_after` time. Before
resuming after interruption or an expired lease, call `tsp_session_resume` with
`payload={"resume_handle":"<private recovery handle>"}` and replace the active
handle with the returned one. The previous active handle is fenced. Recovery
credentials remain stable so a lost response can be retried; concurrent resumes
replace one another, so only the last active lease remains valid.

Use `tsp_context_set` to establish this worker's plan and branch — `main`, or a
branch found with `tsp_branches_list` or forked with `tsp_branch_create` when the
work should stay off the trunk. Explicit selectors override the default for one
call without changing it, field by field: restating `plan_id` keeps the selected
branch, and only an explicit `branch_name` (including `"main"`) changes it.
A branch the plan does not have fails with `BRANCH_NOT_FOUND` rather than
answering from `main`; recover with `tsp_branches_list`. Never infer another
worker's context from a shared OAuth token or transport connection, and never
treat branch work as merged — a human merges it in the app.

At worker shutdown, finish required workflow `record_result` and handoff notes,
then call `tsp_session_close`. Do not close an application session merely because
one skill invocation finishes if the same worker will continue using TSP.
Transport disconnects do not close workers. An explicitly closed worker requires
a new registration to work again. An invalid active handle calls for recovery;
failed OAuth authentication calls for the harness's OAuth login/refresh flow.
These are separate failures.

## Branch selection

Every TSP plan has `main` plus any number of named branches. Work that should
not touch the trunk goes on a branch, and the branch a worker selects is
inherited by every later call — reads answer from it and writes land on it.

**Find one.** `tsp_branches_list` (canonical `tsp.branches.list`) returns the
plan's branches, main first, each with node/edge counts and a `selected` flag
marking the one this worker is on.

**Fork one.** `tsp_branch_create` takes
`payload={"new_branch_name":"<name>"}` and forks from the branch the call
targets — the one already selected, or `main`. Names are 1-100 characters of
lowercase letters, digits, `.`, `_` or `-`, starting and ending with a letter
or digit, with no `..` and no `.lock` suffix. The result's `selector` addresses
the new branch.

**Select one.** Pass that `plan_id` and `branch_name` to `tsp_context_set`, then
omit both fields on later calls. Restating `plan_id` alone keeps the selected
branch; only an explicit `branch_name` changes it, and `branch_name="main"` is
how a worker deliberately returns to the trunk.

**When it goes wrong.** A branch the plan does not have answers
`BRANCH_NOT_FOUND` — it is never quietly answered from `main`, so a stale or
mistyped branch cannot read the trunk and then write to it. Call
`tsp_branches_list` to see what exists; no branch flagged `selected` means the
selection went stale and `tsp_context_set` needs a live one. A branch a live
generation session owns refuses structure writes with `PLAN_FORBIDDEN` and
`reason: generation_session_active`: act on that branch through
`tsp_review_submit`, or target another branch.

**Merging is the human's call.** No TSP tool merges, publishes or promotes a
branch. Branch work reaches `main` only when a person merges it in the app.
Report finished branch work by branch name and stop there — never describe it as
landed in the plan, and never work around the boundary by copying a branch's
content onto `main`.
