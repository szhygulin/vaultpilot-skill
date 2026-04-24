# Changelog

All notable changes to the `vaultpilot-preflight` skill are documented here.
The skill is versioned separately from `vaultpilot-mcp` so an MCP compromise
cannot silently alter the skill's content.

## 0.1.0 — initial release

- Static agent-side invariants for every `vaultpilot-mcp` send flow
  (EVM, TRON, Solana).
- CHECKS PERFORMED block template the agent emits even when the MCP's
  per-call agent-task block is missing.
- Missing-directive detection: the skill instructs the agent to treat
  an absent server-side agent-task block as a possible compromise
  signal and surface it to the user.
