# Branch policy

Write and commit ONLY to `main`. `main` is the only branch that flows into the
release, so features added anywhere else (e.g. `feat/hypergraph` or any other
feature branch) will not ship. Do not create or commit to feature branches for
our work. If a task seems to call for a different branch, stop and ask first.

(Superseded: an earlier instruction said to develop only on `feat/hypergraph`.
That was wrong — hypergraph is one of the upstream team's feature branches and
is not where our releasable work belongs.)

# Subagents

尽可能使用subagent team来完成复杂任务，且必须使用opus 4.6作为驱动所有subagents的模型
(Prefer subagent teams for complex tasks; drive all subagents with Opus.)
