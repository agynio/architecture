# Skills on the Agent Filesystem

## Target

- [Resource Definitions — Skill](../architecture/resource-definitions.md#skill)
- [agynd — Skills](../architecture/agynd-cli.md#skills)
- [Agents Service — Resources](../architecture/agents-service.md#resources)
- [Console — Agents](../product/console/console.md#agents)

## Delta

`agynd` fetches every skill at startup and writes each one as a single file named for the skill — `~/.claude/skills/<name>` — holding the body and nothing else. Claude Code and Codex discover a skill as a directory holding a `SKILL.md`; a plain file sitting in that directory is not a skill, and is passed over without complaint. Skills reach the agent's filesystem and no agent reads one.

`description` compounds it. The field is stored, served, editable in the console and settable from Terraform, and `agynd` never reads it off the wire — in a layout with no front matter there is nowhere for it to go. It is the one part of a skill a discovering CLI reads first, so a corrected layout alone would produce skills that nothing triggers.

`name` is unconstrained. The Agents Service validates MCP, volume and sandbox names against patterns and applies none to skills, so a skill may be stored as `Skill 4f2a` — a space and a capital — which cannot be a directory name any CLI will accept.

Only `agn` agents are unaffected, and only because they never use the files: their skills go into the system prompt. That is what has kept the gap looking narrow. An agent observably acting on its skills is an `agn` agent, taking them through a path the other two SDKs do not share.

## Acceptance Signal

- A skill on an agent running Claude Code or Codex appears in that CLI's own skill listing, under the stored name and description, and the agent acts on the body when the situation the description names arises.
- The stored `description` reaches the agent. A skill written in the console with a trigger description is triggered by it, with no other change to the agent's configuration.
- `CreateSkill` and `UpdateSkill` reject a name that is not a slug, as `CreateMcp` already rejects one that does not match its pattern.
- `agn` agents are unchanged: skills arrive in the system prompt in creation order, and no skills directory is written for them.
- A malformed skill is skipped, its reason is on the container's stderr, and the agent starts.
- The suites that exercise skills — the Agents Service e2e suite and the Terraform acceptance tests — create them under names the API accepts.

## Notes

The skills the platform models are the feature Claude Code and Codex implement, where a skill costs its description until the moment it is used. Folding every body into the system prompt is what `agn` does because it has no alternative; it is not the target behavior for the SDKs that have one.

The [open question](../open-questions.md) about a skills directory convention for `agn` is resolved by there not being one.
