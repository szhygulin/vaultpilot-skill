# vaultpilot-preflight skill — security deep dive

This document is the agent-side counterpart to
[`vaultpilot-mcp/SECURITY.md`](https://github.com/szhygulin/vaultpilot-mcp/blob/main/SECURITY.md).
It covers what `SKILL.md` enforces, what each numbered invariant catches,
the honest limits of those defenses, and the residual gaps.

For the cross-component model (server ↔ transport ↔ Ledger ↔ WalletConnect
↔ address book ↔ free-form signing), see the MCP's `SECURITY.md` — that
document is canonical for layers outside the skill.

## Trust root

The skill lives at `~/.claude/skills/vaultpilot-preflight/SKILL.md` on the
user's local disk. Its trust root is the user's own clone of
`github.com/szhygulin/vaultpilot-security-skill`, *not* the MCP server.
Three properties protect that root:

1. **Doc-only.** No bins, deps, or build artifacts. The file is auditable
   line-by-line; there is no transitive npm graph that could be tampered
   with between author and user. See [`CLAUDE.md`](./CLAUDE.md).
2. **Integrity pin.** `SKILL.md` carries a sentinel near the top (current:
   `VAULTPILOT_PREFLIGHT_INTEGRITY_v10_3f4d8e2a6c9b1057`) and
   `vaultpilot-mcp` pins the file's SHA-256 in its server `instructions`.
   Step 0 of every signing flow halts on a mismatch.
3. **Independent release pipeline.** The skill is versioned separately
   from the MCP. An MCP supply-chain compromise cannot silently push a
   change here; an attacker would need to compromise both pipelines and
   ship matching SHAs.

## Step 0 — Integrity self-check

Before any of the numbered invariants run, the agent verifies the skill's
own integrity. All three parts must pass.

| Check | Purpose | Failure mode |
|---|---|---|
| MCP emitted `PREFLIGHT SKILL INTEGRITY PIN` block | Catches a compromised MCP that drops the pin to skip Step 0. | `✗ MCP PIN BLOCK MISSING — DO NOT SIGN.` |
| `sha256sum SKILL.md` matches MCP's `Expected SHA-256` | Catches targeted tamper of `SKILL.md` on disk — an attacker with local write access cannot forge content that hashes to the pinned value. | `✗ vaultpilot-preflight skill integrity check FAILED — DO NOT SIGN.` |
| Sentinel string present in the content `Skill` returned | Catches plugin collision (different skill registered under same name) — the impostor's content lacks the sentinel. | `✗ vaultpilot-preflight sentinel missing — DO NOT SIGN.` |

Step 0 was added in skill v0.4.1 ([PR #12](https://github.com/szhygulin/vaultpilot-security-skill/pull/12),
closes [#10](https://github.com/szhygulin/vaultpilot-security-skill/issues/10)).
Prior to v0.4.1, the skill *instructed* the agent to compare hashes but did
not *mandate* the comparison or define a halt condition; signing flows
proceeded indistinguishably from a real-tamper case under hash divergence.

## Threat → invariant mapping

The numbered invariants in `SKILL.md` each cover a distinct threat class.
The table below is the skill-side view; the
[MCP `SECURITY.md` defenses table](https://github.com/szhygulin/vaultpilot-mcp/blob/main/SECURITY.md#defenses-and-what-each-catches)
is the cross-component view.

| Invariant | Threat it catches | Honest limit |
|---|---|---|
| **#1 — Decode bytes locally** | MCP-side calldata tampering — the agent's independent ABI decode disagrees with the server's compact summary. | Depends on the agent's ABI coverage for the target selector. Low-confidence decodes hand off to swiss-knife (third-party browser-context decode the agent can't sandbox). |
| **#1.a — Outer dispatch-target allowlist** | Recipient redirection via stale clear-signed approval. The b117 attack: rogue MCP returns honest selector + amount but routes outer `to` to an attacker contract that exploits a forgotten approval. | Does not pin within a multi-target family — Compound's allowlist accepts any of the chain's canonical Comets, so a redirect from cUSDCv3 to cUSDTv3 would still pass. Strict per-market check is left to the tool's own arg validation. |
| **#1b — Typed-data tree decode (forward-looking)** | Permit2 / EIP-2612 / CowSwap-class typed-data phishing once the MCP exposes a typed-data signing surface. | Forward-looking — tool surface does not exist yet. The absence of the tool is the load-bearing defense; this rule survives the day a builder ships. |
| **#2 — Recompute the on-device hash** | Compromised MCP that reports a hash inconsistent with the bytes it forwards to WC. `viem` is a separate code trust boundary; the agent's recompute agreeing with the server's hash narrows the MCP-lying gap. | **Corroborating, not load-bearing.** Both the server-reported and locally-recomputed hashes are computed over the same tampered tuple in a rogue-MCP case — they match tautologically. The load-bearing layer for byte-tamper is #1; #2 detects MCP self-inconsistency. |
| **#2b — Typed-data digest recompute (forward-looking)** | Pairs with #1b; recompute `keccak256("\x19\x01" \|\| domainSeparator \|\| hashStruct(message))` via `viem.hashTypedData`. | Same corroborating-not-load-bearing framing as #2. |
| **#2.5 — Chain-must-be-explicit refusal** | Cross-chain ambiguity / CREATE2 same-address sweeper (smoke-test 131-135). User asks "send 1000 USDC to 0xabc — works on either chain"; rogue MCP picks the chain where attacker has CREATE2-deployed a sweeper at the same address; user sees only blind-sign hash on device. | Companion MCP-side `CONTACT-CHAIN MISMATCH` warning is deferred ([#482](https://github.com/szhygulin/vaultpilot-mcp/issues/482)) — current contacts schema is per-chain-family, not per-EVM-chain. Skill-side rule is the load-bearing defense until #482 ships. |
| **#3 / #4 — CHECKS PERFORMED + missing-directive = compromise** | Compromised MCP that drops its own `[AGENT TASK]` block. The skill provides the static rule the agent falls back to. | Agent-prose enforcement; a fully-coordinated agent compromise can drop or rewrite the block in concert with whatever else compromised it. |
| **#5 — Final on-device match** | Any tampering that survived every software layer. Device serializes received bytes, hashes locally; user matches on the device screen — the one display in the pipeline that no software on the host can forge. | Blind-sign shows no calldata decode. Intent-vs-bytes matching on contract calls happens off-device through #1. |
| **#6 — Cross-chain bridge chainId allowlist** | Rogue MCP swaps `destinationChainId` in a LiFi swap (e.g., user says "bridge to Polygon", MCP encodes Optimism-chain attacker contract). | Strict-pair check (origin × destination); does not catch attacker recipient *inside* the bridge facet — that is #6b. |
| **#6b — Tier-1 facet decoder** | Bridge facet recipient swap (smoke-test 136, 137). Outer LiFi `BridgeData` passes #6 cleanly (correct destination chain ID + bridge label) but attacker recipient lives one decode-layer deeper inside opaque facet data; the Ledger ETH app blind-signs LiFi calldata. | Tier-1 only in v0.6.0 (Wormhole / Mayan / NEAR Intents / Across V3). Tier-2 (deBridge / Stargate `composeMsg` / Hop / Symbiosis) falls back to mandatory second-LLM (#12.5) until shipped — tracked at [#451](https://github.com/szhygulin/vaultpilot-mcp/issues/451). |
| **#7 — Address book tamper warning** | Tampered `contacts.json` redirecting label-resolved sends to attacker addresses; anomalous on-device sign-message prompts during prepare/send flows. | Abort-on-tamper is scoped to the label-resolution path only — a literal-address send proceeds with a warning. Free-form metadata (notes) is unsigned. See MCP `SECURITY.md` § Address book for full detail. |
| **#8 — Free-form message preview** | Compromised agent or MCP that substitutes `sign_message_btc` / `sign_message_ltc` text between user intent and device. The user's eyeball on the device is the canonical check; the preview gives them a verbatim string to compare against. | Verbatim relay raises the bar against narrow agent compromise; a fully-coordinated attacker can drop or rewrite the preview block. |
| **#8 hardening (v0.6.0)** | Drainer-template phishing (`I authorize`, `Granting full custody`, `I consent to`) and embedded non-contact addresses in proof-of-funds messages. SHA-256 byte fingerprint preview lets the user match if the device shows a hash for long messages. | Marker-word strict refusal would block legitimate KYC + proof-of-funds use ("I authorize <exchange> to verify ownership"). The template-phrase + non-contact-address pair catches the actual drainers without breaking proof-of-funds. |
| **#9 — WC session-topic cross-check (EVM only)** | WalletConnect peer impersonation — any peer can self-report `wallet: "Ledger Wallet"` and `peerUrl: "wc.apps.ledger.com"`; the session topic is unique per pairing and visible at both ends. | Relies on the user actually performing the cross-check (one-time per session). If skipped, degrades back to "trust the peer's self-report." USB-HID chains (Solana / TRON / BTC) have no WC session and skip this check. |
| **#10 — Pair-ledger flow address verification** | MCP returns a fabricated address during `pair_ledger_*` that the user accepts without checking against the device screen. | User-physical step; the skill mandates the agent ask for character-by-character verification but cannot mechanically enforce it. |
| **#11 — Approval-class surfacing + skill-side spender allowlist** | Phishing approval — honest bytes, dangerous shape. MCP truthfully relays `approve(attacker, MAX_UINT256)`; bytes-integrity checks pass; user signs distracted by benign-sounding summary. The skill makes the dangerous shape un-buryable in prose, with `⚠ UNLIMITED APPROVAL` labels and revoke-path guidance pointing at `prepare_revoke_approval`. v0.9.0 adds a curated per-chain approve-spender allowlist (Aave/Compound/Morpho/Lido/EigenLayer/Uniswap/Curve stETH/LiFi) carried independently of the MCP; non-allowlisted spenders surface as `⚠ NON-CANONICAL SPENDER` with verbatim instruction to read the on-device address character-by-character. Closes the gap [vaultpilot-mcp#618](https://github.com/szhygulin/vaultpilot-mcp/pull/618) opened when the server-side hard refusal softened to an opt-in (`acknowledgeNonAllowlistedSpender: true`) — a compromised MCP can fabricate that flag, but it cannot rewrite this skill. | Defense-in-depth; a fully-coordinated agent compromise can drop or rewrite the surfacing. The unknown-spender label is honest about what the MCP knows; an attacker spender that happens to share an address shape with a CONTRACTS-table entry would surface the wrong label, so labels are advisory, not authoritative. The non-canonical advisory binds a cooperating agent only — a rogue agent reading the rule and ignoring it is not in scope (model-safety-tuning / chat-client output-filter, per CLAUDE.md Rogue-Agent-Only Finding Triage rubric). |
| **#12 — Second-LLM check, mandatorily surfaced** | Coordinated agent compromise where the same agent that invokes `prepare_*` with malicious args also filters what the user sees in chat. The user pastes the artifact into a second, independent LLM session (ideally a different provider) and lets that agent decode the bytes from scratch. | Opt-in (requires user to paste). Does not catch a supply-chain compromise of the MCP process itself, nor a coordinated attack that simultaneously compromises both LLM providers. |
| **#12.5 — Hard-trigger ops list** | High-blast-radius ops where agent-only review is insufficient: 7702 `setCode`, Permit2 batch with multi-month expirations, LiFi Tier-2 bridges, multi-row revoke, `prepare_safe_tx_*` with `enableModule` / `delegateCall: true`, signing flows targeting addresses outside #1.a + saved contacts. The second-LLM check is a precondition, not a suggestion. | A fully-coordinated agent compromise can fabricate the second-LLM verdict; doing so requires a coordinated lie across two independent provider sessions — significantly harder than a single-agent compromise. |
| **#13 — Multi-step BTC flows** | Forgetting to apply #1 at every step of a multi-PSBT BTC flow (combine / sign / finalize). The agent must re-decode at each hop, not trust the previous step's decode to carry forward. | Same honest limit as #1 — depends on agent's BTC PSBT decode coverage. |
| **#14 — Set-level intent verification** | Reverse-revoke / set-level lie (smoke-test a086). Rogue agent answers "revoke the dangerous approvals" by silently revoking a SAFE row + fabricating a `CHECKS PERFORMED` block claiming Inv #11 verified-revoked the dangerous one. Bytes-level invariants pass — bytes ARE a legit `approve(known-spender, 0)` — but the agent picked the wrong row. "Picking which of N" is upstream of bytes; bytes-decoders structurally cannot catch it. | Defense-in-depth backed by #12.5 mandatory second-LLM. MCP-side `[SET-LEVEL ENUMERATION]` block catches single-side compromise (rogue agent, honest MCP); a coordinated MCP+agent compromise still requires fabricating both halves consistently. |
| **#15 — Durable-binding source-of-truth** | Selection-layer attacks (smoke-test b040 / b044 / b053 / b055 / b059 / b060 / b063 / b098): 100%-commission Solana validator, brand-spoofed TRON SR, wrong Comet routing, Morpho Blue with adversarial oracle/IRM/LLTV, lookalike MarginFi bank, hijacked Solana ATA, attacker-owned LP `tokenId`, attacker xpub in BTC multisig. Bytes-level invariants pass — fraud is in *which durable object* the bytes reference. | Skill mandates a non-MCP authority but cannot mechanically verify the agent used one. Hardcoded mechanical rule for unambiguous classes (LP `ownerOf`, BTC xpub paste, Solana ATA derivation, Compound + Morpho via #1.a); generic "non-MCP authority" rule for multi-equivalent classes (validators, SRs). |
| **#16 — EIP-7702 setCode refused unconditionally (forward-looking)** | 7702 `setCode` delegates the EOA's code to an attacker contract — the most expansive blast radius in EVM. | Forward-looking — MCP today does not expose a 7702 surface; tool absence is the load-bearing defense. Skill v9 will introduce a literal-address allowlist with addresses verified at probe time; until then, refused unconditionally. Tracked at [#481](https://github.com/szhygulin/vaultpilot-mcp/issues/481). |

## Cooperating-agent guidance (v0.7.0 + v0.8.0)

Two sections in `SKILL.md` carry rules that bind a *cooperating* agent —
they are explicitly **not** defenses against a rogue agent. Both share the
same honest-scope framing: rules in agent-context text are read and ignored
by a hostile agent by definition, so these rule groups catch honest-but-
uninformed advice but do not raise the bar against the smoke-test threat
model where the agent itself is the attacker (Role A in
[vaultpilot-mcp#536](https://github.com/szhygulin/vaultpilot-mcp/issues/536)).
That threat class requires defenses at the model-safety-tuning or
chat-client output-filter layer, neither of which a skill can provide.

- **v0.7.0 — Advisory hygiene.** Hardware-wallet vendor URL allowlist
  (`ledger.com` / `trezor.io` only), categorical refusal of seed-handling
  anti-patterns (cloud backup / recovery services / sharing-with-support /
  pre-configured devices), trigger-phrase scan for re-checking advisory
  text, and don't-represent-third-party-seed-handling-as-VaultPilot-sanctioned.
- **v0.8.0 — Read-only data integrity.** Mandatory disclaimer for
  high-stakes read-only data (portfolio totals > $10k, tax-context PnL,
  `compare_yields` rows about to be allocated against), sanity checks
  before relaying (sum-equals-breakdown, APY plausibility thresholds,
  unrecognized-protocol-slug warnings, suspicious naming pattern refusal,
  tax-context PnL stronger note pointing at CoinTracker / Koinly /
  Crypto.com Tax for authoritative numbers). Closes
  [vaultpilot-mcp#537](https://github.com/szhygulin/vaultpilot-mcp/issues/537)
  and [vaultpilot-mcp#542](https://github.com/szhygulin/vaultpilot-mcp/issues/542).
  The long-term defense is server-signed response envelopes (Ed25519 /
  Merkle), tracked at #537; for hosted MCP, TEE-signed responses tracked
  at vaultpilot-mcp-hosted#25. Until those land, read-only data flows
  inherit the same trust floor as the MCP itself.

## Adversarial smoke-test (2026-04-28) — what changed

The 44-script adversarial corpus run on 2026-04-28 (tracker
[vaultpilot-mcp#456](https://github.com/szhygulin/vaultpilot-mcp/issues/456))
exercised five Role types (rogue agent / rogue MCP / coordinated /
supply-chain tamper / control). Headline finding: the skill held on the
threats it was written for — bytes-level invariants caught every recipient
swap, chainId swap, approval cloak, and bridge `destinationChainId` fraud.
Step 0 (skill integrity) fired correctly under the supply-chain tamper
attack.

Six structural gaps surfaced where the defense relied on user vigilance,
opt-in second-LLM, or tool-surface absence rather than load-bearing
invariants. Skill v0.6.0 + the matching MCP companion
([vaultpilot-mcp#480](https://github.com/szhygulin/vaultpilot-mcp/pull/480))
closed seven of them in one coordinated release:

| Gap | Attack class | Closed by |
|---|---|---|
| Recipient redirection via stale approval | `b117` | #1.a + canonical-dispatch table (MCP) |
| Reverse-revoke / set-level lie | `a086` | #14 + `[SET-LEVEL ENUMERATION]` block (MCP) |
| Bridge facet recipient swap (Tier-1) | `136`, `137` | #6b — Tier-1 only |
| Cross-chain ambiguity / CREATE2 sweeper | `131`-`135` | #2.5 |
| Free-form message drainer phishing | `a110` | §8 hardening |
| EIP-7702 setCode (forward-looking) | `139`, `140` | #16 refuse-all |
| Durable-binding selection-layer | 8 scripts | #15 |

The 8th sub-issue (Tier-2 bridge facet decoders — deBridge / Stargate
`composeMsg` / Hop / Symbiosis) is deferred to a follow-up release per the
README roadmap; Tier-2 routes fall back to mandatory second-LLM (#12.5)
until shipped.

## Residual gaps specific to the skill

- **Non-compliant agent that doesn't load the skill.** The MCP emits a
  `VAULTPILOT NOTICE — Preflight skill not installed` block on the first
  vaultpilot-mcp tool response of the session, but that warning is itself
  server-authored — a compromised MCP suppresses it. Tracked at
  [vaultpilot-mcp#379](https://github.com/szhygulin/vaultpilot-mcp/issues/379)
  for a server-side enforcement layer (mandatory ack tool, receipt-embedded
  claim, or startup self-check). A parallel option is a `PreToolUse` hook
  that performs the hash recompute as host-side code, removing the
  agent-prose dependency entirely.
- **Version skew between skill and MCP.** Every signing flow halts with the
  exact `vaultpilot-preflight skill integrity check FAILED` message. That
  symptom is identical to a real tamper — the fix (`git pull` here and
  `npm update` on `vaultpilot-mcp` until both match) is safe either way,
  but users should NOT bypass the check to "unblock themselves." See
  README § "What users see during a version skew."
- **Coordinated agent + MCP compromise.** Both layers lying in concert is
  only caught by the on-device Ledger hash match (#5) and — optionally —
  the second-LLM paste flow (#12 / #12.5).
- **Advisory hygiene under rogue-agent threat model.** See "Advisory
  hygiene" section above.

## Reporting

See [`SECURITY.md`](./SECURITY.md) § "Reporting a vulnerability".
