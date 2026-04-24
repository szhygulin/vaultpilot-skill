# Changelog

All notable changes to the `vaultpilot-preflight` skill are documented here.
The skill is versioned separately from `vaultpilot-mcp` so an MCP compromise
cannot silently alter the skill's content.

## 0.1.2 — Fast-retry mode for MarginFi borrow/repay (coordinated with vaultpilot-mcp 0.5.4)

- Add **Fast-retry mode** section documenting the skill's buy-in rules for
  the abridged CHECKS template that `vaultpilot-mcp` ≥ 0.5.4 may emit on
  MarginFi borrow/repay retries after a Switchboard transient-oracle
  failure. Invariant 2 (pair-consistency Ledger hash) stays mandatory.
  Invariant 1 (full instruction decode) may be substituted by CHECK B
  (program-id whitelist) + CHECK C (semantic-args match vs. the prior
  approval) iff five conditions all hold — most importantly, the agent
  must remember matching the `priorLedgerHash` on-device earlier in the
  same session.
- **Requires a vaultpilot-mcp release that carries the matching pin.**
  On older MCP versions the server never emits a `FAST-RETRY MODE` block,
  so the new section is inert — behavior degrades back to 0.1.1.

## 0.1.1 — integrity pin (coordinated with vaultpilot-mcp 0.5.3)

- Added an in-file integrity sentinel
  (`VAULTPILOT_PREFLIGHT_INTEGRITY_v1_7780bfeee9a49f01`) as an HTML comment
  near the top of `SKILL.md`. Invisible when the file is rendered, present
  in the raw text the `Skill` tool returns.
- `vaultpilot-mcp` now pins the expected SHA-256 of this `SKILL.md` file in
  its server source code and instructs the agent to verify both the hash
  and the sentinel on every signing flow. See the README "Integrity pin"
  section for the attack classes this defends against and the
  coordinated-release workflow maintainers must follow when editing this
  file.
- **Requires a vaultpilot-mcp release that carries the matching pin.**
  On older MCP versions the agent won't run the hash check — behavior
  degrades back to 0.1.0.

## 0.1.0 — initial release

- Static agent-side invariants for every `vaultpilot-mcp` send flow
  (EVM, TRON, Solana).
- CHECKS PERFORMED block template the agent emits even when the MCP's
  per-call agent-task block is missing.
- Missing-directive detection: the skill instructs the agent to treat
  an absent server-side agent-task block as a possible compromise
  signal and surface it to the user.
