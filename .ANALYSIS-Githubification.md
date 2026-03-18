# Githubification Analysis — PicoClaw

### How this repository could become a GitHub Action based mechanism

---

## Executive Summary

PicoClaw is an ultra-lightweight personal AI assistant written in Go, designed for $10 hardware with <10MB RAM. It already supports 14 chat platform adapters (Telegram, Discord, Slack, WhatsApp, etc.) through a clean channel-agnostic message bus architecture. This analysis outlines how PicoClaw can be **Githubified** — converted from software that must be cloned and run locally into software that executes directly on GitHub via GitHub Actions, using GitHub Issues as its user interface.

**Recommended Strategy: Channel Addition (Strategy 5)** — GitHub becomes the 15th channel adapter.

**Fallback Strategy: Wrapping (Strategy 2)** — if the agent binary should remain unmodified.

---

## What is Githubification?

Githubification is the act of converting a repository into **GitHub-as-infrastructure**. Instead of cloning the repo and running the software elsewhere, the repo becomes something that **runs on GitHub itself** via GitHub Actions.

Four GitHub primitives serve four universal roles:

| GitHub Primitive | Role |
|---|---|
| **GitHub Actions** | Compute — the runner that executes the agent |
| **Git** | Storage and memory — sessions, conversations, state are committed |
| **GitHub Issues** | User interface — each issue is a conversation thread |
| **GitHub Secrets** | Credential store — LLM API keys, tokens |

This mapping is invariant across all Githubified repositories, from single-dependency native agents to compiled Rust binaries with zero runtime dependencies.

---

## PicoClaw — Current Architecture

### What PicoClaw Is

| Characteristic | Detail |
|---|---|
| **Language** | Go 1.25+ |
| **Binary** | Single static binary, CGO_ENABLED=0 |
| **Size** | <10MB RAM footprint |
| **Platforms** | linux/amd64, arm64, riscv64, loong64, arm, darwin, windows, freebsd |
| **Channels** | 14 chat adapters (Telegram, Discord, Slack, WhatsApp, Feishu, DingTalk, QQ, LINE, WeChat, OneBot, CLI, MaiXCam, Pico, Email) |
| **LLM Providers** | OpenAI, Anthropic, Gemini, DeepSeek, Ollama, Zhipu, Cerebras, OpenRouter |
| **Tools** | File operations, web search, shell exec, MCP protocol, cron, skills |
| **Session Management** | JSON-based sessions in workspace/sessions/ |
| **Memory** | MEMORY.md (long-term) + daily notes (YYYYMM/YYYYMMDD.md) |
| **Skills** | Marketplace-discoverable, installable into workspace |
| **CLI Agent Mode** | `picoclaw agent --message "hello" --session "cli:default"` |

### Architecture Diagram

```
User Input (CLI, Telegram, Discord, etc.)
    ↓
Channel Adapter (validates, shows typing indicator)
    ↓
Message Bus (InboundMessage)
    ↓
Agent Loop (processMessage)
    ├─ Route to agent (AgentRegistry)
    ├─ Load session history (SessionManager)
    ├─ Build system prompt (workspace: AGENTS.md, IDENTITY.md, SOUL.md, USER.md)
    ├─ Call LLM with tools (OpenAI, Anthropic, etc.)
    ├─ Tool execution loop (max 20 iterations)
    │   ├─ File operations
    │   ├─ Web search/fetch
    │   ├─ Shell execution
    │   ├─ MCP tools
    │   └─ Skill invocations
    ├─ Save session (SessionManager)
    └─ Generate response
    ↓
Message Bus (OutboundMessage)
    ↓
Channel Adapter (format + send to platform)
    ↓
User Output
```

### Key Architectural Strengths for Githubification

1. **Clean channel separation**: The `Channel` interface and message bus decouple input/output from agent logic. Adding GitHub Issues as a channel requires no changes to the agent core.

2. **Static binary with zero runtime dependencies**: `CGO_ENABLED=0` produces a fully static binary. No npm, no pip, no Docker required on the Actions runner.

3. **CLI agent mode already exists**: `picoclaw agent --message "text"` accepts a message and returns a response — the exact invocation model for a GitHub Actions step.

4. **Session persistence via filesystem**: Sessions are JSON files in the workspace directory. Git can version-control these directly.

5. **Workspace-based memory**: MEMORY.md, daily notes, and AGENTS.md are plain files that map naturally to git-committed state.

6. **Multi-platform cross-compilation**: GoReleaser already builds for linux/amd64 (the Actions runner platform). Pre-built binaries are available via GitHub Releases.

---

## Githubification Strategy Assessment

### Strategy Selection (Decision Tree)

```
Does the agent exist yet?
└── Yes (PicoClaw is a mature agent)
    └── Can it run on GitHub Actions?
        └── Yes (static Go binary, no persistent server required for CLI mode)
            └── Does it have a multi-channel/adapter architecture?
                └── Yes (14 channels via message bus)
                    └── Strategy 5: Channel Addition ✅
```

### Why Channel Addition (Strategy 5)

PicoClaw's architecture is **structurally identical** to MicroClaw's (the Githubification case study for channel addition):

| Attribute | MicroClaw (Rust) | PicoClaw (Go) |
|---|---|---|
| Language | Rust (compiled) | Go (compiled) |
| Runtime dependencies | Zero | Zero (CGO_ENABLED=0) |
| Binary delivery | Pre-built from releases | Pre-built from releases |
| Channel adapters | 14 platforms | 14 platforms |
| Channel separation | Clean adapter pattern | Clean Channel interface + message bus |
| CLI invocation | Subcommand | `picoclaw agent --message` |
| Actions runner install | Download binary | Download binary |
| Estimated new code | ~500 lines (Rust adapter) | ~500-800 lines (Go adapter + workflow) |

The lesson from MicroClaw: **When the agent IS the runtime, Githubification becomes a channel adapter problem.**

PicoClaw IS the runtime. A single binary that accepts input, processes it through an LLM, and returns output. GitHub Actions merely provides the trigger and the environment.

### Alternative Strategies Considered

| Strategy | Viability | Reason |
|---|---|---|
| **Native (Strategy 1)** | ❌ Not applicable | PicoClaw already exists; Native is for agents designed from scratch for GitHub |
| **Wrapping (Strategy 2)** | ✅ Viable fallback | If the binary should remain completely unmodified, a TypeScript/Bun lifecycle wrapper (like GMI's `agent.ts`) could orchestrate PicoClaw from outside |
| **Substitution (Strategy 3)** | ⚠️ Unnecessary | PicoClaw CAN run on Actions — no need to substitute a different agent |
| **Transformation (Strategy 4)** | ❌ Overkill | PicoClaw is a single agent, not a multi-agent system requiring routing |
| **Channel Addition (Strategy 5)** | ✅✅ Best fit | Clean channel architecture makes this the natural and lowest-friction path |

---

## Implementation Plan

### Phase 1 — Minimal Viable Githubification (MVP)

**Goal**: PicoClaw responds to GitHub Issues using its existing `picoclaw agent` CLI mode, without modifying any Go source code.

#### 1.1 GitHub Actions Workflow

Create `.github/workflows/picoclaw-agent.yml`:

```yaml
name: PicoClaw Agent
on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]

permissions:
  contents: write
  issues: write

concurrency:
  group: picoclaw-issue-${{ github.event.issue.number }}
  cancel-in-progress: false

jobs:
  authorize:
    runs-on: ubuntu-latest
    if: >-
      !github.event.issue.pull_request &&
      github.actor != 'github-actions[bot]'
    outputs:
      allowed: ${{ steps.check.outputs.allowed }}
    steps:
      - id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PERM=$(gh api repos/${{ github.repository }}/collaborators/${{ github.actor }}/permission \
            --jq '.permission')
          if [[ "$PERM" == "admin" || "$PERM" == "maintain" || "$PERM" == "write" ]]; then
            echo "allowed=true" >> "$GITHUB_OUTPUT"
          else
            echo "allowed=false" >> "$GITHUB_OUTPUT"
          fi

  reject:
    needs: authorize
    if: needs.authorize.outputs.allowed != 'true'
    runs-on: ubuntu-latest
    steps:
      - run: |
          gh api repos/${{ github.repository }}/issues/${{ github.event.issue.number }}/reactions \
            -f content='-1'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  respond:
    needs: authorize
    if: needs.authorize.outputs.allowed == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Indicate processing
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          COMMENT_ID: ${{ github.event.comment.id }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          if [ -n "$COMMENT_ID" ]; then
            gh api "repos/${{ github.repository }}/issues/comments/${COMMENT_ID}/reactions" \
              -f content='rocket' || true
          else
            gh api "repos/${{ github.repository }}/issues/${ISSUE_NUMBER}/reactions" \
              -f content='rocket' || true
          fi

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Download PicoClaw binary
        run: |
          # Download latest release binary for linux/amd64
          RELEASE_URL=$(gh api repos/${{ github.repository }}/releases/latest \
            --jq '.assets[] | select(.name | endswith("linux-amd64.tar.gz")) | .browser_download_url')
          curl -sL "$RELEASE_URL" -o picoclaw.tar.gz
          tar xzf picoclaw.tar.gz
          chmod +x picoclaw
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract message
        id: message
        env:
          EVENT_NAME: ${{ github.event_name }}
          COMMENT_BODY: ${{ github.event.comment.body }}
          ISSUE_TITLE: ${{ github.event.issue.title }}
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: |
          if [ "$EVENT_NAME" = "issue_comment" ]; then
            printf '%s' "$COMMENT_BODY" > /tmp/picoclaw-message.txt
          else
            printf '%s: %s' "$ISSUE_TITLE" "$ISSUE_BODY" > /tmp/picoclaw-message.txt
          fi

      - name: Run PicoClaw agent
        id: agent
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          SESSION_KEY="github:issue-${{ github.event.issue.number }}"
          ./picoclaw agent \
            --message "$(cat /tmp/picoclaw-message.txt)" \
            --session "$SESSION_KEY" > /tmp/picoclaw-response.txt 2>/dev/null

      - name: Post response as comment
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          gh issue comment "$ISSUE_NUMBER" \
            --repo "${{ github.repository }}" \
            --body-file /tmp/picoclaw-response.txt

      - name: Persist session state
        run: |
          git config user.name "picoclaw[bot]"
          git config user.email "picoclaw[bot]@users.noreply.github.com"
          git add -A .picoclaw-state/
          if ! git diff --cached --quiet; then
            git commit -m "state: update session for issue #${{ github.event.issue.number }}"
            git pull --rebase origin "${{ github.ref_name }}" || true
            git push
          fi
```

#### 1.2 State Directory

Create `.picoclaw-state/` in the repo root to hold git-committed session state:

```
.picoclaw-state/
├── sessions/           # Session JSON files (one per issue)
├── memory/
│   ├── MEMORY.md       # Persistent memory across all issues
│   └── YYYYMM/        # Daily notes
└── workspace/
    ├── AGENTS.md       # Agent personality (repo-specific)
    ├── IDENTITY.md     # Agent identity
    ├── SOUL.md         # Values and personality
    └── USER.md         # User context
```

#### 1.3 Configuration

Place a minimal `config.json` in `.picoclaw-state/` that:
- Sets workspace path to `.picoclaw-state/workspace/`
- Points session storage to `.picoclaw-state/sessions/`
- Enables only the LLM provider(s) configured via secrets
- Disables all chat channels (GitHub Actions workflow handles I/O)
- Optionally enables MCP tools for file operations within the repo

#### 1.4 Git Attributes

Add to `.gitattributes`:
```
.picoclaw-state/memory/MEMORY.md merge=union
```

This ensures concurrent agent runs append to rather than conflict on the memory file. **Note**: `merge=union` concatenates all conflicting lines without deduplication or ordering guarantees. For structured memory content, a custom merge driver or post-merge deduplication script may be needed in production.

### Phase 2 — Native GitHub Channel Adapter

**Goal**: Add a `github` channel to PicoClaw's codebase, making GitHub Issues a first-class platform alongside Telegram, Discord, etc.

#### 2.1 New Package: `pkg/channels/github/`

Implement the `Channel` interface for GitHub:

```go
// pkg/channels/github/github.go
type GitHubChannel struct {
    owner      string
    repo       string
    token      string
    httpClient *http.Client
}

func (c *GitHubChannel) Name() string { return "github" }

func (c *GitHubChannel) Send(ctx context.Context, msg bus.OutboundMessage) error {
    // POST /repos/{owner}/{repo}/issues/{number}/comments
    // with msg.Content as the comment body
}
```

This adapter would:
- Read issue/comment events from environment variables (set by the workflow)
- Convert GitHub issue comments to `InboundMessage` structs
- Convert agent responses to GitHub issue comment API calls
- Support reactions (🚀 for processing, 👍 for success, 👎 for error)
- Map issue numbers to session keys (`github:issue-{number}`)

#### 2.2 CLI Subcommand: `picoclaw github`

Add a `github` subcommand that:
1. Reads `GITHUB_EVENT_PATH` (the JSON event payload from Actions)
2. Extracts the issue number and comment body
3. Invokes the agent loop with the GitHub channel
4. Posts the response back to the issue
5. Commits session state to git

This is equivalent to `picoclaw agent --message` but GitHub-aware.

#### 2.3 Configuration Extension

Add GitHub channel to `config.json`:

```json
{
  "channels": {
    "github": {
      "enabled": true,
      "owner": "japer-technology",
      "repo": "githubification-picoclaw",
      "session_path": ".picoclaw-state/sessions/"
    }
  }
}
```

### Phase 3 — Full Lifecycle Pipeline

**Goal**: Implement the complete Githubification lifecycle with security, memory, and operational maturity.

#### 3.1 Lifecycle Pipeline

| Step | Implementation |
|---|---|
| **Guard** | Workflow authorization step (check collaborator permission) |
| **Indicate** | Add 🚀 reaction to the triggering issue/comment |
| **Execute** | Run `picoclaw github` with event payload |
| **Commit** | Git push session state, memory updates, daily notes |

#### 3.2 Fail-Closed Security

Two options (both are compatible):

| Mechanism | Implementation |
|---|---|
| **Workflow authorization** | GitHub API permission check (admin/maintain/write only) |
| **Sentinel file** | `.picoclaw-state/ENABLED.md` must exist; workflow checks before proceeding |

#### 3.3 Session Persistence via Git

After each agent execution:
1. Session JSON is saved to `.picoclaw-state/sessions/github_issue-{N}.json`
2. Memory updates go to `.picoclaw-state/memory/MEMORY.md`
3. Daily notes append to `.picoclaw-state/memory/YYYYMM/YYYYMMDD.md`
4. All changes are committed and pushed with `git push`
5. Use `merge=union` on memory files to handle concurrency

#### 3.4 Workspace Identity

Customize the workspace files for the GitHub persona:

**IDENTITY.md** — Define who PicoClaw is in this repository context  
**AGENTS.md** — Agent instructions for handling GitHub issues  
**SOUL.md** — Personality traits for issue interactions  
**USER.md** — Context about the repository and its community

---

## Mapping to GitHub Primitives

| GitHub Primitive | PicoClaw Mapping |
|---|---|
| **GitHub Actions** | Runs `picoclaw agent --message` (or `picoclaw github`) on ubuntu-latest. Binary downloaded from releases — no build step needed. |
| **Git** | Session files, MEMORY.md, daily notes, and workspace config committed to `.picoclaw-state/`. Full conversation history is version-controlled. |
| **GitHub Issues** | Each issue maps to a PicoClaw session (`github:issue-{N}`). Issue comments become InboundMessages; agent responses become issue comments. |
| **GitHub Secrets** | `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `BRAVE_SEARCH_API_KEY`, etc. |

---

## Challenges and Mitigations

### Challenge 1: Binary Size and Download Time

PicoClaw's static binary is larger than a single `node_modules` dependency but still small by Go standards. The Actions runner downloads it once per workflow run.

**Mitigation**: Cache the binary using `actions/cache@v4` keyed by version tag. Alternatively, build from source in the workflow (Go setup + `make build` takes ~60 seconds).

### Challenge 2: Session State Concurrency

If two issues trigger simultaneously, both workflows may try to push session state, causing git conflicts.

**Mitigation**: 
- Use per-issue concurrency groups (`concurrency: picoclaw-issue-${{ github.event.issue.number }}`)
- Use `merge=union` on memory files
- Implement git pull + rebase before push in the commit step

### Challenge 3: Workspace Configuration

PicoClaw expects `~/.picoclaw/config.json` by default, but the Actions runner has no pre-configured workspace.

**Mitigation**: 
- Set `PICOCLAW_CONFIG` or `PICOCLAW_WORKSPACE` environment variables to point to `.picoclaw-state/`
- Or use CLI flags: `--config .picoclaw-state/config.json --workspace .picoclaw-state/workspace/`

### Challenge 4: Tool Execution Security

PicoClaw's `exec` tool allows arbitrary shell execution, which is powerful but dangerous in a public repository context.

**Mitigation**:
- Disable the `exec` tool in the GitHub configuration
- Enable only safe tools: file read/write (sandboxed to repo), web search, MCP servers
- Use PicoClaw's `restrict_to_workspace: true` setting to sandbox file operations

### Challenge 5: Gateway Mode Not Applicable

PicoClaw's `gateway` command runs a long-lived process for real-time chat platforms. This is incompatible with Actions' ephemeral model.

**Mitigation**: Not needed. The `agent` command's single-shot execution model is the correct fit for Actions. The gateway is for Telegram/Discord; the agent CLI is for GitHub.

---

## Comparison with Other Githubified Repos

| Attribute | GMI (Native) | OpenClaw (Wrapping) | MicroClaw (Channel) | **PicoClaw (Channel)** |
|---|---|---|---|---|
| Strategy | Native | Wrapping | Channel Addition | **Channel Addition** |
| Language | TypeScript | Python/TS | Rust | **Go** |
| Runtime deps | 1 (pi-coding-agent) | 30+ (Python) | 0 (compiled) | **0 (compiled, CGO=0)** |
| Binary delivery | npm install | npm + pip install | Download release | **Download release** |
| New code | 0 (IS the agent) | ~2000 lines (wrapper) | ~500 lines (adapter) | **~500-800 lines (adapter)** |
| Channel architecture | N/A | N/A | 14 adapters | **14 adapters** |
| Session format | JSONL | JSONL | — | **JSON** |
| Memory | Git commits | Git commits | — | **MEMORY.md + daily notes** |
| Lifecycle steps | 3 | 5 | 3 | **3-4** |

PicoClaw sits closest to MicroClaw on the Githubification spectrum. Both are compiled binaries with channel-agnostic architectures where Githubification is an adapter problem, not an orchestration problem.

---

## Estimated Effort

| Phase | Scope | Effort | Dependencies |
|---|---|---|---|
| **Phase 1 (MVP)** | Workflow + state dir + config | 1-2 days | GitHub Secrets configured |
| **Phase 2 (Channel)** | Go channel adapter + CLI subcommand | 3-5 days | Phase 1 validated |
| **Phase 3 (Lifecycle)** | Security + memory + operational maturity | 2-3 days | Phase 2 merged |
| **Total** | Full Githubification | **~1-2 weeks** | |

Phase 1 requires **zero changes** to PicoClaw's Go source code. The entire MVP is a workflow file, a state directory, and a configuration file. This means Githubification can begin immediately, validated incrementally, and refined over time.

---

## Recommendation

**Start with Phase 1.** Ship the GitHub Actions workflow that downloads PicoClaw's pre-built binary and invokes `picoclaw agent --message` on issue events. This proves the concept with zero Go code changes.

Then iterate to Phase 2 — adding GitHub as a native channel adapter — when the limitations of the CLI-based approach become apparent (e.g., needing reactions, typing indicators, or richer state management).

Phase 3 brings operational maturity: fail-closed security, persistent memory, and the full lifecycle pipeline that every production Githubification requires.

The core lesson from the Githubification playbook applies directly to PicoClaw:

> **When the agent IS the runtime, Githubification becomes a channel adapter problem.**

PicoClaw is the runtime. GitHub is the 15th channel.

---

## References

- [Githubification Framework](https://github.com/japer-technology/githubification) — strategies, case studies, and consolidated playbook
- [GitHub Minimum Intelligence](https://github.com/japer-technology/github-minimum-intelligence) — reference implementation of a GitHub-native AI agent
- [Githubification Lesson Consolidation](https://github.com/japer-technology/githubification/blob/main/.githubification/lesson-consolidation.md) — unified playbook from six case studies
- [Githubification Winners](https://github.com/japer-technology/githubification/blob/main/.githubification/winners.md) — ranked repos by Githubification suitability
