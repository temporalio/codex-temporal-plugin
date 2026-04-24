# Temporal Plugin for OpenAI Codex

This repository provides an [OpenAI Codex plugin](https://developers.openai.com/codex/plugins/build) for working with [Temporal](https://temporal.io/) — developing applications, using the CLI, managing server, and working with Temporal Cloud.

> [!WARNING]
> This plugin is in Public Preview, and will continue to evolve and improve.
> We would love to hear your feedback - positive or negative - over in the [Community Slack](https://t.mp/slack), in the [#topic-ai channel](https://temporalio.slack.com/archives/C0818FQPYKY)

## Installation

### Install from the Codex marketplace (recommended)

The Temporal plugin is published in the Codex plugin marketplace.

**Codex app:**

1. Open the plugins menu.
2. Search for **temporal**.
3. Click **+** (or **Add to Codex**) next to the plugin.

**Codex CLI:**

1. Run `/plugin`.
2. Search for **temporal**.
3. Mark it for installation.

Restart Codex after installing.

### Repo-scoped

Install the plugin locally through this repo:

1. Clone this repo into your project root:

   ```bash
   cd your-project
   git clone https://github.com/temporalio/codex-temporal-plugin.git
   ```

2. Copy the marketplace and plugin files into your project:

   ```bash
   # Create the directories Codex looks for at the repo root
   mkdir -p .agents/plugins plugins

   # Copy the plugin
   cp -r codex-temporal-plugin/plugins/temporal-developer plugins/

   # Copy the marketplace catalog
   cp codex-temporal-plugin/.agents/plugins/marketplace.json .agents/plugins/
   ```

   > If you already have a `.agents/plugins/marketplace.json`, merge the `temporal-developer` entry from the one in this repo into your existing file's `plugins` array.

3. Optionally, remove the cloned repo:

   ```bash
   rm -rf codex-temporal-plugin
   ```

4. Restart Codex and open the plugin directory. In the marketplace dropdown, switch from **OpenAI** to **Temporal**, then click the **+** button to install the plugin.


## What's Included

- **temporal-developer** skill — Comprehensive guidance for developing Temporal applications: creating workflows, activities, and workers; handling signals, queries, and updates; debugging non-determinism errors; implementing saga patterns, versioning strategies, and testing approaches across Python, TypeScript, Go, and Java SDKs.

## Standalone Skill

The skill content is derived from [temporalio/skill-temporal-developer](https://github.com/temporalio/skill-temporal-developer). If you only need the skill without the plugin wrapper, you can install it directly from that repo.

## Other Coding Agents

- **Claude Code**: See [temporalio/claude-temporal-plugin](https://github.com/temporalio/claude-temporal-plugin)
- **Cursor**: See [temporalio/cursor-temporal-plugin](https://github.com/temporalio/cursor-temporal-plugin)

## License

MIT
