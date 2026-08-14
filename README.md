# agents-claude

Custom subagent definitions for [Claude Code](https://claude.com/claude-code).

An agent definition is a markdown file with YAML frontmatter that fixes a subagent's **model**, **tool access**, and **standing instructions**. Claude Code discovers them and makes them available to the `Agent` tool by name. The value over putting the same text in a prompt is that the rules cannot be forgotten — they travel with the agent rather than depending on whoever dispatches it.

## Contents

```
agents/
├── LICENSE
├── README.md                      ← you are here
└── travel/                        Research agents for the travel-agent skill
    ├── README.md                  Full docs: what each agent does, install, usage
    ├── travel-researcher.md       Sonnet · fine-print tracks where errors cost money
    └── travel-scout.md            Haiku · retrieval and list-building tracks
```

## What's here

**[`travel/`](travel/)** — a matched pair of live-research agents. `travel-researcher` takes the tracks where fine print costs money or causes trouble (flights, lodging, entry and health, law, points, fare rules, food); `travel-scout` takes the retrieval-shaped ones (ground transport, experiences, community sentiment, language). Both carry a strict anti-fabrication protocol, a blocked-source ladder, an API-first source list, budget-triage rules, and a **gather-don't-compute boundary** — neither holds a credential or does arithmetic, so keys stay out of transcripts and derived figures always ship with their method.

> **These two agents are required by the [`travel-agent` skill](https://github.com/wrobelsec/agent-skills-claude).** The skill dispatches them by name and will not work without them installed. See [`travel/README.md`](travel/README.md).

## Install

Clone into your Claude Code agents directory:

```bash
git clone https://github.com/wrobelsec/agents-claude.git ~/.claude/agents
```

Or copy a single folder into an existing setup:

```bash
cp -r travel ~/.claude/agents/
```

Then confirm the agents are discovered — start Claude Code and check that `travel-researcher` and `travel-scout` appear in the available agent list. Discovery recurses into subdirectories, so the `travel/` folder is fine as-is.

## Related

- **[agent-skills-claude](https://github.com/wrobelsec/agent-skills-claude)** — skills that use these agents, including `travel-agent`.

## Licence

See [LICENSE](LICENSE).
