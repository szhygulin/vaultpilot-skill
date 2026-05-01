# Repo conventions for Claude Code

> **Generic process rules live in `~/.claude/CLAUDE.md`** (auto-loaded by Claude Code from the private [claude-md-global](https://github.com/szhygulin/claude-md-global) repo). The rules below are project-specific.

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
`LICENSE`, `SECURITY.md`, `SKILL-SECURITY.md`. That's the whole shape
— keep PRs within it.

## Typed-Data Signing Discipline (skill-side)
- **Invariant #1b (typed-data tree decode) and Invariant #2b (digest recompute) must ship and evolve as a pair.** Never weaken the skill prose to a recompute-only check — a hash recompute over a tampered tree passes tautologically because the digest was computed *over* the swap. The load-bearing layer is #1b's field-level decode + `verifyingContract` pin; #2b is corroborating, in the same way #2 corroborates #1.
- **Off-chain typed-data signing has the worst blast-radius asymmetry in EVM.** ONE permit signature → perpetual transfer authority for the lifetime of `deadline`. Worst case (Permit2 batch with multi-year expiration on USDT, smoke-test script 126) is irrevocable once signed. The skill MUST refuse a blind-signed typed-data digest whenever the Ledger device cannot clear-sign the typed-data type for the target token / domain — the user can't distinguish `Permit{spender: TRUST_ROUTER}` from `Permit{spender: ATTACKER}` by reading a hex digest on-screen.
- **Inv #1b prose requirements** when activated: decode `domain` / `types` / `primaryType` / `message` locally; walk every address-typed field (`spender`, `to`, `receiver`, `verifyingContract`) and surface them in CHECKS PERFORMED with bold + inline-code markup; surface `deadline` / `validTo` / `expiration` with delta-from-now and flag if > 90 days; pin `verifyingContract` against a curated map (Permit2 = `0x000000000022D473030F116dDEE9F6B43aC78BA3`, USDC permit domain, CowSwap settlement, etc.) and refuse on mismatch; apply Inv #11 unlimited / long-lived rules per entry when `primaryType` ∈ `{Permit, PermitSingle, PermitBatch, Order}`.
- **Inv #2b prose requirements** when activated: independently recompute `keccak256("\x19\x01" || domainSeparator || hashStruct(message))` from the decoded tree and match against the MCP-reported digest.
- **Lockstep with `vaultpilot-mcp`.** Inv #1b / #2b currently ship as forward-looking rules — the MCP exposes no typed-data signing tool today, so the gap is closed by tool-surface absence (see `SKILL-SECURITY.md`). The skill version that lifts #1b / #2b out of "forward-looking" status MUST land in coordination with the MCP version that first exposes any typed-data signing tool (`prepare_eip2612_permit`, `prepare_permit2_*`, `prepare_cowswap_order`, `sign_typed_data_v4`, or any other `eth_signTypedData_v4` exposure — tracked at [vaultpilot-mcp#453](https://github.com/szhygulin/vaultpilot-mcp/issues/453)). The skill PR activating the rules merges first, or in the same coordinated release — never after. The moment the MCP gap closes without the skill-side activation, every existing skill defense is silently bypassed for the typed-data path.
- **How to apply.** When reviewing a skill PR that touches §1b or §2b: confirm the field-decode rule and the digest-recompute rule are both present and consistent; confirm the `verifyingContract` curated map is current; confirm Inv #11 wiring for `Permit*` / `Order` primary types is preserved. When a coordinated MCP PR adds a typed-data tool, push back on any plan that ships it without the matching skill activation in the same release window.
