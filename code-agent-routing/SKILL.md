---
name: code-agent-routing
description: "Use for delegating coding tasks or switching the code agent."
version: 1.0.0
---

# Code-Agent Routing (Lufria's hybrid setup)

Lufria works hybrid: Hermes handles exploration/lint/test runs (0 quota cost), heavy refactor/logic goes to an external code agent. A primary agent is switchable at runtime; auto-fallback applies when it hits limits.

## The `/aiagent` convention — CRITICAL

`/aiagent` is NOT a registered Hermes slash command, and **Discord rejects unknown leading-slash messages before they reach Hermes** ("Unknown command /aiagent"). Never tell the user to type slash commands that aren't in the registry.

Working convention (plain message, no slash):
```
aiagent claude-code          → switch primary
aiagent claude-code opus     → switch + pin model
aiagent opencode             → switch back
aiagent status               → show active config
```
On switch: confirm + persist to user memory (`AI-agent primary AKTIF: …`). Manual override AND auto-fallback are both enabled (user chose "dua-duanya").

## Current agent setup

1. **Claude Code** — installed and authenticated. Owner-assigned focus: critical/complex backend, database, authentication, architecture, security, and system work.
2. **Codex CLI 0.151.0** — installed at `/usr/local/bin/codex` and authenticated via ChatGPT OAuth. Owner-assigned focus: UI/UX, frontend, layout, visual components, animation, responsive design, and polishing.
3. **Fallback behavior:** when Claude Code reaches its limit, Codex continues suitable implementation work. Hermes remains the coordinator and verifier; it handles exploration, commands, routine debugging, tests, lint, builds, and regression verification.
4. **OpenCode** is not part of the active chain unless explicitly re-enabled later.

## Claude Code model routing (auto-pick per task)

| Task | Model | Extra flags |
|---|---|---|
| Complex refactor / deep logic | `--model opus` | `--effort high`, generous `--max-turns` |
| Normal feature / bug fix | `--model sonnet` | default effort |
| Review diff / light analysis | `--model haiku` | `--max-turns 1-2` |

Always pass `--fallback-model haiku` (print mode) so overload degrades gracefully. Prefer print mode (`claude -p`) with `workdir` set; tmux only for multi-turn.

## GitHub publishing safety learned

- Treat any PAT pasted into Discord as compromised; advise immediate revocation and never reuse it.
- Prefer GitHub device flow or local terminal input over chat-delivered secrets.
- Shell snippets must be copy-paste-safe in Discord: avoid token-like placeholder text that masking systems may rewrite, avoid brittle `sed` expressions for secret replacement, and verify the file has exactly one non-empty token entry with mode `0600` without printing the value.
- Never embed PATs in Git remotes; use SSH, credential helpers, or a temporary askpass process with the token kept out of command arguments and output.

- Hermes direct: file exploration, grep, linting, test runs, verification → 0 Claude tokens.
- Claude Code: focused refactor/logic prompts with pre-extracted context (`-p` style), not whole-repo exploration.

## Clipper-Apps UI delegation notes

Ranking detail page (`ranking_detail.html`, ~1000 lines): 8 sections stacked vertically with a sticky stage rail (`components/_ranking_stages.html`), scroll-spy via IntersectionObserver. Backend routers FROZEN — form action/method/input names preserved verbatim on any UI rebuild; verify templates compile via `app.template_engine`, then `systemctl restart clipper-web clipper-worker`. Agreed next rebuild (RA phases): accordion layout (not wizard — workflow is non-linear), AJAX form submits (no reload), live job polling + detailed progress bar.
