# LeftSize Agent Skills

Agent skills for [LeftSize](https://leftsize.com) — Dependabot for cloud waste.

```bash
npx skills add LeftSize/agent-skills
```

Works with any agent that reads `SKILL.md`: Claude Code, Codex, Cursor, Copilot, Devin, Gemini, OpenCode, and others.

## Skills

| Skill | What it covers |
|---|---|
| [`leftsize`](skills/leftsize/SKILL.md) | List and analyse AWS and Azure findings, ask for fix steps and scripts, and snooze, ignore or report false positives — all through the GitHub CLI |

## Why there is no SDK, CLI or MCP server here

LeftSize has no proprietary API for agents. Findings are GitHub Issues, detail is one `@leftsize` comment away, and the issue comments hold the answers. Any agent that can run `gh` — or use the GitHub MCP server — is already integrated. This skill teaches it the conventions, and the footguns.

## Documentation

- [Coding agents guide](https://support.leftsize.com/agents)
- [Commands](https://support.leftsize.com/commands)
- [Rule catalog](https://leftsize.com/rules/)
- Machine-readable docs: [`llms.txt`](https://support.leftsize.com/llms.txt) · [`leftsize.com/llms.txt`](https://leftsize.com/llms.txt)

## License

[MIT](LICENSE)
