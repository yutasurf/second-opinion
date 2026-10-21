# second-opinion

A Claude Code skill that independently verifies the last factual claim Claude made.

Two sources, two jobs. Exa pulls public web evidence. GPT (via the Flovia MCP) judges the claim strictly against that evidence, with outside knowledge forbidden. Claude's own statement is the thing under test, so Claude does not vote.

Verdicts: CONFIRMED, PARTLY CONFIRMED, CONTRADICTED, NOT VERIFIABLE, each with cited sources.

## Requirements

- Claude Code
- The [Flovia](https://flovia.ai) MCP server connected, exposing `flovia_exa` and `flovia_openai`

## Install

```
git clone https://github.com/yutasurf/second-opinion ~/.claude/skills/second-opinion
```

Restart Claude Code. The skill triggers on "2nd opinion", "second opinion", "verify that", "double check that", "are you sure", "fact check that", or `/second-opinion`.

## What it will not verify

Opinions, drafts, and anything from private sources (internal databases, Slack, Notion, Telegram). Exa cannot see those, so the skill says so and points to the right internal check instead.

## Cost

Each Exa call is about 1% of the Flovia 5-hour window, each GPT call under 1%. Three claims is roughly 6%.

See `SKILL.md` for the full procedure and prompt template.
