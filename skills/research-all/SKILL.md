---
name: research-all
description: The user's default research skill. Fans out one question across every available research lane — multi-search-engine SERP coverage, deep-research synthesis, last30days community pulse, agent-reach platforms, web-access browser reading, academic search, and a codebase lane — then merges them into one cited answer. Works in Claude Code, Codex, and WorkBuddy with runtime fallbacks. Use it EVERY time the user says 조사, 조사해줘, 리서치, research, 알아봐, 찾아봐, "다 돌려", or asks what is needed to build something and the answer depends on outside sources. Use it instead of calling deep-research, multi-search-engine, web-access, last30days, agent-reach or search alone. Skip only for a one-fact lookup that needs no sources.
---

# research-all

One question in, every available research lane out in parallel, one merged answer back. The value is breadth plus cross-checking: a claim two lanes found independently is stronger than a claim one lane found, and gaps show up as lanes that came back empty.

Lanes degrade gracefully. A missing skill, tool or browser is a fallback or a `needs setup` status, never a reason to abort the whole run.

## 0. Runtime probe (once per run)

Do not assume Claude Code. Check what this runtime exposes before planning the fan-out.

| Capability | If present | If absent |
|---|---|---|
| `Skill` tool | load each lane's skill by name | read that lane's `SKILL.md` from the skill roots below and follow it |
| `Agent` tool | spawn one subagent per lane in one message | run the lanes sequentially in this session with the same prompts |
| `WebSearch` / `WebFetch` | use them for discovery and extraction | use `curl` plus the search-engine URLs in `multi-search-engine` |
| Local Chrome + CDP | `web-access` browser lane | skip CDP and mark `needs setup: Chrome CDP` |
| Academic MCP tools | Consensus / PubMed | WebSearch on scholar sources |

Skill roots, first match wins:

- WorkBuddy: `~/.workbuddy-ai/skills/<name>/SKILL.md`
- Claude Code: `~/.claude/skills/<name>/SKILL.md`
- Codex and shared tools: `~/.agents/skills/<name>/SKILL.md`

If a lane's skill is missing, use its fallback column in the table below. Do not stop the run.

## 1. Frame the question (inline, no agent)

Write a 2-4 line brief: the question, what a useful answer looks like, the time window that matters, and the languages/regions to cover (for a Korean-market topic, Korean sources first). If the question names a local project ("이 엔진", a repo path, "our code"), note the path; that turns on the codebase lane. If the question is too vague to brief, ask one question first.

## 2. Fan out: one subagent per lane, all in the same message

Spawn the lanes below with the Agent tool in a single message so they run concurrently. If the runtime has no Agent tool, run them sequentially in the same order. Prompts to subagents are in English. Each prompt contains the brief, names the skill the agent must load, and asks for the return format in step 3.

| Lane | Primary | Fallback | What it is for |
|---|---|---|---|
| web | `deep-research` | WebSearch + WebFetch | Broad web synthesis, official docs, existing products |
| search | `multi-search-engine` | Bright Data `search`, then `fetch` for blocked pages | Multi-engine SERP coverage (Google, Bing, Baidu, DuckDuckGo, etc.), operators and time filters |
| community | `last30days` | WebSearch with date filters | What people said in the last 30 days (Reddit, X, YouTube, HN, GitHub) |
| platforms | `agent-reach` | `web-access` CDP | Platforms the web lane reads badly: YouTube transcripts, Xiaohongshu, Bilibili, V2EX, GitHub issues, Korean/Chinese communities |
| browser | `web-access` | none; report `needs setup: Chrome CDP` | Login-gated, JS-rendered, anti-bot pages; local Chrome history/bookmarks; video frames |
| academic | Consensus / PubMed MCP tools (find them with ToolSearch "consensus" / "pubmed") | WebSearch on scholar sources | Peer-reviewed evidence, validity studies |
| codebase | `Explore` subagent on the named path | `rg` + `Read` directly | What the project already has, what is missing, exact files/functions |

Lane selection:

- Always run `web`, `search`, `community` and `academic` unless the question clearly cannot use them.
- Run `platforms` when social, video or community sources matter.
- Run `browser` only when the question needs login-gated or JS-heavy sources, or after another lane returns a central `UNREAD` URL. Do not pay the browser cost on every run.
- Run `codebase` only when the question names a local project or path.

`scrapling-official` is not a lane: when a lane reports a key page it could not read, use it in step 4.

Tell every lane: read-only. Existing Chrome login state may be used for reading, but no posting, account creation, purchases or new logins. If a page requires login, return `needs login`. If its skill needs setup or auth, stop that lane and report `needs setup: <what>` instead of walking the user through it.

Before the first CDP action in a session, show the user this warning once:

> 일부 사이트는 자동화 탐지가 엄격해 계정 정지 위험이 있습니다. 기존 Chrome 로그인 세션을 읽기 전용으로 사용하고, 새 로그인·게시·구매는 하지 않습니다.

If the runtime cannot pause for consent, skip CDP and use static fetch. If the user has already authorized browser use in this session, proceed.

`web-access`'s "all network operations must go through this skill" rule applies only to browser/CDP work. It does not replace WebSearch, `multi-search-engine`, Bright Data, `agent-reach` or MCP tools in the other lanes.

## 3. Lane return format

Ask each lane to return, under 400 words:

```
STATUS: ok | partial | needs setup: <what> | needs login | no results
FINDINGS:
- <claim> — <source URL or file:line> — <date if known>
GAPS: <what it looked for and could not find>
UNREAD: <URLs it found but could not open>
```

Short structured returns keep the merge step cheap and make it obvious which claims have sources.

## 4. Merge (inline)

1. For `UNREAD` URLs that look central, try `web-access` CDP first. If Chrome CDP is unavailable, try `scrapling-official` or Bright Data `fetch`. If all fail, leave the URL in `Not covered`.
2. Group claims across lanes. Mark each: **confirmed** (2+ independent lanes or a primary source), **single-source**, or **conflicting** (show both sides).
3. Drop claims with no source.
4. For codebase questions, map each outside requirement to "exists in repo (file:line)" or "missing".

## 5. Report

Answer first, then evidence. Structure:

```
## Answer
<3-8 lines: the direct answer / the build list>

## Details
<grouped findings, each with [lane] and source links; mark single-source and conflicting>

## For this codebase   (only with the codebase lane)
| Need | Status | Where / what to add |

## Lane status
| Lane | Status | Notes |

## Not covered
<gaps, setup needed, unread pages>
```

Save the report as Markdown where the project keeps research notes (follow the repo's convention, e.g. `docs/`); with no project, save it in the working directory. Then give the user the short Answer and the file path in chat.

## 6. Runtime notes

- **WorkBuddy:** `Skill` loads from `~/.workbuddy-ai/skills`; `Agent` supports `Explore` and `general-purpose` subagents. `web-access` and `multi-search-engine` are installed there.
- **Claude Code:** lane skills live under `~/.claude/skills`; the Agent tool supports parallel subagents. `web-access` is symlinked there.
- **Codex:** reads skills from `~/.agents/skills`. If the Skill tool is unavailable, read the lane's `SKILL.md` and follow it directly. If the Agent tool is unavailable, run lanes sequentially.
- **Tool name mapping:** `multi-search-engine`'s `web_fetch` means WebFetch in Claude Code/WorkBuddy, or `curl` in Codex. `web-access` uses the shell (`Bash`) to call its local CDP proxy.
- **web-access paths:** use `${CLAUDE_SKILL_DIR}` when it is set; otherwise use `~/.workbuddy-ai/skills/web-access` (or the symlinked `~/.agents/skills/web-access`). Run `node <skill-dir>/scripts/check-deps.mjs` before the first CDP action.
