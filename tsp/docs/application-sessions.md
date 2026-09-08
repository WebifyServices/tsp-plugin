# Worker session lifecycle

Before using TSP tools, obtain an application session for this individual worker.
Call `tsp.session.register` with `payload={"name":"<short worker name>"}`. Keep
its private `session_handle` and `resume_handle` in this worker's private runtime
state. Each parallel worker registers separately even when OAuth credentials are
shared. A workflow `session_id` from `tsp.workflow.record_start` is a different
concept and cannot substitute for either handle.

Include `session_handle` in the ordinary `payload` arguments of every subsequent
tool, including explicit-plan calls and `tsp.context.get/set/clear`. Use the
returned immutable `session_id` for public attribution. Never expose either
private handle in plan fields, notes, logs, commits or messages to other workers.

Renew using `tsp.session.renew` by the server-provided `renew_after` time. Before
resuming after interruption or an expired lease, call `tsp.session.resume` with
`payload={"resume_handle":"<private recovery handle>"}` and replace the active
handle with the returned one. The previous active handle is fenced. Recovery
credentials remain stable so a lost response can be retried; concurrent resumes
replace one another, so only the last active lease remains valid.

Use `tsp.context.set` to establish this worker's plan and main branch. Explicit
selectors override the default for one call without changing it. Never infer
another worker's context from a shared OAuth token or transport connection.

At worker shutdown, finish required workflow `record_result` and handoff notes,
then call `tsp.session.close`. Do not close an application session merely because
one skill invocation finishes if the same worker will continue using TSP.
Transport disconnects do not close workers. An explicitly closed worker requires
a new registration to work again. An invalid active handle calls for recovery;
failed OAuth authentication calls for the harness's OAuth login/refresh flow.
These are separate failures.
