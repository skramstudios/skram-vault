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
| `spec_new`, `spec_list`, `spec_show`, `spec_set` | specs; present when `vault.specs` is set. `spec_set` changes `status`, `superseded_by` (with `superseded_by_set`), `title`, `summary`, or one section |
| `ticket_new`, `ticket_list`, `ticket_show`, `ticket_set`, `ticket_close` | tickets; present when `vault.tickets` is set. `ticket_list` defaults to open tickets only (`include_closed` or an explicit `status` reaches a closed one); with `frontier` it is the work that can start now; `ticket_set` with `claim` takes it. `ticket_set` also changes `status`, `spec` (`spec_set` with an empty `spec` clears it; an id that is not a spec in the namespace is refused), `title`, `summary`, and `blocked_by` in one commit |

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

Claude Code keeps its own per-machine memory per checkout. Some of it is
durable — a rule, a gotcha, a pointer — and belongs in lore; some of it is a
handoff note for the next session and does not. Which is which is an
editorial call, made one memory at a time: the `metadata.type` label a
memory carries was picked by whichever agent wrote it, so it is a hint,
never a gate. `agent promote-memory` lists what is there and promotes
exactly the memories you name:

```bash
skram-vault agent promote-memory                              # list every covered checkout's memories; never writes
skram-vault agent promote-memory --here                       # list the checkout you are in
skram-vault agent promote-memory --here --json                # the same listing as one JSON document
skram-vault agent promote-memory --here some-trap some-rule   # dry run: what those two would promote
skram-vault agent promote-memory --here --yes some-trap       # promote it without asking
skram-vault agent promote-memory --here --yes --prune some-trap   # …and delete the memory file
```

**Listing.** Without names the command only reports, never writes, and
never prompts. Each checkout shows its target namespace, its memory
directory, and one row per memory file (every file but `MEMORY.md`, the
memory's own index): the name, the type label, `IN LORE` — the
`<namespace>/<id>` of a lore topic with exactly that id in any namespace,
or `-` — and the summary. Frontmatter parsing is tolerant: when strict YAML
fails (as it does on a memory whose unquoted `description:` contains
`": "`, which is how Claude Code sometimes writes one), a line-based
fallback still recovers the name, description and type. A file whose name
still cannot be recovered gets a row marked unreadable, with its error.
`--json` emits
`{checkouts:[{root, repo, namespace, memory_dir, memories:[{name, file,
type, summary, in_lore, similar, error}]}]}`, with `in_lore` and `error`
`null` when there is nothing to say, and `similar` omitted when there is
nothing similar.

A memory whose id is not an exact lore topic but is close to one is hinted
as similar: both ids are split on `-`, stop tokens (`the`, `a`, `an`,
`and`, `of`, `to`, `in`) are dropped, and they are similar when they share
at least 2 tokens and the shared tokens are at least half of the shorter
id's tokens. `IN LORE` then shows the best match as
`<namespace>/<id>?   (similar id)`; `--json` carries every match, best
first and up to 3, as `similar: ["<namespace>/<id>", ...]`. It is
informational only — naming a memory with a similar id still promotes it.

Selection matches `install-rules`, worktrees included: a linked worktree
resolves to its main checkout's memory, so the two are one visit.
`--namespace` overrides the target namespace (the first non-`shared` one the
checkout's `repos:` entry lists); `--memory-dir` overrides the memory lookup
for one checkout, so it needs `--here`.

**Promoting.** Names need `--here`: a name means something only inside one
memory directory. A name matches a memory's `name:` field or its file's
basename. A name that matches nothing, matches more than one file, or
matches a file that cannot be read, and two names that would promote to the
same topic id, fail the run before anything is written. `--yes` or
`--prune` without names fails too:

```
Error: name the memories to promote; run without --yes to list them
```

A named memory whose id is already a topic in any namespace is refused as
`already in lore as <namespace>/<id>`, never overwritten or copied; a name
that is not a valid flat topic id is refused as `invalid id`. Refusals do not
stop the other names, and the run exits non-zero only when every name was
refused.

A named run is a dry run unless `--yes` is given. Standing at a terminal
with no `--yes`, it asks; an empty answer or "y"/"yes" writes, anything else
does not. Piped stdin writes nothing, and a run that cannot write does not
sync the vault's remote either. What is promoted lands in one attributed
vault commit, as a Skeleton topic whose summary is the memory's description
and whose body is the memory's body plus a line naming the file it came
from. With `--json` the report carries `promoted`, `refused` (as
`{name, reason, existing}`), and the commit once written.

Memory files are left in place unless `--prune` is also passed: opt-in and
only meaningful with a write, it deletes — once the commit succeeds — every
memory file that commit promoted and that file's line in `MEMORY.md`,
leaving every other memory file and index line untouched. Without a write
(a dry run, or a declined or piped prompt), `--prune` deletes nothing and
the report lists what it would remove; a failed commit prunes nothing
either.

`agent import-memory`, the command's name in 0.6.0, still runs for one
release: it prints `import-memory is now promote-memory` and then behaves
exactly as `promote-memory`.

`skram-vault doctor` prints no memory line: whether anything is worth
promoting is an editorial call, and `promote-memory --here` is how you make
it, not something doctor nags about.

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
