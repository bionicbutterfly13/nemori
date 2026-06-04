# Branch policy

Write and commit ONLY to `main`. `main` is the only branch that flows into the
release, so features added anywhere else (e.g. `feat/hypergraph` or any other
feature branch) will not ship. Do not create or commit to feature branches for
our work. If a task seems to call for a different branch, stop and ask first.

(Superseded: an earlier instruction said to develop only on `feat/hypergraph`.
That was wrong — hypergraph is one of the upstream team's feature branches and
is not where our releasable work belongs.)

# Subagents

Use subagents for complex tasks.

Choose the appropriate agent for each task:
- Use specialized agents when their domain matches the work.
- Use research/explorer agents for read-only codebase investigation and upstream comparison.
- Use custom agents when the task needs a focused role that the default agents do not cover.
- Choose the smallest capable model for each subagent based on task difficulty, cost, and risk; do not default all subagents to premium models.
