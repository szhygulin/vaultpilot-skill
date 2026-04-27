# Changelog

All notable changes to the `vaultpilot-preflight` skill are documented here.
The skill is versioned separately from `vaultpilot-mcp` so an MCP compromise
cannot silently alter the skill's content.

## 0.4.1 — rename repo: `vaultpilot-skill` → `vaultpilot-security-skill`

GitHub repository renamed for clarity. GitHub provides automatic
redirects for the old URL, so existing clones continue to work, but
the in-file references inside `SKILL.md` (and the README / earlier
CHANGELOG entry) are updated to the new canonical URL. No invariants
or behavior change.

- New SKILL.md SHA-256 (URL change shifts the hash):
  `e48d5c0cdeb85be7b3a431a678d1cf2ff40aa52a69259567bb575779af75007a`.
- Sentinel unchanged at
  `VAULTPILOT_PREFLIGHT_INTEGRITY_v5_9c4a2e7f3d816b50` — same v5 release.
- **Requires `vaultpilot-mcp` ≥ 0.10.x with the matching pin bump.**
  Coordinated MCP-side PR updates `Expected SHA-256` to the new value
  AND the URL refs (`SKILL_REPO_URL`, install instructions, SECURITY.md
  cross-references). Until both ship, signing flows halt with
  `vaultpilot-preflight skill integrity check FAILED`.

## 0.4.0 — close documented agent-side gaps from `vaultpilot-mcp`'s threat model

Adds four new invariants (#9–#12) and tightens two existing ones (#2, #4),
all closing gaps where `vaultpilot-mcp`'s `SECURITY.md` documents an
agent-side defense that the skill did not yet enforce statically. A
compromised MCP that omits the corresponding server-prose block previously
disabled the defense; v0.4.0 makes the defense survive that omission.

- **Bumps integrity sentinel to**
  `VAULTPILOT_PREFLIGHT_INTEGRITY_v5_9c4a2e7f3d816b50`.
- **Adds Step 0 — Integrity self-check (MANDATORY, runs FIRST).**
  Closes [#10](https://github.com/szhygulin/vaultpilot-security-skill/issues/10).
  The previous `0.1.x`–`0.3.x` skills relied on the MCP-emitted
  `PREFLIGHT SKILL INTEGRITY PIN` block to instruct the agent to run
  `sha256sum` and compare against the pin — but the skill itself had no
  prose making this mandatory. Result: when the local `SKILL.md` had
  drifted from the MCP-pinned hash, multiple signing flows ran with no
  alarm surfaced. v0.4.0 adds an explicit Step 0 in this file (placed
  *before* the numbered invariants) that requires the agent to (a)
  extract `Expected SHA-256` and the assembled `A+B+C` sentinel from
  the MCP's pin block, (b) compute `sha256sum
  ~/.claude/skills/vaultpilot-preflight/SKILL.md` locally and compare,
  (c) confirm the sentinel is present in the skill content the `Skill`
  tool returned. Any mismatch — or any inability to complete the check
  (file missing, hash tool unavailable, MCP omitted the pin block) —
  halts the signing flow with `✗ vaultpilot-preflight skill integrity
  check FAILED — DO NOT SIGN.` and surfaces both hashes side-by-side.
  Re-runs on every signing-related call rather than caching, so mid-
  session tamper is caught.
- **Tightens Invariant #2 (hash recompute)** with an explicit `chainId`-
  field assertion in the EIP-1559 RLP. The hash recompute alone does not
  catch a chainId swap (the agent recomputes from the same tampered
  tuple); v0.4.0 adds the cross-check and a `✗ CHAIN-ID FIELD MISMATCH`
  failure phrasing.
- **Tightens Invariant #4 (missing-directive detection)** with a chain-
  conditioned table of the specific MCP-emitted blocks each
  `prepare_*` / `preview_*` response is required to contain
  (`PREPARE RECEIPT`, `[CROSS-CHECK SUMMARY]`, `LEDGER BLIND-SIGN HASH`,
  `LEDGER MESSAGE HASH`, etc.). The agent now names the specific missing
  block to the user rather than emitting a generic "directive missing"
  warning.
- **Adds Invariant #9** — WalletConnect session-topic cross-check (EVM
  only, before the first `send_transaction` of a session). Catches WC
  peer-impersonation: any peer can self-report `wallet: "Ledger Wallet"`
  / `peerUrl: "wc.apps.ledger.com"`, so the name/URL pair is not a
  trusted identity. The session topic is. The agent surfaces the last
  8 chars and asks the user to confirm a matching session in
  Ledger Live → Settings → Connected Apps.
- **Adds Invariant #10** — `pair_ledger_*` on-device address verification.
  `pair_ledger_live` / `pair_ledger_solana` / `pair_ledger_tron` establish
  the trust root for every downstream `prepare_*`. A compromised MCP can
  return any address as "your Ledger address." The agent now requires the
  user to read the address character-by-character off the Ledger device
  and confirm match before treating the pairing as successful or
  proceeding to any subsequent flow.
- **Adds Invariant #11** — ERC-20 / Permit2 approval surfacing as an
  attack class. When decoded calldata is `approve(spender, amount)` /
  `Permit2.approve(...)` / `permit(...)`, the agent surfaces spender +
  amount as a distinct CHECKS PERFORMED line; if `amount == 2^256 - 1`,
  labels `⚠ UNLIMITED APPROVAL` with revoke-path guidance. Defense-in-
  depth against the case where bytes are honest but the agent's natural-
  language summary buries the dangerous shape.
- **Adds Invariant #12** — mandatory unconditional surfacing of
  `get_verification_artifact` second-LLM check. The defense's "optional
  to use" property does not extend to "optional to surface": a
  compromised agent (the threat this defense targets) would simply never
  mention it. v0.4.0 elevates the v0.3.x CHECKS PERFORMED template's
  passive `□ SECOND-LLM CHECK — optional` checkbox to a mandatory
  verbatim sentence the agent emits on every signing flow regardless
  of whether the user asked.
- **CHECKS PERFORMED template** updated to include the new
  CHAIN-ID-FIELD line, an APPROVAL-DETECTED line conditional on
  Invariant #11, and the mandatory SECOND-LLM-AVAILABLE prose.
- **Scope** updated: adds `get_ledger_status` (used by Invariant #9) and
  `sign_message_btc` / `sign_message_ltc` / contacts CRUD as in-scope
  signing-trust-root tools.
- **Requires `vaultpilot-mcp` ≥ 0.10.0** (the release that pins the new
  v5 SHA-256 of this `SKILL.md`). On older MCP versions this skill
  fails the integrity check and halts signing flows — align versions
  or stay on skill 0.3.x until the matching MCP release ships. A
  companion issue in `vaultpilot-mcp` requests adding the phishing-
  approval class (Invariant #11) to that repo's `SECURITY.md`
  defenses table for documentation symmetry.

## 0.1.3 — remove Fast-retry mode section (coordinated with vaultpilot-mcp 0.6.1)

- Remove the **Fast-retry mode** section added in 0.1.2. Its consumer —
  the `FAST-RETRY MODE` agent-task block emitted by `vaultpilot-mcp` on
  MarginFi borrow/repay retries — was reverted upstream in vaultpilot-mcp
  PR #136 (the abridged-checks path didn't measurably reduce wall time
  and added UX divergence between the full-path and retry-path CHECKS
  output). With the MCP no longer emitting the block, the skill's
  buy-in rules were dormant; removing them eliminates misleading
  guidance that references a code path that no longer exists.
- **Requires vaultpilot-mcp ≥ 0.6.1.** On MCP 0.6.0 (which still
  carries the v0.1.2 pin) this skill fails the SHA-256 integrity check
  and halts signing flows — align versions or stay on skill 0.1.2
  until the matching MCP release ships.

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
