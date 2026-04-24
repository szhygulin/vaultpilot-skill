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

## Use with other agents (non-Claude clients)

The **content** of this skill (the invariants in `SKILL.md`) is agent-agnostic
— it's plain prose about how to verify bytes before signing. The **mechanism**
(YAML frontmatter + `~/.claude/skills/<name>/SKILL.md` layout) is
[Anthropic's Agent Skills format](https://www.anthropic.com/news/agent-skills),
honored by Claude Code, Claude Desktop, the Claude Agent SDK, and Claude's API.
Cursor / Cline / Windsurf / Continue / Aider don't scan that path.

To use the invariants with a non-Claude MCP client, copy the body of
[`SKILL.md`](./SKILL.md) (everything after the `---` frontmatter) into
whatever that tool uses for persistent system-prompt rules:

| Agent / IDE   | Where to paste the invariants                                      |
|---------------|--------------------------------------------------------------------|
| Cursor        | `.cursorrules` at the project root, or **User Rules** in settings  |
| Cline         | `.clinerules` at the project root                                  |
| Windsurf      | `.windsurfrules` at the project root                               |
| Continue      | `systemMessage` in the Continue config                             |
| Aider         | `chat-language` / system prompt in `.aider.conf.yml`               |
| Generic MCP   | Your client's system prompt / persistent instructions              |

`vaultpilot-mcp`'s per-call `CHECKS PERFORMED` / `VERIFY-BEFORE-SIGNING` blocks
arrive inside tool responses regardless of client, so you still get the
server-emitted layer. This skill's role is the *static*, server-independent
layer — it has to be loaded into the agent's context by some mechanism your
client actually supports.

### Silencing the MCP's missing-skill warning

`vaultpilot-mcp` checks for a file at `~/.claude/skills/vaultpilot-preflight/
SKILL.md` and prefixes every `prepare_*` / `preview_*` response with a warning
when it's absent. On non-Claude clients that warning fires even if you loaded
the invariants via `.cursorrules` etc. To suppress it, set:

```bash
export VAULTPILOT_SKILL_MARKER_PATH=/path/to/any/existing/file
```

Only do this **after** you've actually loaded the invariants into your
agent's context — otherwise you're silencing a useful signal. A reasonable
pattern is to point the env var at your non-Claude rules file itself (e.g.
`VAULTPILOT_SKILL_MARKER_PATH=$PWD/.cursorrules`) so the marker's existence
tracks the rules' existence.

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
