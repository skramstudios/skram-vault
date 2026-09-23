# Skram Vault

A git-backed folder of markdown you and your agents share: lore (what you
learned the hard way), specs, and tickets.

One command, `skram-vault`, reads and writes it, and one MCP server hands the
same tools to Claude Code. Every write is a git commit, so the vault is
history you can review, sync, and roll back.

You need only this binary. It stands alone: no other Skram product has to be
installed, and nothing here assumes you have seen one.

```
my-vault/                  a git repo
  vault.md                 generated root index
  lore.md                  generated lore index
  shared/                  namespace every checkout can read
  my-app/                  one namespace per project or area
    lore.md                generated index for this namespace
    some-topic.md          a topic: frontmatter + markdown body
    specs/  tickets/       specs and tickets for this namespace
```

## Why not Beads, why not memory

Beads is the better issue tracker for agents on the axes a stranger compares
first: docs, an id scheme, a merge story. A team that already has an issue tracker should keep it. Skram Vault's claim
is the combination: one namespace holds what was learned (lore), what was
decided (specs), and what is left (tickets), outside every repo it serves,
the same on every machine, readable by any agent through one MCP server.

It is not your editor's built-in memory either. Claude Code's auto-memory is
per machine and per repo; a vault is shared across both, and it lives in a
repo you own, not inside a tool's own storage.

## 1. Install

Releases are built for macOS and Linux, on amd64 and arm64.

```bash
curl -fsSL https://raw.githubusercontent.com/skramstudios/skram-vault/main/install.sh | sh
skram-vault version
```

The script downloads the latest release for your machine, verifies it against
the release's `checksums.txt`, and puts `skram-vault` in `~/.local/bin`, which
must be on your `PATH`. `VERSION=v0.2.1` picks a release and `BIN_DIR=…` a
different directory. To do it by hand, take the archive for your platform
from the [releases page](https://github.com/skramstudios/skram-vault/releases),
check it against `checksums.txt`, and unpack `skram-vault` from it.

If you downloaded the archive in a browser, macOS quarantines the binary and
refuses to run it. Clear the flag once:

```bash
xattr -d com.apple.quarantine ~/.local/bin/skram-vault
```

## 2. Create a vault: `skram-vault init`

```bash
skram-vault init ~/vault --namespace my-app --dry-run   # show exactly what would change
skram-vault init ~/vault --namespace my-app --yes       # do it
```

`init` creates the directory, runs `git init`, creates `shared/` and the
`my-app/` namespace, commits the generated indexes, and writes a `vault:`
block into `~/.config/skram/config.yaml` (creating the file if you have none)
with specs and tickets turned on. Standing in a git checkout, it also offers a
`repos:` entry for that checkout.

It is additive only. It writes on `--yes` or when you answer `y` at a terminal,
never from a pipe; `--dry-run` prints the YAML diff and writes nothing. Your
comments and key order in an existing config survive, and the result is parsed
before the file is replaced. Run it twice and the second run says there is
nothing to do.

Until a config has a `vault:` block, every command that touches the vault
refuses to run, exits non-zero, and points you back here.

## 3. Bind your checkouts: the `repos:` block

The `repos:` block says which namespaces apply inside which checkout. If you
already have one to copy, paste it under the top level of
`~/.config/skram/config.yaml`; otherwise write one:

```yaml
repos:
  my-app:
    remote: my-org/my-app            # owner/name or any git URL; every checkout with this origin matches
    paths: [~/Dev/my-app]            # checkouts on this machine (~ and globs); the fallback when there is no origin
    namespaces: [my-app]             # vault namespaces in scope here; `shared` is always in scope
vault:
  path: ~/vault                      # the git repo from step 2
  specs: {}                          # turn on `skram-vault spec` and the spec_* tools
  tickets: {}                        # turn on `skram-vault ticket` and the ticket_* tools
```

Lore is always on. Specs and tickets are each on only when their key is
present; `init` writes both, and a `vault:` block you wrote by hand needs them
added.

From inside `~/Dev/my-app`, `skram-vault index` and `skram-vault lore search`
cover `my-app` and `shared`. Elsewhere they cover every namespace.

The file is read for exactly two top-level keys, `repos:` and `vault:`. Every
other key is ignored, so a config file shared with other tools, or one holding
nothing but these two blocks, both work. A malformed `repos:` or `vault:` block
is a hard error rather than a silent skip.

## 4. Register the MCP server

```bash
claude mcp add --scope user skram-vault -- skram-vault mcp
```

That registers it once, for every project, at Claude Code's user scope. The
server offers `lore_index`, `lore_search`, `lore_read`, `lore_write`,
`lore_decide`, and (when `specs:` and `tickets:` are set) the `spec_*` and
`ticket_*` tools, and tells the agent which namespaces cover the directory it
was started in.

### Tell agents about the vault in each checkout

```bash
skram-vault agent install-rules
```

For every checkout a `repos:` entry binds to a namespace, this writes
machine-local files only, and never anything you would commit:

- a `## Knowledge vault (skram-vault)` section in `AGENTS.local.md` naming the
  namespaces and the `skram-vault` commands and MCP tools to use;
- `docs/agents/issue-tracker.md` and `triage-labels.md`, when the checkout is
  covered by a namespace other than `shared` and the vault has `specs:` or
  `tickets:` configured (agent skills look for them at exactly that path);
- the one-line `@AGENTS.local.md` import in `CLAUDE.local.md`, so Claude Code
  loads it;
- their lines in the repo's `.git/info/exclude`, so none of it shows in
  `git status`.

A checkout's linked worktrees are covered the same way as the checkout
itself, so a fresh `git worktree add` (or a task-lane checkout opened beside
it) gets the section on the next `install-rules`; one whose directory has
since been removed is skipped without a note.

`--repos a,b` narrows to those entries and `--here` to the checkout you are in.
`--check` writes nothing, reports anything missing, stale, or edited, and exits
1. A checkout whose entry names no namespace is not visited: nothing is written
there and nothing is removed.

`AGENTS.local.md` may be shared with another tool that keeps its own section in
it. `skram-vault` only ever reads or replaces the section under its own
heading, and `--check` ignores every other section, so the two can run in
either order and end at the same file.

### Check the install

```bash
skram-vault doctor
```

Prints `[ok]` / `[--]` / `[!!]` lines for the vault (path, namespaces, spec and
ticket counts, and every lint problem, the root index included), the `repos:`
entries that name namespaces (a namespace the vault does not have is an issue;
a remote no checkout on this machine matches, and a path spelled in a different
case than on disk, are notes), whether the MCP server is registered at user
scope (with the exact `claude mcp add` line when it is not, an issue when the
registration launches something other than `skram-vault mcp`), and whether each
covered checkout's agent files are current. It exits 1 when any `[!!]` line is
printed. Missing agent files are a note; a stale one is an issue. Both come
from the same plan as `agent install-rules --check`.

## 5. Adopt a vault you already have

Point the config at it. `init` on an existing path adopts it and never modifies
what is inside:

```bash
skram-vault init ~/existing-vault --dry-run
```

An existing `.git` and `shared/` are kept, an existing `vault:` block is
printed and left alone, and a `repos:` entry you wrote is never edited; if it
lacks the `namespaces:` line, `init` prints the line to add.

**A vault built by hand.** If each namespace directory has an `index.md`
whose first line is the `<!-- generated` marker, and there is no `lore.md`,
tell the vault to keep using that file name rather than starting a second
index beside it:

```yaml
vault:
  path: ~/existing-vault
  lore:
    index_name: index.md
```

`init` proposes this line when it finds that shape. It never regenerates
anything for you. Before your first write, read the vault's own report:

```bash
skram-vault lint          # missing summaries, bad statuses, broken links, stale indexes
```

then run `skram-vault regen` when you are happy with it and look at the `git
diff` it leaves. (`index_name: vault.md` is rejected: that name belongs to the
root index.)

Topics are any `.md` file with a frontmatter block:

```markdown
---
title: Webhook retries
summary: The gateway retries three times, then drops silently.
status: Confirmed
related: [gateway-timeouts]
updated: 2026-01-01
---
Body in plain markdown.
```

`title`, `summary`, `status`, `related`, and `updated` are the ones the vault
reads; any other keys you add pass through untouched.

## 6. The commands

| Command | What it does |
| --- | --- |
| `skram-vault index [ns]` | list topics in a namespace, or every namespace covering the current directory |
| `skram-vault lore search <query>` | search topics; headings outrank summaries outrank body text |
| `skram-vault lore read <ns> <topic>` | a topic and its recent history |
| `skram-vault lore write [ns] <topic>` | create or update a topic or one `--section` (body from `--file` or stdin) |
| `skram-vault lore decide [ns] <title>` | append a dated decision to the namespace's decision log |
| `skram-vault lore import <dir> --namespace ns` | migrate a `knowns.md`-style wiki into a namespace |
| `skram-vault spec new\|list\|show\|set` | specs: `draft`, `ready`, `implemented`, `superseded` |
| `skram-vault ticket new\|list\|show\|set\|close` | tickets; `list --frontier` shows open, unclaimed, unblocked work; `set --claim` claims one |
| `skram-vault lint` | check every topic, spec, and index; exit 1 on problems |
| `skram-vault regen` | rewrite the generated indexes |
| `skram-vault sync` | `git pull --rebase` then push the vault's remote |
| `skram-vault mcp` | the MCP server |
| `skram-vault doctor` | report the vault, its `repos:` bindings, the MCP registration, and each checkout's agent files; exit 1 on any `[!!]` |
| `skram-vault agent install-rules` | write the vault's machine-local agent files into each covered checkout; `--check` verifies them |

Every read command takes `--json`. Every write is one commit in the vault, so
`git log` in the vault directory is the audit trail. Generated indexes carry a
`<!-- generated by skram … -->` marker on their first line and are rewritten by
every write: edit topic frontmatter, never the index.

`sync` needs the vault to have a remote. It commits any stale generated
indexes itself, but stops if a topic, spec, or ticket has uncommitted changes.
Point it at a different remote or branch with
`vault.sync: {remote: origin, branch: main}`.

## More

Everything below, and anything added later, is indexed at
[docs/](docs/README.md).

- [The shape of a vault](docs/vault-shape.md): the layout, namespaces, the
  frontmatter of topics, specs, and tickets, the frontier, history and sync.
- [The MCP server and agent files](docs/mcp.md): registering it in Claude Code
  or another client, the tools and resources, what `install-rules` writes,
  and how Matt Pocock's skills drive the vault.
- [Let an agent set it up](docs/agent-setup.md): a prompt to paste into your
  coding agent that does sections 1 to 4 with you, asking before it writes.
- [Troubleshooting sync](docs/troubleshooting.md): a conflict in a generated
  index versus one in a topic, spec, or ticket, a write refused mid-merge,
  and what `doctor` does and does not catch.

## Issues

Report problems and ask for things on the
[issue tracker](https://github.com/skramstudios/skram-vault/issues). The
source is not published at the moment, so there are no pull requests.

## License

The binaries are free to use and to redistribute under Apache-2.0, see
[LICENSE](LICENSE). The licences of the Go modules built into them are in
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

## Siblings

skram-vault needs none of these. They share its config file and its
`repos:` block, and nothing else.

- [skram](https://github.com/skramstudios/skram): one job queue that you and
  your coding agents share, so nobody clobbers the local cluster or Docker VM.
- [skram-tunnel](https://github.com/skramstudios/skram-tunnel): share a local
  app, and the identity provider it logs in against, on one public URL.
