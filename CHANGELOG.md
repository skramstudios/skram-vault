# Changelog

What changed in each skram-vault release, for someone running the binary.
Versions follow semver.

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
