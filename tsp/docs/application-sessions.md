# Worker session lifecycle

Before using TSP tools, obtain an application session for this individual worker.
Call `tsp_session_register` (canonical `tsp.session.register`) with
`payload={"name":"<short task name>"}`; use the name a Build prompt gives you,
otherwise follow the `name` field's description. Keep
its private `session_handle` and `resume_handle` in this worker's private runtime
state. Independent workers register separately. A Build seed carries a single-use
`claim`: pass it to register first. The response gives the attempt and brief
coordinates. A launched agent’s subagents reuse its handle; they never register
separately. A claim can be redeemed again only before the handle’s first use,
to recover a lost response; that closes the earlier handle. The session handle is
the attempt credential; caller-supplied attribution cannot substitute for it.

Include `session_handle` in the ordinary `payload` arguments of every subsequent
tool, including explicit-plan calls and `tsp_context_get` / `tsp_context_set` /
`tsp_context_clear`. Use the returned immutable `session_id` for public
attribution. Never expose either private handle in plan fields, notes, logs,
commits or public messages. Share a launched agent’s handle only with its own
subagents.

Renew using `tsp_session_renew` by the server-provided `renew_after` time. Before
resuming after interruption or an expired lease, call `tsp_session_resume` with
`payload={"resume_handle":"<private recovery handle>"}` and replace the active
handle with the returned one. The previous active handle is fenced. Recovery
credentials remain stable so a lost response can be retried; concurrent resumes
replace one another, so only the last active lease remains valid.

Use `tsp_context_set` to establish this worker's plan. Explicit selectors
override the default for one call without changing it. Agents work on the plan
itself. Never infer another worker's context from a shared OAuth token or
transport connection.

A Build seed carries a claim for registration. A non-null `attempt` in the
`tsp.session.register` response identifies a bound worker: omit `node_ids`
on every `tsp.attempt.progress`, including the first milestone. A hand-run
worker registered with `attempt=null` supplies all nodes it will work on in
`node_ids` on its first progress only, opening its attempt. Supplied `node_ids`
on a bound worker must equal the whole attempt scope as a set; a mismatch
is refused with the bound nodes named.
Record each node result with
`tsp.workflow.record_result`, which writes the attempt report and retained status
projection. At worker shutdown, finish those reports and handoff notes,
then call `tsp_session_close`. Do not close an application session merely because
one skill invocation finishes if the same worker will continue using TSP.
Transport disconnects do not close workers. An independent, explicitly closed worker requires a new registration. A launched
agent whose credential is closed or whose handles are lost must stop and tell the
person; never re-register. Continue issues a new claim and revokes the old agent. An invalid active handle calls for recovery;
failed OAuth authentication calls for the harness's OAuth login/refresh flow.
These are separate failures.

## Attempts and older plugins

Execution belongs to attempts. Separate agents can work on the same node without
waiting for a leaf owner. Continue gives the next agent a new epoch on the same
attempt; the old credential cannot write through it.

Older plugins may still call `record_start`, `record_progress`, and `record_result`.
These translate onto the caller's attempt without creating a second execution
record. Current agents use `tsp.attempt.progress` and `tsp.workflow.record_result`
with their session handle; no run id is needed. `agent_session_id` is accepted for
old callers but does not authorize a write. A supplied `work_ref` must match the
attempt's reference. No call creates a run record or claims a leaf.
