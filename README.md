# Maxwell's skills

Four focused workflows for coding agents, UI critique and conversation practice. Each skill is a plain `SKILL.md` file, suitable for Claude Code or Codex.

| Skill | Use it for | Boundary |
| --- | --- | --- |
| `modernize` | Decide whether an old PR should be updated, reworked or retired before rebasing it | A relevance question stays read-only; a recommendation to close is not permission to close |
| `takeover` | Continue inherited work while preserving intent, contributors and fresh evidence | A handoff is context, not verification or permission for unrelated changes |
| `rehearse` | Practise a meeting with one question and concise feedback at a time | No invented project facts, private thoughts or commitments to the real counterpart |
| `ui-benchmark-review` | Shape a UI review brief, choose useful references, compare screenshots and prioritize visual fixes | Read-only critique; scores need visible evidence and do not establish runtime behavior or human approval |

## UI benchmark review

Start with your UI screenshots and a request such as:

> Use ui-benchmark-review to help choose references, compare this UI, and prioritize improvements to clarity and polish.

The skill asks a short Q&A about the task, desired direction and strengths to preserve when that context is missing. It can suggest two to four references, explaining which screen or pattern each helps evaluate. You can choose, replace or skip those references, or ask the agent to pick.

The review returns a seven-category visual scorecard, concrete strengths, and prioritized MUST / SHOULD / COULD fixes. Follow-up rounds retain the agreed brief and use fresh screenshots. Reference research requires available research tools; screenshot scoring requires image viewing. Delegated reviewers are optional and require an explicit request.

## Install selectively

Read a skill before installing it. Copy only the desired folder into your agent's skill directory. For example, from a clone of this repository:

```sh
mkdir -p "$HOME/.codex/skills"
skill_target="$HOME/.codex/skills/modernize"
test ! -e "$skill_target" && test ! -L "$skill_target" && cp -R skills/modernize "$skill_target"
```

For Claude Code, use `~/.claude/skills` instead. Do not overwrite an existing skill or symlink. Reload the skill catalog or start a new session after installation.

`takeover` links to `modernize`, so install both together. `file-pr` and `babysit-pr` are optional existing helpers; repository instructions can supply those procedures directly. No particular model, provider, subscription or local executable path is assumed.

## Authority and evidence

The current request and repository instructions control permitted actions. Reading a skill does not authorize a push, message, merge or deployment. Keep source identity, local checks, runtime behavior and human approval distinct.

These are curated release copies from the maintainer's operating collection. `PROVENANCE.json` records exact skill hashes and upstream sources. The maintainer updates the canonical source and exports the selected release files, rather than installing a second copy alongside existing skills.

## Credit and license

`modernize` and `takeover` adapt Theo Browne's [tstack](https://github.com/t3dotgg/tstack), pinned in their metadata and the provenance manifest. `rehearse` is a portable rewrite of a local conversation-practice workflow. See [third-party notices](THIRD_PARTY_NOTICES.md) and [MIT license](LICENSE).
