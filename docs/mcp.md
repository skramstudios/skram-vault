# The MCP server and agent files

`skram-vault mcp` is a stdio MCP server over the same vault the CLI uses. It
starts in the directory the agent was started in, works out which namespaces
cover that checkout from the `repos:` block, and tells the agent so in its
instructions, with up to 40 topic summaries per namespace. Every tool call
that writes is one git commit in the vault, attributed to the agent.

## Register it

**Claude Code**, once, for every project:

```bash
claude mcp add --scope user skram-vault -- skram-vault mcp
claude mcp list        # skram-vault should be listed as connected
```

**Any other MCP client** that launches stdio servers from a JSON file (Cursor's
`~/.cursor/mcp.json`, for one) takes the same definition:

```json
{
  "mcpServers": {
    "skram-vault": { "command": "skram-vault", "args": ["mcp"] }
  }
}
```

The client must be able to find `skram-vault` on its `PATH`; give the absolute
path (`~/.local/bin/skram-vault`, expanded) if it cannot. To use a config file
other than `~/.config/skram/config.yaml`, put `--config <file>` before `mcp`
in the arguments.

`skram-vault doctor` checks the Claude Code registration: missing prints the
line above, and one that launches something other than `skram-vault mcp` is
reported as an issue.

## Tools

| Tool | What it does |
| --- | --- |
| `lore_index` | topics for the namespaces covering the working directory |
| `lore_search` | search headings, summaries, and bodies; call it before debugging |
| `lore_read` | one topic with its recent history |
| `lore_write` | create a topic, or replace or append one section |
| `lore_decide` | append a dated decision and its rationale to the decision log |
| `spec_new`, `spec_list`, `spec_show`, `spec_set` | specs; present when `vault.specs` is set |
| `ticket_new`, `ticket_list`, `ticket_show`, `ticket_set`, `ticket_close` | tickets; present when `vault.tickets` is set. `ticket_list` with `frontier` is the work that can start now; `ticket_set` with `claim` takes it |

Resources: `skram://vault` (the root index), `skram://lore` and
`skram://lore/<ns>` (topic indexes), `skram://lore/<ns>/<topic>`,
`skram://vault/<ns>/specs`, `skram://vault/<ns>/tickets`.

## Tell agents the vault exists: `agent install-rules`

The server describes itself to a client that connects to it. The agent files
cover the rest: an agent reading the checkout's instructions learns there is
a vault, which namespaces apply, and that specs and tickets live there rather
than in GitHub Issues.

```bash
skram-vault agent install-rules          # every checkout a repos: entry binds to a namespace
skram-vault agent install-rules --here   # only the checkout you are in
skram-vault agent install-rules --check  # write nothing; exit 1 if anything is missing, stale, or edited
```

It writes machine-local files only:

- a `## Knowledge vault (skram-vault)` section in `AGENTS.local.md`;
- a one-line `CLAUDE.local.md` importing it (Claude Code does not read
  `AGENTS.local.md` by itself);
- `docs/agents/issue-tracker.md` and `docs/agents/triage-labels.md`, when the
  checkout has a namespace of its own and specs or tickets are on;
- those paths in `.git/info/exclude`, so `git status` stays clean and nothing
  reaches the repo's history. `.gitignore` is never touched.

**If the checkout relies on `AGENTS.md` and has no `CLAUDE.md`:** Claude Code's
documentation says a `CLAUDE.local.md` makes it stop reading `AGENTS.md`
there. Set Claude Code's *Project instructions* setting to
`claude-md-and-agents-md` to keep both.

## With Matt Pocock's skills

[mattpocock/skills](https://github.com/mattpocock/skills) read
`docs/agents/issue-tracker.md` to learn where "the issue tracker" is, and
`install-rules` writes that file. With it in place `/to-spec` publishes a spec
to the vault, `/to-tickets` publishes tickets with their blockers, and
`/implement` takes one from the frontier, claims it, and closes it with a
resolution. Nothing is added to the skills and nothing is committed to the
repo. If `/setup-matt-pocock-skills` asks which issue tracker you use, the
vault is a custom one: `install-rules` supplies the document that setup would
otherwise write, so run `install-rules` after it.
