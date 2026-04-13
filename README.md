# Temporal Developer Plugin for OpenAI Codex

This repository provides an [OpenAI Codex plugin](https://developers.openai.com/codex/plugins/build) for developing [Temporal](https://temporal.io/) applications.

> [!WARNING]
> This plugin is in Public Preview, and will continue to evolve and improve.
> We would love to hear your feedback - positive or negative - over in the [Community Slack](https://t.mp/slack), in the [#topic-ai channel](https://temporalio.slack.com/archives/C0818FQPYKY)

## Installation

### Repo-scoped (recommended)

Clone this repo into your project and Codex will discover the plugin via the marketplace file at `.agents/plugins/marketplace.json`:

```bash
git clone https://github.com/temporalio/codex-temporal-plugin.git
```

Restart Codex, open the plugin directory, and install **Temporal Developer** from the marketplace.

### Personal

To install for all projects, copy the plugin to `~/.codex/plugins/` and add an entry to `~/.agents/plugins/marketplace.json`.

## What's Included

- **temporal-developer** skill — Comprehensive guidance for developing Temporal applications: creating workflows, activities, and workers; handling signals, queries, and updates; debugging non-determinism errors; implementing saga patterns, versioning strategies, and testing approaches across Python, TypeScript, Go, and Java SDKs.

## Standalone Skill

The skill content is derived from [temporalio/skill-temporal-developer](https://github.com/temporalio/skill-temporal-developer). If you only need the skill without the plugin wrapper, you can install it directly from that repo.

## Other Coding Agents

- **Claude Code**: See [temporalio/claude-temporal-plugin](https://github.com/temporalio/claude-temporal-plugin)
- **Cursor**: See [temporalio/cursor-temporal-plugin](https://github.com/temporalio/cursor-temporal-plugin)

## License

MIT
