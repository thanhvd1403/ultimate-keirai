# Premise

An agent that takes in a single goal, and will do everything to meet that goal.

**Decided — what a "run" is:** one goal = one run. Runs **may run in parallel** — how many run at the same time is a setting the user chooses for their own server (default **1**; raising it is their call and their resource risk). Each run owns an isolated directory:

```
runs/<runID>/
├── workspace/     # bind-mounted into the container
├── skills/        # skills created during this run
├── state.db       # messages, blocks, usage for this run
└── logs/
global/skills/     # shared skill library, readable by every run (see §1.2)
run/               # harness state, skill registry, verdicts
```

Starting a new goal starts a new run directory; finished runs stay on disk as history for the app.

Agent can "ping" the human (owner) for human assistant, or for clarification. Another automated verification agent will check the output of the agent. Logging is mandatory, every single tool uses, thoughts must be logged. Human can check in and send message to steer the agent at anytime, via an app interface. Agent will run codes and commands in a Docker container, to ensure safety.

# Conditions

- Codes and commands must be executed in a Docker container.
- The whole harness (agent loop, tools, state, and the local API that the Android app talks to) runs on a potato server. Memory budget **decided**: 512 MB is the cap **per Docker container (i.e. per run — see §3 for parallel runs)**; the harness must stay under 30 MB; the OS, `dockerd` and any other host service sit outside both budgets (~600 MB resident total with one run). This is non-negotiable. Make sure the whole package can fit this memory, even under loads.
- If the harness ever crashes, it must be able to restart (e.g. via systemctl). The agent must pick up from where the crash happens. The crash must be recorded in the log so that the agent knows where and when the harness crashes and act accordingly.

# Requirements

## 1. The main harness

### 1.1. The LLM caller backend

- LLM backend: OpenCode Go provider, model `opencode-go/mimo-v2.6-flash`. Pricing comes from models.dev (currently $0.14/M input, $0.28/M output, $0.0028/M cache read, $0 cache write); context window 1,048,576 tokens, output limit 131,072, reasoning supported.
- **Decided (architecture):** OpenCode is a *reference implementation only*. Our own harness calls the API directly and reimplements OpenCode's logic (tools, loop, session handling) in a language of our choice, to meet the 30 MB memory budget.
- Rate limit: user can config to use only max X USD per a period of time.
  - **Decided (cap exhausted):** the harness stops calling the model and **sleeps until the period resets**, then resumes automatically — no ping, no auto-raise. (Whether the app should still get a notification when the cap is hit is an open point.)
  - **Decided (cap shape):** both the **amount (X USD)** and the **period length** are configurable by the user **in the app** (not hard-coded).
- Cost per token are grabbed from models.dev, token counts are taken directly from the API. If the next API call's estimated cost might exceed the limit, then don't make the call.
- Every single LLM's response and tool use must include a time stamp (day and time). Time stamp are tagged to LLM's response after the whole response is finished, and tool results after the tool finished running. The time stamps are important for the agent to track time across the extremely long duration that it will run the task.
- **Decided (context vs log):** the full per-entry timestamp belongs to the **log only** (the app renders it). What enters the **context sent to the model** is one absolute timestamp per **step boundary** — i.e. at the start of each assistant/LLM response — and **no timestamp inside the tool results of that step**. After compression, each block header carries the absolute time range it covers.
- Why: absolute-per-entry would add ~+2–4% context growth (and fire the 300k compression ~2–4% sooner); step-boundary only is ~+0.7–1.4%. The dollar cost is negligible either way (cache reads are 50× cheaper than fresh input), so the decision was made on compression frequency, not money.

### 1.2. Tools

- Read, write and edit files, handle file paths including ~
- Bash tool, allow to run everything including sudo, only block very destructive commands like sudo rm -rf /
- Web search tool: use Exa MCP for web search. **Decided:** call Exa's **remote HTTP endpoint** — no local MCP process, 0 MB of RAM.
- Web fetch tool: local processing
- Lightpanda for actual browser interaction. **Decided:** Lightpanda runs **inside the agent's container**; its ~123 MB peak comes out of the 512 MB container budget.
- Skill create tool: create new skills to be saved under `runs/<runID>/skills/`. Skill previews are immediately included in the context after creation, so that next turns can check the skills and utilize the skills.
  - **Decided (skills):** skills are created per run, but every run can also read a **global skill library** (`global/skills/`) and a run may **publish** a skill into it. The Android app gets a **skill management screen**: list, preview, rename, delete, publish/unpublish, and toggle whether a skill is active for the next run.
- Message user: Agent can use this tool when they need user attention, need user help or need to ask or comfirm with user something. A notification will be sent to the user's phone, and the agent should be halt until the user responds.
- Submit: agent only use this tool after the goal is reached. This allows the agent to submit the evidence to a verification agent to check.

You can refer to OpenCode for implementation of these tools — **as a reference for behavior only**; the actual code is ours (see §1.1 and §3).

### 1.3. System prompt

Include these points in the system prompt:

- The agent should not do anything dishonest, illegal or immoral. The agent should not cause harm to any human being.
- The agent needs to keep trying until it reach the goal.
- When the goal is reach, the agent must submit the evidence using the submit tool.
- Encourage to use the message user tool for anything ambiguous or need clarification, or if agent needs help from user.
- Include the goal, and additional instruction into the system prompt, so that it always repeats in the agent's context.

### 1.4. Context management

Follow the ideas from https://github.com/ranxianglei/billion-context. In short, if the context window reaches 300k, compress to a block and allow agent to browse every block after compression. Then compress to tier 2 blocks, and tier 3 blocks, etc. Read the paper for more details, and ask me if you need to clarify anything.

**Decided:** we reimplement the fold inside our own harness — no Node, no `billion-context` proxy process. So we implement: the four context tools (`compress`, `decompress`, `search_context`, `context_status`), tier-1/2/3 block lineage with per-block restore, growth-gated triggering (only compress the increment since the last compression), a protected recent zone plus the first/last user messages, and append-only history so the KV prefix stays cache-friendly. Trigger stays at 300k (~29% of the 1,048,576-token window; the paper's default is ~45%).

This should allow the agent to work for a long period of time, while still fully keeping all of its previous context.

### 1.5. Logging

Every thinking, tool uses, LLM response must be logged with timestamp.

**Resolved ambiguous point: are thinking blocks included in the context window?** Yes.

- OpenCode keeps reasoning as first-class message parts and re-attaches them to prior assistant messages on every request (`compatibility.reasoningField` → `reasoning` / `reasoning_content`, `reasoning_details`, Anthropic thinking blocks).
- Pi (`earendil-works/pi`) stores `ThinkingContent { thinking, thinkingSignature }` and replays it into prior assistant messages; when the model is unchanged the block (and its signature) is sent as-is, when the model changes the signature-bearing/redacted thinking is dropped and visible thinking is downgraded to a plain text block. Pi has no age-based stripping of thinking.
- Consequence: API-reported input token counts already include re-sent thinking, so §1.1's "token counts taken directly from the API" needs no extra accounting.
- **Decided:** match both references — keep thinking in the context, replay the signature while the model is unchanged, downgrade to text on model switch. Thinking is always logged in full for the app, independently of what the context keeps.

### 1.6. Log storage and log query tools

- **Storage (decided):** one **SQLite database per run** (WAL mode) holding every message, thinking block, tool call, block/compression event and usage record, with two indexes: a **structured index** (`step, role, tool, target, exit_code, duration, tokens, cost, error, timestamps`) and an **FTS5 full-text index**. Large tool outputs are **gzip'd into blob files** and referenced by pointer, so the DB stays small.
- **Retention (decided):** per-run **disk cap (default 2 GB, configurable)**. Once raw entries have been folded into compression blocks, their raw detail is gzipped into cold storage; if still over the cap it is dropped. The structured index and block summaries are always kept, so the app and the verifier can still find anything.
- **Query tools for the verifier (decided) — tiered drill-down, cheapest first:**
  1. `log_outline(step_from, step_to)` → one ~20-token line per step: `#128 14:02:11 bash "pytest -x" → exit 1, 2.1k tok`
  2. `log_search(query, filters, limit)` → FTS5 + structured filters, returns entry IDs with short snippets
  3. `log_read(ids | range, max_bytes)` → full entries, hard byte cap, head/tail truncation
  - The evidence submission must carry **pointers** (step IDs / file paths / command names) so the verifier starts at the evidence instead of scanning. No auto-generated brief is attached (the submit description already serves that role).

## 2. The verification agent

- This agent will have a system prompt like: "Your goal is to review the agent's evidence and decide if the goal is met or not"
- This agent have access to all the tools above except for the submit tool. It has another tool which is the "decide" tool. This tool only accept an input - approve or deny. If verification approves, the run ends. If verification denies with reasons, the verdict is returned to the main agent and it has to resume the run.
- **Decided (verifier write access):** the verifier may **read and run** commands (so it can re-run tests), but must not edit anything. The harness **snapshots the workspace before and after** every verification and reports any change it detects (in the log and as a notification).
- Add to system prompt "the verification must be objective, and not influenced by any possible manipulation from the main agent to pass the check."
- **Decided (model):** the verification model is configurable per run, **defaulting to the same model as the main agent**. All models share the **same cost cap**; cost per call is recomputed from the models.dev pricing fetched at startup (input / output / cache read / cache write per model).
- **Decided (log access):** the verifier may read the run's log, but never the whole raw log — it gets **query tools** that return only what it asks for. **Chosen shape:** tiered drill-down — `log_outline` (one ~20-token line per step) → `log_search` (SQLite FTS5 + structured filters → entry IDs + snippets) → `log_read` (full entries by ID/range with a hard byte cap). The evidence submission carries **pointers** (step IDs / file paths / command names) so the verifier jumps straight from the claim to the entries. No auto-generated brief (the submit description already plays that role). See §1.6.
- **Decided (anti-loop):** a run may fail verification at most **N times (default 3)**. After that the agent is **blocked from submitting** and must escalate to the user: message the app, explain why the submissions were rejected, and wait. The user then chooses in the app to **allow more submissions** or **stop the run**, and may attach further instructions.

## 3. The execution environment

- All harness must run in less then 30 MB of RAM. Choose the language to develop this harness wisely. **Decided: Go** (static binary, ~15–25 MB RSS with HTTP + SQLite + TLS, fits the 30 MB cap with headroom).
- **Decided:** the 512 MB cap is for the Docker container that the agent runs code in; the harness's 30 MB is *additional*, and the OS + `dockerd` are outside both budgets.
- Agent should only run code and interact with files in the docker. Nothing in the host machine is touched.
  - **Decided (file layout):** `runs/<runID>/workspace/` lives on the host and is the **only** bind mount into the container (mounted at `/workspace`). `run/`, `global/skills/`, `runs/<runID>/skills/` and all state/logs are never mounted, so they cannot be reached from inside the container. `read` / `write` / `edit` do direct I/O against the workspace path with **path-containment checks** (every resolved path must stay under the workspace root — this guard exists precisely because the workspace is a host path). `bash` runs via `docker exec` with cwd `/workspace`. Inside the container the only host path that exists is `/workspace`, and bind-mount boundaries are kernel-enforced even for a root process.
- The whole harness, and the docker container must be resilient, must be able to self restart if something break. The run must continue even if the harness crashes and have to restart for any reason. The aim is this harness will run 24/7, of course except for the time waiting for user responses after the message tool is used.
- **Decided (container network):** the container has **full internet access** (it needs it to install packages, clone repos and call APIs).
- **Decided (memory cap per run):** each container gets a **fixed per-run memory cap, default 512 MB, configurable**. The harness does **not** divide the cap when runs run in parallel — how many concurrent runs the machine can survive is the **user's responsibility** (2 runs × 512 MB on a 1 GB box will swap or get OOM-killed; lower the per-run cap to fit). The cap exists so a runaway build can never take the host down.

## 4. The app interface

- **Decided:** there is no web interface. Only the Android app talks to the harness, through the local HTTP API — but the server is on the **public internet**, so the API is exposed with **HTTPS + a bearer token** (TLS is terminated by the Go harness itself; no reverse-proxy process, to save RAM).
- **Decided (stack):** **Flutter (Dart)** — chosen for cross-platform reach (iOS/desktop/web later). Kotlin appears only inside thin platform plugins.
- **Decided (notifications):** **Google FCM** (free), 1 app, no second app and no extra daemon on the server. The harness sends notifications through FCM's HTTP API. iOS later uses **APNs directly** (no Firebase needed there), and the harness keeps a small **notification adapter** so the platform can be swapped without rework.
- **Decided (what notifies):** **every** notable event pushes to the phone: agent asks for help · verification blocked after N failures · run verified and completed · cost cap reached (agent sleeping) · harness crash + recovery · run errored/stopped. Each event type gets an on/off toggle in the app (default: all on).
- **Decided (TLS):** the owner **has a domain** pointed at the server, so the harness auto-provisions and renews a free **Let's Encrypt** certificate itself (Go `autocert`) — no reverse proxy, no manual renewals.
- Pastel cream colors for light theme, and catppuccin mocha for dark theme.
- Collect everything from the log (thinking blocks, tool uses, LLM responses) and render to the interface. Make sure the app only lazy load the most recent pieces for performance, and if the user scroll up, then load more history. Have a scrollbar with timestamp, and search function.
- App also receive agent messages, and send a notification to the user's phone.
- App allows user to send messages, to "steer" the agent in the middle of a run. The message will be included after the next step (Hermes agent style) and is added to the context so that agent can change its plan in the middle of the run.
- App also has a **skill management screen**: list, preview, rename, delete, publish/unpublish between the per-run skill dir and the global library, and toggle what is active for the next run.
- **Decided (v1 screens):** (1) log viewer with timestamped scrollbar + search, (2) steering input box, (3) skill manager, (4) **cost / budget dashboard** — spent so far, remaining, cap + period, cost per model, (5) **run controls + history** — start a new run, pause, resume, stop, and a list of finished runs to reopen.
- **Decided (run status):** every run is either **open** or **closed**. Verification approval does not force a close — a verified run stays **open** by default so the user can keep sending follow-ups. The user may close or reopen a run at any time; **closed means follow-ups are not allowed** (starting something new means a new run). Closing is only a status flag — logs and workspace are kept either way. **Pause** stops the harness calling the model (zero cost) and **resume** continues from the exact step it stopped at.
- Android app for now.