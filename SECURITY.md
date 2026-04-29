# Security model

`vaultpilot-security-skill` is one component in a multi-layer security model.
Both halves of the model are documented:

- **This file** scopes what this repository is responsible for, names what it
  explicitly does *not* defend, and points at the canonical cross-component
  picture.
- **[`SKILL-SECURITY.md`](./SKILL-SECURITY.md)** is the deep dive for this
  repo: trust root, integrity pin, the threat class each numbered invariant
  in `SKILL.md` catches, and the honest limits.
- **[`vaultpilot-mcp/SECURITY.md`](https://github.com/szhygulin/vaultpilot-mcp/blob/main/SECURITY.md)**
  is the canonical multi-layer model (server, transport, Ledger device,
  WalletConnect, address book, free-form signing). When in doubt, that
  document is authoritative for cross-component questions; this repo
  documents only the agent-side slice.

## Scope of this repository

This repo ships a single Claude Code skill —
`~/.claude/skills/vaultpilot-preflight/SKILL.md` — that the agent loads on
every `vaultpilot-mcp` tool call. The skill's job is to enforce *agent-side*
integrity invariants that don't depend on the MCP's own `CHECKS PERFORMED` /
`[AGENT TASK]` directives. A compromised MCP that omits its own directives
still gets caught by these invariants.

What this repo defends:

- Bytes-level integrity of prepared transactions (decode `data` locally,
  recompute the pre-sign hash, match the device hash).
- Honest-bytes-but-dangerous-shape attacks (unlimited approvals,
  drainer-template message signing, dispatch redirect to attacker contracts).
- Selection-layer attacks (validators / SRs / Comets / markets / banks /
  ATAs / LP `tokenId` / multisig xpub) by mandating the candidate be sourced
  from a non-MCP authority.
- Cross-chain ambiguity / CREATE2 same-address sweepers.
- Supply-chain tamper of `SKILL.md` itself (caught by the integrity pin,
  halts before any signing).

What this repo does **not** defend:

- A fully-coordinated agent compromise that lies on both args and output —
  the on-device Ledger hash match and the optional `get_verification_artifact`
  second-LLM paste flow are the backstops.
- An agent that decides to ignore the skill entirely (no skill loaded, prose
  dropped from context). Tracked at
  [vaultpilot-mcp#379](https://github.com/szhygulin/vaultpilot-mcp/issues/379)
  for a server-side enforcement layer (mandatory ack tool, receipt-embedded
  claim, or `PreToolUse` hook that performs the hash recompute as host-side
  code).
- Cooperating-agent guidance rules (v0.7.0 advisory hygiene, v0.8.0
  read-only data integrity) under a rogue-agent threat model. These rules
  catch honest-but-uninformed advice from a cooperating agent; a hostile
  agent reads them and ignores them by definition. Defenses there belong at
  the model-safety-tuning or chat-client output-filter layer, neither of
  which this skill can provide. Cryptographic verification of read-only
  responses (server-signed envelopes) is the long-term architectural fix —
  tracked at [vaultpilot-mcp#537](https://github.com/szhygulin/vaultpilot-mcp/issues/537).

## Why this repo is a separate trust root

The MCP cannot reach this repo to alter or suppress its content. This repo's
trust root is the user's own `git clone` — the same auditable file the
integrity-pin SHA covers. To preserve that property the repo is **doc-only
by design**: no `bin/`, no `package.json`, no runtime dependencies, no
buildable artifacts. Every dep added here would erode the property the skill
exists to defend. See [`CLAUDE.md`](./CLAUDE.md) for the policy.

## Reporting a vulnerability

If you believe you've found a security issue in this skill or its
integrity-pin coordination with `vaultpilot-mcp`, please open a GitHub
security advisory at
<https://github.com/szhygulin/vaultpilot-security-skill/security/advisories/new>
rather than a public issue. Include a reproduction, the affected skill
version (sentinel + SHA-256 of `SKILL.md`), and your assessment of the
impact.

For issues in the MCP server itself, transport, or other layers documented
in [`vaultpilot-mcp/SECURITY.md`](https://github.com/szhygulin/vaultpilot-mcp/blob/main/SECURITY.md),
please report to that repo's security advisory page instead.
