# Mad Fish MCP Skills

Workflow skills for the [Mad Fish MCP](https://www.madfishdigital.com) marketing-ops
server. Each skill teaches an LLM (Claude, and any client that supports Agent
Skills) to run real tasks against your connected ad and analytics platforms —
executing through the MCP's discovery meta-tools rather than just giving advice.

## Prerequisite

You need an active **Mad Fish MCP connection** (`madfish-adops`) with the
relevant platform licensed. Skills call the MCP's discovery meta-tools
(`discover_search_operations`, `discover_execute_operation`,
`discover_list_platforms`, `discover_help`, `discover_skills`). Without the MCP
connected, a skill can still give advice but can't act on a live account.

## Install (Claude Cowork / Claude Code)

Add this repo as a marketplace once, then install the packs you want:

```
/plugin marketplace add benherman/madfish-mcp-skills
/plugin install google-ads-skills@madfish-mcp-skills
```

Or with the skills CLI:

```
npx skills add https://github.com/benherman/madfish-mcp-skills --skill google-ads-manager
```

Not sure which skills apply to you? Ask your LLM **"what skills can I install?"** —
the MCP's `discover_skills` tool returns the packs relevant to your licensed
platforms and hands back the exact install command.

## Skills

| Skill | Platform | What it does |
|---|---|---|
| `google-ads-manager` | Google Ads | Account audits, keyword research, campaign build, RSA copy, bid/budget optimization, negative keywords, performance reporting. |

_More platform packs (GA4, GTM, Meta, LinkedIn, Search Console) coming._

## How these differ from generic marketing skills

A generic "Google Ads" skill gives best-practice advice. These skills carry the
same expertise **and** map every workflow to the operations your MCP actually
exposes, so the LLM can inspect and change a live account. They're also
**capability-aware** — if your seat is Read-only, the skill will surface the
upgrade path instead of failing on a write.

## Capability & safety model

- Read workflows (audits, reporting, keyword research) work on any licensed seat.
- Write/Delete workflows respect your subscription's capabilities; the MCP
  refuses ungranted operations and the skill surfaces the upgrade path.
- Every skill confirms before writes, builds campaigns PAUSED, and reads back
  after changes. Money-affecting actions always show dollar amounts first.

---

Made by [Mad Fish Digital](https://www.madfishdigital.com). Skills are plain
Markdown (`SKILL.md`) — browse `skills/` to read or fork any of them.
