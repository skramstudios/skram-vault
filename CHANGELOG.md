# Changelog

What changed in each skram-vault release, for someone running the binary.
Versions follow semver.

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
