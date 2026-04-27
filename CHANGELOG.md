# Changelog

All notable changes to the `vaultpilot-preflight` skill are documented here.
The skill is versioned separately from `vaultpilot-mcp` so an MCP compromise
cannot silently alter the skill's content.

## 0.4.2 — `vaultpilot-verify-solana-msg` bin (closes #15)

Ships an optional Node CLI, `vaultpilot-verify-solana-msg`, that
recomputes the Ledger Solana Message Hash (`base58(sha256(messageBytes))`)
and decodes a message's instructions. Faithful port of the
`combinedCheckScript` in `vaultpilot-mcp`'s `render-verification.ts`.

When the bin is on PATH, `vaultpilot-mcp` (next coordinated release)
will emit `vaultpilot-verify-solana-msg '<msgB64>'` in its Solana
agent-task block instead of a 30-line inline `node -e` script.
Allows users to allowlist `Bash(vaultpilot-verify-solana-msg:*)`
per-project rather than granting the broader `Bash(node:*)`.
Reverses the security regression of approving an opaque inline
script on every Solana send (vaultpilot-mcp #396).

- Adds `bin/vaultpilot-verify-solana-msg`, `package.json` (deps:
  `@solana/web3.js`).
- README documents `npm install` + symlink-into-`~/.local/bin/` step.
- `SKILL.md` Invariant #1 Solana branch now names the bin as the
  preferred CHECK 1 + CHECK 2 verifier and instructs the agent to
  surface the MCP-rendered labeling line before invoking it.
- New SKILL.md SHA-256:
  `763872eb5cbebc310e21d15d0c3dbec112d72e4267203170b6e4f914d8952c48`.
- Sentinel unchanged at
  `VAULTPILOT_PREFLIGHT_INTEGRITY_v5_9c4a2e7f3d816b50` — text-only
  addition, no protocol change.
- **Requires `vaultpilot-mcp` with the matching pin bump.**
  Coordinated MCP-side PR will (a) update `EXPECTED_SKILL_SHA256` to
  the new value and (b) detect the bin on PATH and emit the named-bin
  form in `renderSolanaAgentTaskBlock`. Until both ship, signing
  flows halt with `vaultpilot-preflight skill integrity check FAILED`.

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

## 0.3.1 — Invariant #8 (free-form message signing) + #7 carve-out + Safe note (coordinated with vaultpilot-mcp 0.9.3)

Backfilled — entry was missing from this changelog at release time. Source:
PR [#5](https://github.com/szhygulin/vaultpilot-security-skill/pull/5),
merged commit `13a2886`.

- **Invariant #8 — `MESSAGE-PREVIEW` block for `sign_message_btc` /
  `sign_message_ltc`.** Free-form BIP-137 message signing flows the
  user-supplied UTF-8 string through user → agent → MCP → device, and
  a compromised agent or MCP can substitute the message text in flight
  (the user asks to sign `"I own bc1q...mine"`, the device shows
  `"I authorize transfer of all funds to bc1q...attacker"`). Invariant
  #8 has the agent render the EXACT UTF-8 string in a fenced verbatim
  CHECKS PERFORMED block before invoking the tool, ask the user to
  confirm character-by-character match against the device screen, and
  treat any deviation as `✗ MESSAGE-PREVIEW MISMATCH — DO NOT SIGN.`
- **Invariant #7 carve-out.** Added the on-device sign-message
  allowlist (contacts CRUD + `sign_message_btc` / `sign_message_ltc`)
  so any sign-message prompt during a `prepare_*` / `send_transaction`
  flow triggers the anomaly path.
- **Safe-multisig note.** Documented that
  `prepare_safe_tx_propose` / `_approve` / `_execute` are ordinary
  `eth_sendTransaction` calls (Safe uses on-chain `approveHash`, not
  EIP-712 typed-data) — the agent decodes the OUTER selector
  (`Safe.approveHash` / `Safe.execTransaction`) and the INNER tx the
  Safe is being asked to authorize.
- **Bumps integrity sentinel to**
  `VAULTPILOT_PREFLIGHT_INTEGRITY_v4_7655818578c7a044`.
- **SKILL.md SHA-256:**
  `cd689838314a700dfff80d4c881bf51190cd6c71c747f3152e7cab8d943df2cc`.
- **Requires vaultpilot-mcp ≥ 0.9.3** (the release that pinned the v4
  SHA via [`vaultpilot-mcp` commit `8f22f51`](https://github.com/szhygulin/vaultpilot-mcp/commit/8f22f51)).

## 0.3.0 — Invariant #7 (address-book label decoration + tamper warnings) (coordinated with vaultpilot-mcp 0.9.3)

Backfilled — entry was missing from this changelog at release time. Source:
PR [#4](https://github.com/szhygulin/vaultpilot-security-skill/pull/4),
merged commit `7afeb5f`.

- **Invariant #7 — Address-book.** Surfaces label decorations on the
  recipient line in `CHECKS PERFORMED`:
  `(contact: <label> — verified)` for label-resolved sends,
  `(also saved as: <label>)` for reverse-decorated literal addresses,
  `(resolved via ENS, also saved as: <label>)` for ENS + reverse, and
  `(unknown — verify on-device)` for clean-but-unsaved destinations.
  When the verification block carries
  `⚠ contacts file failed verification`, the agent leads its reply
  with `⚠ CONTACTS-FILE TAMPER WARNING` and refuses to call
  `send_transaction` until the user confirms the recipient out-of-band.
  Cross-checks `verify_contacts({ chain })` after every successful
  `add_contact` to catch the case where the MCP claims success but the
  persisted blob doesn't actually verify.
- **Bumps integrity sentinel to**
  `VAULTPILOT_PREFLIGHT_INTEGRITY_v3_2d3b876b38550fe5`.
- **SKILL.md SHA-256:**
  `21c8c60ac26528732dbe0b40b4be7e4790607db1881489fe4cd1751f75536cd6`.
- **Requires vaultpilot-mcp ≥ 0.9.3** (the release that pinned the v3
  SHA via [`vaultpilot-mcp` commit `abf16e3`](https://github.com/szhygulin/vaultpilot-mcp/commit/abf16e3)).

## 0.2.0 — Invariant #6 (NEAR-Intents intermediate-chain cross-check) (coordinated with vaultpilot-mcp 0.9.1)

Backfilled — entry was missing from this changelog at release time. Source:
PR [#3](https://github.com/szhygulin/vaultpilot-security-skill/pull/3),
merged commit `90b4432`.

- **Invariant #6 — Cross-chain bridge chain-ID cross-check.** When
  `prepare_swap` / `prepare_solana_lifi_swap` / `prepare_tron_lifi_swap`
  returns LiFi `BridgeData` calldata for a CROSS-CHAIN bridge, the
  skill independently verifies the encoded `destinationChainId`
  against a chain-ID table baked into the skill (NOT taken from the
  MCP). An allowlist of known intermediate-chain bridges (NEAR
  Intents, settling on `1885080386571452`) covers the legitimate case
  where the bridge encodes its own settlement chain rather than the
  user's final chain. Any mismatch outside the allowlist is a chain-
  ID-swap attack and the agent leads with
  `✗ CHAIN-ID MISMATCH FAILED — DO NOT SIGN.` The skill carries the
  table independently of the MCP's
  `src/modules/swap/intermediate-chain-bridges.ts` so a single-side
  tamper (in either) is caught.
- **Bumps integrity sentinel to**
  `VAULTPILOT_PREFLIGHT_INTEGRITY_v2_43b1d2403a0c2a94`.
- **SKILL.md SHA-256:**
  `fc4a40469e11156e7cccdc1539b93e430b60a49bf75d01b41de112d8c301f1a8`.
- **Requires vaultpilot-mcp ≥ 0.9.1** (the release that pinned the v2
  SHA via [`vaultpilot-mcp` commit `5d495a3`](https://github.com/szhygulin/vaultpilot-mcp/commit/5d495a3)).

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
