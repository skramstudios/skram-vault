# Let an agent set it up

Paste the prompt below into Claude Code (or another coding agent with a shell)
from inside the repository you want the vault to cover. It installs
`skram-vault`, creates or adopts a vault, binds this checkout to a namespace,
registers the MCP server, writes the machine-local agent files, and verifies
the result. It asks before anything is written and commits nothing to your
repository.

````text
Set up skram-vault for this repository. Work through the steps in order, show
me each command's output, and stop and ask me if a step fails or surprises you.

Reference: https://github.com/skramstudios/skram-vault (README.md, docs/mcp.md,
docs/vault-shape.md). Read the README first.

1. Ask me three things before doing anything:
   a. Where the vault should live (default ~/vault), and whether that is an
      existing vault or git repo I want to adopt.
   b. The namespace for this repository (default: the repository's name).
   c. Whether the vault should sync to a private git remote, and its URL if so.

2. Install: if `skram-vault version` fails, run
   curl -fsSL https://raw.githubusercontent.com/skramstudios/skram-vault/main/install.sh | sh
   and confirm `skram-vault version` works. If ~/.local/bin is not on my PATH,
   tell me the line to add to my shell profile; do not edit the profile yourself.

3. Create or adopt the vault. Always dry-run first and show me the diff:
   skram-vault init <path> --namespace <ns> --dry-run
   Run it again with --yes only after I approve. Run it from this repository's
   root so it can offer the repos: entry for this checkout. If it prints a
   `namespaces:` line for me to add to an existing repos: entry, show me the
   exact edit to ~/.config/skram/config.yaml and make it only with my approval.

4. If I gave a remote: in the vault directory, add it as `origin` when there is
   none, then run `skram-vault sync`. Never force-push. If the remote already
   has history, stop and ask.

5. Register the MCP server if `claude mcp list` does not show it:
   claude mcp add --scope user skram-vault -- skram-vault mcp

6. Write the machine-local agent files for this checkout, then confirm my
   repository is untouched:
   skram-vault agent install-rules --here
   git status --short        # must show none of the files it wrote
   If this repository has an AGENTS.md and no CLAUDE.md, tell me about the
   Claude Code "Project instructions" setting described in docs/mcp.md.

7. Verify: `skram-vault doctor` must print no [!!] line, and
   `skram-vault agent install-rules --check` must exit 0. Explain any [--] note.

8. Prove the loop with one real entry, not a test: ask me for one thing about
   this codebase that cost me time to learn and is not written down, record it
   with `skram-vault lore write <ns> <topic-id>` (frontmatter summary = the
   finding in one sentence), then show `skram-vault lore search` finding it and
   the commit it made in the vault's `git log`.

9. Finish with a five-line summary: vault path, namespace, config file,
   whether sync is set up, and that I must restart the agent session for the
   MCP server to appear.

Rules: never commit to or push this repository. Never write a secret, token,
or customer data into the vault. Use --dry-run wherever a command offers it.
````

## A shorter prompt for day-to-day work

Once it is set up, agents find the vault through the MCP server and the agent
files. If you want to be explicit at the start of a session:

```text
Before you start: search the skram-vault lore for anything about this area
(lore_search), and take work from the ticket frontier (ticket_list with
frontier, then ticket_set with claim). When you finish, close the ticket with
a one-line resolution, and if you learned something the code does not say,
write it to lore.
```
