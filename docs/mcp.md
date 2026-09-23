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

A checkout's linked worktrees (`git worktree add`, or a task-lane checkout
opened beside it) are covered the same way as the checkout itself, so a fresh
worktree gets the section too; a worktree whose directory has since been
removed is skipped without a note.

**If the checkout relies on `AGENTS.md` and has no `CLAUDE.md`:** Claude Code's
documentation says a `CLAUDE.local.md` makes it stop reading `AGENTS.md`
there. Set Claude Code's *Project instructions* setting to
`claude-md-and-agents-md` to keep both.

## From Claude Code's memory

Claude Code keeps its own per-machine memory per checkout (its durable
`feedback` and `reference` entries are exactly the kind of thing lore is
for). `agent import-memory` finds it, reports what it would promote, and —
on `--yes`, or a terminal "y" to the prompt — promotes it:

```bash
skram-vault agent import-memory                        # dry run: report only, write nothing
skram-vault agent import-memory --here --yes           # promote the checkout you are in without asking
skram-vault agent import-memory --here --yes --prune   # …and delete what it promoted
```

The default is a dry run: it prints exactly what it would promote and writes
nothing. Standing at a terminal with no `--yes`, it then asks; an empty
answer or "y"/"yes" writes, anything else does not. `--yes` writes without
asking, terminal or not. Without `--yes`, only a terminal prompts: piped
stdin writes nothing, and a run that cannot write does not sync the vault's
remote either.

Selection matches `install-rules`, worktrees included: a linked worktree
resolves to its main checkout's memory, so the two are one import, visited
once. The report lists, per checkout, what it would promote, what already
conflicts with a topic of the same id, and what it would leave behind (a
handoff note, an untyped entry, an invalid name, or an unreadable file whose
frontmatter does not parse, listed with its error; none of these fails the
run). Two memories that resolve to the same id in one run, from one
checkout or two, promote the first and report the rest as conflicts.
`--namespace` overrides the target namespace; `--memory-dir` overrides the
memory lookup for one checkout, so it needs `--here`. `--json` emits the same report as one document, carrying the commit(s) once
`--yes` (or an accepted prompt) has written.

Every promoted memory of one run lands in one attributed vault commit per
namespace the run touches — one commit for the whole run in the usual case,
every visited checkout sharing a namespace; a sweep that spans several
namespaces lands one commit per namespace instead. Run the same import on a
second machine after `skram-vault sync`, and the two machines' memories meet
in one namespace: a memory whose id is already a topic there is reported as
a conflict and never written or merged, so the second import surfaces
duplicates for you to merge by hand instead of silently overwriting them.

Memory files are left in place unless `--prune` is also passed: opt-in,
never implied, and only meaningful together with a write, it deletes — once
a namespace's commit succeeds — every memory file that commit promoted and
that file's line in the memory directory's own index, MEMORY.md, leaving
every other memory file and every other index line untouched. Without a
write (a dry run, or a declined or piped prompt), `--prune` deletes nothing
and the report lists what it would remove instead; a failed commit prunes
nothing either. The report and `--json` both gain a `pruned` list.

`skram-vault doctor` adds a `[--]` note, per checkout, when its memory holds
a typed `feedback` or `reference` entry not yet promoted to a topic in the
target namespace, ending with the `agent import-memory --here` line to run.
A checkout with no memory directory gets no note, and the note never changes
`doctor`'s exit code — it is a pointer, not a problem.

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
