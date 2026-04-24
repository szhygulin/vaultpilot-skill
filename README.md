# vaultpilot-preflight — agent-side skill for `vaultpilot-mcp`

An independent Claude Code skill that enforces bytes-verification invariants
for transactions prepared by [`vaultpilot-mcp`](https://github.com/szhygulin/vaultpilot-mcp).

## Why this repo is separate

`vaultpilot-mcp` normally emits `CHECKS PERFORMED` / `VERIFY-BEFORE-SIGNING`
blocks that tell the agent how to verify bytes before signing. A compromised
MCP can silently omit those instruction blocks — with no skill installed,
the agent has no static rule to fall back on and will drop the checks. The
skill in this repo closes that gap: it loads from the user's local disk
(`~/.claude/skills/vaultpilot-preflight/SKILL.md`) and instructs the agent
to run the integrity checks regardless of what the MCP says.

The repo is **intentionally independent** of `vaultpilot-mcp`. An attacker
who compromises the MCP's release pipeline cannot push a change that
weakens or removes this skill — the skill's trust root is the user's own
clone of this repository.

## Install

```bash
git clone https://github.com/<OWNER>/vaultpilot-skill.git \
  ~/.claude/skills/vaultpilot-preflight
```

Restart Claude Code so the skill is discovered. When `vaultpilot-mcp`
starts and sees the file at
`~/.claude/skills/vaultpilot-preflight/SKILL.md`, it stops prefixing its
`prepare_*` / `preview_*` responses with the "skill not installed" warning.

## Update

```bash
cd ~/.claude/skills/vaultpilot-preflight
git pull --ff-only
```

Diff the new `SKILL.md` against the current one before pulling if you
want to audit the change. See `CHANGELOG.md` for per-version notes.

## Honest limits

- The skill instructs the agent to run local checks. It does not
  mechanically block a compromised agent from skipping them.
- A fully-coordinated agent + MCP compromise (both lying together) is
  only caught by the on-device Ledger hash match and — optionally — the
  `get_verification_artifact` second-LLM paste flow.
- `vaultpilot-mcp` also has a roadmap entry for a `PreToolUse` hook that
  would enforce the hash recompute as host-side code, not agent prose.

See `SKILL.md` for the full invariants the agent is told to enforce.

## License

MIT — see [LICENSE](./LICENSE).
