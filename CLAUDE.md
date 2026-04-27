# Repo conventions for Claude Code

## This skill ships zero buildable artifacts

`vaultpilot-security-skill` is **doc-only by design**. Do not add:

- `bin/` executables (Node CLIs, shell scripts, etc.)
- `package.json` / `package-lock.json` / runtime dependencies
- Any compiled or buildable artifact

The trust root is the user's own `git clone` of this repo. A doc-only
tree is auditable line-by-line; a Node bin pulls in `@solana/web3.js`
and its transitive dependency graph (npm supply chain), which is
exactly the kind of surface the skill is designed to *defend against*.
Every dep added here erodes the "the skill cannot itself be the
attack vector" property.

If a verification helper is genuinely needed, it lives in
`vaultpilot-mcp` (or a separate purpose-built package the user opts
into), not here. Closing the agent-side gap is done via prose
invariants in `SKILL.md`, not by shipping code.

## What this repo *does* contain

`SKILL.md` (the agent-side invariants), `README.md`, `CHANGELOG.md`,
`LICENSE`. That's the whole shape — keep PRs within it.
