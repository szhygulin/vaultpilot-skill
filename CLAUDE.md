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
`LICENSE`, `SECURITY.md`, `SKILL-SECURITY.md`. That's the whole shape
— keep PRs within it.

## Security Incident Response Tone
- When diagnosing malware/compromise, start with evidence-based scoping before recommending destructive actions (wipe, nuke, rotate-all). Never delete evidence files before reading them.

## Chat Output Formatting
- Prefer Markdown hyperlinks over raw URLs everywhere: `[label](url)` instead of pasting the full URL inline. This keeps the chat scannable — long URLs (especially swiss-knife decoder URLs with multi-KB calldata query strings, Etherscan tx URLs with hashes, tenderly/phalcon simulation URLs) wrap the terminal into unreadable walls when pasted raw. Apply in user-facing responses AND in any text the server instructs the agent to render (verification blocks, prepare receipts, etc.). Raw URLs are acceptable only when the link is short and already scannable (e.g. a bare domain like `https://ledger.com`) or when explicitly required for machine-readable contexts (e.g. inside a JSON paste-block the user copies into another tool).

## Push-Back Discipline
- **If the user's request is built on a faulty premise that means the action won't achieve their stated goal, push back BEFORE acting — don't execute and then add a footnote.** Mid-response caveats ("Important caveat: this won't actually fix the thing you asked for") are evidence the wrong action was taken. The right move is to stop, surface the premise mismatch in plain terms, and ask which way to go.
- Concrete tells that you're about to execute a misguided ask:
  - Re-running a workflow against a frozen tag/commit/branch that predates the fix the user is trying to apply.
  - Re-broadcasting a tx with the same nonce when the original was confirmed (would just revert).
  - Re-querying an API with the same args after a deterministic failure.
  - Wrapping a destructive action with a comment like "this won't really do what you want, but doing it anyway".
- Format for push-back: one sentence stating the mismatch + the two or three concrete alternative paths + a question about which to pursue. Keep it short — the goal is to unblock the user's decision, not lecture.
- If the user explicitly says "do it anyway" after the push-back, proceed. The discipline is about surfacing the issue, not vetoing the user.
- **Past incident (2026-04-27)**: user asked to retrigger release-binaries.yml against the v0.9.4 tag to recover a missing macos-arm64 binary upload. The tag was frozen at a commit that predated the size + retry fixes (#346 / #349 / #361) just merged into main, so the rerun would have produced the same broken-upload-prone 504MB binary. Caught the issue mid-response but executed the rerun anyway. Right move was to flag the frozen-tag problem first and recommend cutting v0.9.5 (with all fixes baked in) as the alternative.

## Smallest-Solution Discipline
- **When assessing a GitHub issue or implementing a plan entry, push back with the smallest solution that actually solves the stated problem.** Before writing code, ask: what is the minimum change that makes the failing case pass? Propose that first; only escalate to a heavier design if the minimum demonstrably doesn't cover the requirement. The plan or issue text is a description of the problem, not a license to build infrastructure around it.
- Concrete tells that the proposed solution is too big:
  - Caching, indexing, or persisting a dataset to "simulate" or "replay" one operation (e.g. keeping a private blockchain fork in memory to re-run a single transaction; building a request log to replay one API call). One-shot operations don't need persistence layers.
  - Adding a new module, abstraction, or config surface when an inline change to the existing call site would do.
  - Introducing a background worker, queue, or scheduler for an action that fires once per user request.
  - Generalizing for hypothetical future callers when there is exactly one caller today.
  - "While I'm here" refactors bundled into a fix PR.
- Format for push-back: one sentence stating the smallest fix you see + one sentence on what the larger proposal adds beyond that + a question on which scope to pursue. If the issue/plan author already specified the heavy approach, surface the lighter alternative explicitly so the user can choose — don't silently downscope either.
- If the user explicitly says the larger scope is intended, proceed. The discipline is about not defaulting to the heavy option, not vetoing it.

## Typed-Data Signing Discipline (skill-side)
- **Invariant #1b (typed-data tree decode) and Invariant #2b (digest recompute) must ship and evolve as a pair.** Never weaken the skill prose to a recompute-only check — a hash recompute over a tampered tree passes tautologically because the digest was computed *over* the swap. The load-bearing layer is #1b's field-level decode + `verifyingContract` pin; #2b is corroborating, in the same way #2 corroborates #1.
- **Off-chain typed-data signing has the worst blast-radius asymmetry in EVM.** ONE permit signature → perpetual transfer authority for the lifetime of `deadline`. Worst case (Permit2 batch with multi-year expiration on USDT, smoke-test script 126) is irrevocable once signed. The skill MUST refuse a blind-signed typed-data digest whenever the Ledger device cannot clear-sign the typed-data type for the target token / domain — the user can't distinguish `Permit{spender: TRUST_ROUTER}` from `Permit{spender: ATTACKER}` by reading a hex digest on-screen.
- **Inv #1b prose requirements** when activated: decode `domain` / `types` / `primaryType` / `message` locally; walk every address-typed field (`spender`, `to`, `receiver`, `verifyingContract`) and surface them in CHECKS PERFORMED with bold + inline-code markup; surface `deadline` / `validTo` / `expiration` with delta-from-now and flag if > 90 days; pin `verifyingContract` against a curated map (Permit2 = `0x000000000022D473030F116dDEE9F6B43aC78BA3`, USDC permit domain, CowSwap settlement, etc.) and refuse on mismatch; apply Inv #11 unlimited / long-lived rules per entry when `primaryType` ∈ `{Permit, PermitSingle, PermitBatch, Order}`.
- **Inv #2b prose requirements** when activated: independently recompute `keccak256("\x19\x01" || domainSeparator || hashStruct(message))` from the decoded tree and match against the MCP-reported digest.
- **Lockstep with `vaultpilot-mcp`.** Inv #1b / #2b currently ship as forward-looking rules — the MCP exposes no typed-data signing tool today, so the gap is closed by tool-surface absence (see `SKILL-SECURITY.md`). The skill version that lifts #1b / #2b out of "forward-looking" status MUST land in coordination with the MCP version that first exposes any typed-data signing tool (`prepare_eip2612_permit`, `prepare_permit2_*`, `prepare_cowswap_order`, `sign_typed_data_v4`, or any other `eth_signTypedData_v4` exposure — tracked at [vaultpilot-mcp#453](https://github.com/szhygulin/vaultpilot-mcp/issues/453)). The skill PR activating the rules merges first, or in the same coordinated release — never after. The moment the MCP gap closes without the skill-side activation, every existing skill defense is silently bypassed for the typed-data path.
- **How to apply.** When reviewing a skill PR that touches §1b or §2b: confirm the field-decode rule and the digest-recompute rule are both present and consistent; confirm the `verifyingContract` curated map is current; confirm Inv #11 wiring for `Permit*` / `Order` primary types is preserved. When a coordinated MCP PR adds a typed-data tool, push back on any plan that ships it without the matching skill activation in the same release window.
