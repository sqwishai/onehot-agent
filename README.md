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

## Or ask your agent to install it

Paste this into your agent:

```text
Install the OneHot skill from https://github.com/sqwishai/onehot-agent. In Claude Code, run
`claude plugin install onehot --marketplace sqwishai/onehot-agent`. In any other agent, run
`npx skills add sqwishai/onehot-agent --skill onehot`, then connect the MCP server at
https://api.sqwish.ai/mcp with the header `Authorization: Bearer ${SQWISH_API_KEY}`, as
https://docs.sqwish.ai/agent-skills shows for this agent. Use only one of the two. If neither
works here, read https://github.com/sqwishai/onehot-agent/blob/main/skills/onehot/SKILL.md
and follow it.
```

Install it one way only, so your agent doesn't load two copies of the skill. If you added
OneHot's MCP server by hand before, remove it (`claude mcp remove onehot`), or the agent sees
every tool twice.

## Try it

Say "using the OneHot skill" in a prompt so any agent loads it. In Claude Code you can also
run `/onehot:onehot`.

- **Find fragile code.** "Using the OneHot skill, find where this project classifies text with
  string matching, regular expressions or by parsing a language model's reply, and propose
  OneHot decisions to replace them."
- **Build a feature.** "Using the OneHot skill, route incoming support tickets to billing,
  technical or other, and send a ticket to a person when no team gets at least 0.6."
- **Tune the wording.** "Using the OneHot skill, lint my decisions, then try them on the
  labelled examples in `tickets.jsonl` and tell me which wording and threshold work best."
- **Train your own model.** "Using the OneHot skill, quote a fine-tune on `tickets.jsonl`,
  start it if it costs under $5, and report the gate as it is."

## Common problems

- **The agent doesn't use the skill.** Run `/onehot:onehot`, or say "using the OneHot skill".
  Check that `/plugin` lists `onehot` as enabled, then restart the agent.
- **`/mcp` says `onehot` failed.** Start the agent from a shell where `SQWISH_API_KEY` is set.
  `invalid_api_key` means the key is wrong or revoked: make a new one in the console.
- **The agent invents request or response fields.** The skill is out of date. Update it, as
  below, and try again.
- **Answers don't match what you expect.** Describe each outcome more precisely, give a way
  out such as "other", and pick thresholds on examples whose answers you know. The linter
  flags the common mistakes.

## Keep it up to date

The skill follows OneHot's API. An old copy can lead your agent to send fields the API no
longer takes, or miss new ones. Claude Code leaves auto-update off for marketplaces that
Anthropic doesn't run, so turn it on once:

1. Type `/plugin` in Claude Code.
2. Open **Marketplaces** and choose **sqwish**.
3. Choose **Enable auto-update**.

Claude Code then checks for a new version during each session, installs it, and tells you
to run `/reload-plugins` to use it. To update by hand instead:

```sh
claude plugin update onehot@sqwish
```

With the `skills` installer, run `npx skills update onehot`.

## Another server

`SQWISH_BASE_URL` points the MCP server somewhere else, such as a local preview at
`http://127.0.0.1:8080`. Left unset, it is `https://api.sqwish.ai`.

## Help

- Docs: [docs.sqwish.ai](https://docs.sqwish.ai)
- The API's contract: [openapi.json](https://api.sqwish.ai/openapi.json) and the short guide,
  [llms.txt](https://api.sqwish.ai/llms.txt)
- Support: help@sqwish.ai. Security reports: security@sqwish.ai

Apache 2.0. See [LICENSE](LICENSE).
