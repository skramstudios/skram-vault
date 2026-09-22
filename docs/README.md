# Skram Vault documentation

A git-backed folder of markdown you and your agents share: lore (what you
learned the hard way), specs, and tickets.

These pages are written for someone who has only the binary. Start at the
[README](../README.md) for what Skram Vault is, how to install it, and the
first loop; come back here for the detail. Every page is published from the
same release as the binary, so it describes the version you are running:

```bash
skram-vault version             # which release this is
skram-vault doctor              # what this machine still needs
skram-vault lint                # what the vault itself still needs
```

## Pages

- [The shape of a vault](vault-shape.md) — the directory layout, namespaces
  and what `uses` does, the frontmatter of a lore topic, a spec, and a
  ticket, the frontier, and how history, sync, and sharing work.
- [The MCP server and agent files](mcp.md) — registering `skram-vault mcp`
  with Claude Code or another client, the tools and resources it serves,
  what `skram-vault agent install-rules` writes into a checkout and why none
  of it is committed, and using the vault from Matt Pocock's skills.
- [Let an agent set it up](agent-setup.md) — a prompt to paste into your
  coding agent that installs the binary, creates the vault, binds the
  checkout, and proves the loop, asking before it writes anything.
- [Troubleshooting sync](troubleshooting.md) — a conflict in a generated
  index against one in something you wrote, a write refused mid-merge, and
  what `doctor` and `lint` do and do not catch.

## Reporting a problem

Open an issue on the [public repository](https://github.com/skramstudios/skram-vault/issues)
with the output of `skram-vault version` and `skram-vault doctor`, your
operating system, and what you expected instead.
