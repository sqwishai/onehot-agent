# OneHot for coding agents

OneHot answers a question with a fixed set of outcomes and gives a probability for each one.
This plugin teaches a coding agent to build with it, and connects the agent to OneHot's MCP
tools, so it can lint a question, quote it, run it and train a model on your own rows.

It holds two things:

- the `onehot` skill: how to write decisions, read the probabilities, tune the wording, train
  a family of models and handle errors without paying twice;
- the OneHot MCP server, `https://api.sqwish.ai/mcp`, with tools such as `decide`,
  `estimate_decide`, `lint_decision`, `list_recipes` and `start_fine_tune`.

## Before you install

Make an API key in the [console](https://console.sqwish.ai) and set it where your agent runs:

```sh
export SQWISH_API_KEY=...
```

The MCP tools use your account. `decide`, fine-tunes and keep warm spend credit, and switches,
rollbacks and deletes change what answers your calls, so approve those calls in your agent.

## Install in Claude Code

```sh
claude plugin install onehot --marketplace sqwishai/onehot-agent
```

That needs Claude Code 2.1.292 or later. With an older version, add the marketplace first:

```sh
claude plugin marketplace add sqwishai/onehot-agent
claude plugin install onehot@sqwish
```

Restart Claude Code, then type `/mcp` and check that `onehot` is connected. Ask the agent to
list OneHot's models to try it. Mention OneHot in a prompt, or run `/onehot:onehot`, to use
the skill.

To offer it to everyone who works in a repository, add this to the repository's
`.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "sqwish": { "source": { "source": "github", "repo": "sqwishai/onehot-agent" } }
  },
  "enabledPlugins": { "onehot@sqwish": true }
}
```

## Install in Codex, Cursor and other agents

```sh
npx skills add sqwishai/onehot-agent --skill onehot
```

Add `-g` to install it for every project. That installs the skill only. Connect the MCP
server with the steps for your agent on the [Agents page](https://docs.sqwish.ai/agent-skills).

## Updates

Claude Code updates a plugin from a third-party marketplace only when you ask:

```sh
claude plugin update onehot@sqwish
```

or turn on auto-update in `/plugin`, under Marketplaces. With the `skills` installer, run
`npx skills update onehot`.

## Another server

`SQWISH_BASE_URL` points the MCP server somewhere else, such as a local preview at
`http://127.0.0.1:8080`. Left unset, it is `https://api.sqwish.ai`.

## Help

- Docs: [docs.sqwish.ai](https://docs.sqwish.ai)
- The API's contract: [openapi.json](https://api.sqwish.ai/openapi.json) and the short guide,
  [llms.txt](https://api.sqwish.ai/llms.txt)
- Support: help@sqwish.ai. Security reports: security@sqwish.ai

Apache 2.0. See [LICENSE](LICENSE).
