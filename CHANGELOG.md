# Changelog

What changed in each skram-vault release, for someone running the binary.
Versions follow semver.

## [Unreleased]

## [0.8.0] — 2026-09-25

- `skram-vault ticket set` can now change a ticket's spec, title, and
  summary: `--spec S-2` sets the spec it serves, `--spec ""` clears it,
  `--title` retitles it, and `--summary` rewrites (or with `""` clears) its
  summary. `skram-vault spec set` gains `--title` and `--summary` too. The
  `ticket_set` and `spec_set` MCP tools take the same fields (`spec` sets
  it; `spec_set` with an empty `spec` clears it, like `blocked_by`). A spec
  id that doesn't exist in the namespace is refused and nothing is written.
  Retitling keeps the file name, so links to it keep working.
- **Breaking:** `skram-vault ticket list` and the `ticket_list` MCP tool now
  show open tickets by default, instead of every ticket ever filed. A
  namespace with a long history of closed tickets used to return a huge
  list; now you see the work that's left. Pass `--closed` (CLI) or
  `include_closed` (MCP) to see closed tickets too, or ask for
  `--status done` directly. `--frontier` and `--all` already only showed
  open work, so they behave the same as before.
- Each ticket in that list is now a shorter summary (id, status, title,
  summary, spec, blockers, assignee, category) instead of the full record;
  `ticket show` still gives you everything. `spec list` still shows specs
  of every status, just without the file path on each one.

## [0.7.0] — 2026-09-23

- **Breaking:** `skram-vault agent import-memory` is now
  `skram-vault agent promote-memory`, and it no longer promotes memories in
  bulk by their type label. Run without names, it only lists each memory
  with its type label, its summary, and `IN LORE`: where a lore topic with
  the same id already exists, in any namespace. It never writes. `--json`
  gives the same listing as one document. To promote, name the memories:
  `promote-memory --here <name>... [--yes] [--prune]` promotes exactly
  those, whatever their type. A named memory that is already in lore
  anywhere is refused and the existing topic is named; the other names
  still go ahead. `--yes` or `--prune` without names is now an error that
  says what to do. `import-memory` still works for this release: it prints
  `import-memory is now promote-memory` and runs.
- Memory files with an unquoted `": "` in their description (the shape
  Claude Code writes, such as "React 19 replaces javascript: hrefs …") are
  now read instead of reported "unreadable": when strict parsing fails,
  `promote-memory` falls back to reading the name, description and type
  directly. A file whose name still cannot be recovered stays unreadable.
- `skram-vault doctor` no longer notes a checkout's unpromoted memories.
  Deciding what to promote is now entirely `promote-memory`'s job.
- `promote-memory`'s listing now hints when a memory's id is close to, but
  not exactly, an existing lore topic — for example
  `react19-javascript-href-blocked` next to
  `shared/react19-blocks-javascript-href`. The `IN LORE` column shows the
  best match as `<namespace>/<id>?   (similar id)`, and `--json` adds
  `similar: ["<namespace>/<id>", ...]` to the memory. This is informational
  only: naming a memory with a similar id still promotes it.

## [0.6.0] — 2026-09-23

- `skram-vault agent import-memory` promotes the durable half of Claude
  Code's per-checkout memory (`feedback` and `reference` memories) into lore
  topics. It is a dry run by default: it reports what it would promote, what
  it leaves alone and why, and any id that already names a topic, which is
  reported as a conflict and never overwritten. `--yes`, or answering the
  prompt at a terminal, writes one commit per namespace; `--prune` then
  deletes the memory files that commit promoted and their `MEMORY.md` lines.
- `skram-vault doctor` notes a checkout whose memory holds durable memories
  not yet in lore. The note is informational and never changes the exit code.
- The Knowledge vault section `agent install-rules` writes gains a line
  sending durable facts to lore rather than editor memory. Existing checkouts
  read as stale under `--check` and `doctor` until `install-rules` runs there
  again.
- `agent install-rules --here` works from a linked worktree of a checkout
  whose `repos:` entry matches by `paths:` alone.

## [0.5.0] — 2026-09-22

- `skram-vault agent install-rules` (and `--check` and `doctor`) also covers
  each checkout's linked worktrees: a `git worktree add` beside a covered
  checkout gets the Knowledge vault section on the next run, with the same
  namespaces as the checkout it came from. A worktree whose directory is
  gone is skipped.

## [0.4.0] — 2026-09-22

- Every write (lore, spec, ticket) syncs with the vault's remote: it pulls
  before it allocates an id or commits and pushes after, so two machines
  writing to one shared vault no longer hand out the same `T-<n>` or `S-<n>`.
  An unreachable remote warns and the write still commits locally.
  `--no-sync` (or `SKRAM_VAULT_SYNC=0`) turns it off.
- `skram-vault lint` and `doctor` report `duplicate-id` when two files in one
  namespace claim the same id, and a sync merge that conflicts on
  `.vault-ids.json` keeps the higher counter.
- `skram-vault doctor` reports a vault left mid-merge or mid-rebase as a
  problem, and ahead, behind, or diverged from its remote as a note. `lint`
  and `doctor` report a git conflict marker left in a file. A write on a
  vault mid-merge says how to finish or abort it. Before this, all of these
  reported ok.

## [0.3.0] — 2026-09-21

- Documentation, in `docs/`: an index (`docs/README.md`), and
  `docs/vault-shape.md` now covers every `vault:` config key with its default,
  the `spec` and `ticket show` commands, how the `lore` subcommands relate to
  the root ones, and how the commit author is chosen (`SKRAM_ACTOR`). Every
  command you can see, every `SKRAM_*` variable, and every config key is in
  the docs; a release cannot be tagged otherwise.
- `docs/troubleshooting.md`: what to do when a sync merge conflicts in a
  generated index or in a topic, spec, or ticket, why a write is refused
  mid-merge, and what `doctor` and `lint` do and do not catch.
- `skram-vault --help` opens with the same sentence as the README.
- `ticket list --frontier --spec S-n` (and the `ticket_list` tool with both)
  now lists that spec's own frontier instead of the whole namespace's.
  `--all` with `--spec` or `--status` is refused instead of silently ignoring
  them.
- The release page on GitHub shows its notes.

## [0.2.1] — 2026-09-21

First public release.

- `skram-vault`, a single binary for macOS and Linux (amd64 and arm64): a
  git-backed folder of markdown lore, specs, and tickets, with a CLI and an
  MCP server over the same vault.
- `install.sh` downloads the latest release for your machine, verifies it
  against `checksums.txt`, and installs to `~/.local/bin` (`VERSION`,
  `BIN_DIR`). No GitHub login is needed.
- `skram-vault init` creates a vault and its config block, additively, with
  `--dry-run` to see the change first.
- Docs: the shape of a vault (`docs/vault-shape.md`), the MCP server and agent
  files (`docs/mcp.md`), and a prompt that has a coding agent do the setup
  with you (`docs/agent-setup.md`).
- `THIRD_PARTY_NOTICES.md`, in the repository and in every release archive.
