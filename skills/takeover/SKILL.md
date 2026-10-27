---
name: takeover
description: Take over existing work. Use when the user asks to take over a PR, branch, or agent thread, or requests a takeover.
license: MIT
metadata:
  upstream: t3dotgg/tstack
  upstream-revision: 7de017b3fc9d50c03cf3ecb98fb71e0ca65a78bd
---

# Takeover

Take responsibility for finishing the original goal while preserving the
original contributors' credit. Do not defend an implementation just because
another agent already built it.

Read the source PR, branch, or thread, including review findings and the user's
later corrections. Resolve the handoff to the actual code and current head,
and account for uncommitted work. Treat the handoff as context, not proof.

For a replacement PR, import the original commits onto a new branch and add
your fixes separately. Honor an explicitly requested in-place continuation, and a stacked continuation only when both the user and repository permit it. Do not overwrite unrelated work or change the original
PR without authorization.

Preserve human Git authorship and verified human co-author trailers. Follow the user's attribution policy; add no model credit unless requested. Credit the original
contributors in the PR description and link the original PR when there is one.
If the work will be squashed, preserve human author/co-author credit in the
merge commit too, using verified identities rather than guessed emails. Model
attribution is not a substitute for contributor credit.

Use [modernize](../modernize/SKILL.md) to check the continuation against its intended base and current architecture before deciding to rebase or rework it. Rework what is necessary to finish the same goal, not
unrelated features. Simplify the implementation without dropping intended
behavior or surfaces. Recheck the source PR too: it may have improved while
you were working.

Independently verify the original failure and realistic workflows, following
repository guidance and permissions. Green mocked tests do not prove the user
experience works. State what remains unverified.

When the requested finish includes a PR, use `file-pr` to file or update it,
then monitor only when the requested finishing contract includes monitoring. Use those skills when installed, otherwise follow the repository's workflow directly. Post replies, merge or close the original PR only when authorized. In T3 Code, link a created, taken-over or pushed PR immediately; only the owning parent watches it.

A takeover does not create permission for other branches, external messages, deployment or nested delegation. Stop before crossing an ownership collision.

Report what you inherited, what you changed, how you verified it, how credit
was preserved, and any remaining decision.
