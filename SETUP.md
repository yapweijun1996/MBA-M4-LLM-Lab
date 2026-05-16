# SETUP.md — Local LLM Runtime for MBA-M4-LLM-Lab

> How to install and run the inference backend (MTPLX + Qwen3.6-27B) on
> a fanless **MacBook Air M4, 32 GB**. Calibrated and validated 2026-05-16.

Companion to [DESIGN.md](DESIGN.md). DESIGN explains *what* and *why*;
SETUP explains *how to bring the runtime online*.

---

## 1. Target hardware

This guide assumes the lab's primary machine:

| Spec | Value |
| --- | --- |
| Model | MacBook Air, `Mac16,12` |
| Chip | Apple M4 (base, **not** Pro / Max) |
| CPU | 10 cores (4P + 6E) |
| GPU | 10-core Apple GPU, Metal 4 |
| RAM | 32 GB unified memory |
| Storage | 460 GiB APFS (watch free space — Qwen 27B alone is 16.4 GB) |
| Cooling | **Fanless** |
| OS | macOS 26.4.1 (Tahoe) |
| Memory bandwidth | ~120 GB/s (the dominant constraint for decode speed) |

If you're on different hardware, the *commands* are the same but the
*throughput numbers* below will differ — bandwidth scales roughly linearly
with chip tier (M4 Pro ~273 GB/s, M4 Max ~546 GB/s).

---

## 2. Install MTPLX

### Why pipx, not pip

macOS 26 + Homebrew Python 3.14 enforces PEP 668. Running `pip install mtplx`
returns `error: externally-managed-environment` and refuses to install
system-wide. Do **not** add `--break-system-packages` — it can corrupt
Homebrew's Python.

Use `pipx`, which auto-creates an isolated venv per CLI tool:

```bash
# pipx should already be at /opt/homebrew/bin/pipx
pipx install mtplx
```

Verify:

```bash
mtplx --version    # → 0.3.6 (or later)
which mtplx        # → /Users/<you>/.local/bin/mtplx
which mtplx-tune   # → same dir; also callable as `mtplx tune`
```

If `mtplx` is not on PATH:

```bash
pipx ensurepath && source ~/.zshrc
```

### Why not Homebrew

`brew install youssofal/mtplx/mtplx` fails on macOS 26 because of the
Xcode 16.2 / Command Line Tools mismatch — Homebrew tries a source build
and the Xcode toolchain is incompatible with macOS 26's SDK. pipx bypasses
this entirely by using pre-built wheels.

---

## 3. Pull the model

```bash
mtplx pull Youssofal/Qwen3.6-27B-MTPLX-Optimized-Speed
```

This downloads ~16.4 GB to `~/.mtplx/models/Youssofal--Qwen3.6-27B-MTPLX-Optimized-Speed/`.
First-time download is bandwidth-bound; expect 10-30 minutes on a typical
home connection.

`runtime contract: true` in the output means the model's manifest embeds
rope_scaling, quantization scheme, and chat template — clients don't need
to configure those manually.

**Disk-space check before pulling more models**:

```bash
df -h /              # confirm at least 25 GB free before next big pull
```

---

## 4. Run the inference server

### 4.1 Daily-driver config (recommended)

For single-user local work on this machine:

```bash
mtplx serve \
  --model Youssofal/Qwen3.6-27B-MTPLX-Optimized-Speed \
  --profile performance-cold \
  --host 127.0.0.1 \
  --port 6767 \
  --reasoning on \
  --no-stats-footer \
  --open-browser
```

What each flag does:

| Flag | Why |
| --- | --- |
| `--profile performance-cold` | MTPLX server defaults to `sustained` which clamps GPU clock too aggressively on M4 base (measured 2.87 vs 7.27 tok/s). For burst / short-context use this is the right pick. **For prompts > 8K tokens, switch to `--profile sustained` despite the slower decode — the prefill fast path matters more for long inputs.** |
| `--host 127.0.0.1` | Loopback only. Skips API-key authentication automatically. The built-in chat UI needs this; see §6 for the why. |
| `--port 6767` | Avoids common conflicts: Vite dev server (5173), Ollama (11434), LM Studio (1234). Change if it conflicts with anything else you run. |
| `--reasoning on` | Enables Qwen3.6 thinking mode. Each response begins with a `<think>…</think>` block. Better for code / math / multi-step reasoning. Switch to `off` for terse chat. **Reasoning mode is set at server-launch time — per-request override does not work in 0.3.6.** |
| `--no-stats-footer` | Without this, MTPLX appends `⚡ MTPLX TPS: … tok/s …` to every assistant message *content*, which breaks downstream UIs. The same data remains available in the structured `mtplx_stats` field of the API response. |
| `--open-browser` | Auto-opens `http://127.0.0.1:6767/` after startup. |

Cold-load is ~12 seconds (8.2s model into MLX + 3.9s warmup). After
that the server keeps the model resident in unified memory.

### 4.2 LAN access (rarely needed)

If you need to reach the server from another device:

```bash
mtplx serve \
  --model Youssofal/Qwen3.6-27B-MTPLX-Optimized-Speed \
  --profile performance-cold \
  --host 0.0.0.0 \
  --port 6767 \
  --api-key sk-mtplx-$(openssl rand -hex 16) \
  --rate-limit 30 \
  --reasoning on \
  --no-stats-footer
```

**Caveat**: the built-in chat UI is unreachable in this mode because the
HTML page itself is gated by `Authorization: Bearer` and the UI has no
input field for the key. Workarounds:

- Chrome/Edge extension **ModHeader** — inject `Authorization` header
  automatically for `127.0.0.1:6767` / `<LAN IP>:6767`.
- Reverse proxy (Caddy / nginx) that runs its own login form, then adds
  the Bearer header upstream.
- Use the API directly from a programmatic client (curl, Python, etc.).

Never put the API key in a `git`-tracked file. Set it via env var if
launching from a script.

---

## 5. Measured throughput on this machine

Reference numbers for **Qwen3.6-27B-MTPLX-Optimized-Speed** on M4 base
(32 GB), 2026-05-16. All values are decode tok/s for short generation.

| Mode | Profile | MTP | tok/s | Verdict |
| --- | --- | --- | --- | --- |
| CLI `mtplx ask` | sustained | depth=3 | **2.87** | Anomalously low — avoid sustained on M4 base for short queries |
| CLI `mtplx ask` | performance-cold | off (AR) | **4.82** | Healthy AR baseline (~64 % of bandwidth ceiling) |
| CLI `mtplx ask` | performance-cold | depth=3 | **7.27** | Near AR ceiling (97 %) |
| Server `mtplx serve` | performance-cold | depth=3 | **10.1 – 11.7** | Warm-state best; ~1.5× CLI because no per-call cold load |

### 5.1 Why server beats CLI by ~1.5×

CLI re-pays the fixed cost of warmup each invocation (~4 seconds for a
16-token warmup pass). Server amortizes that cost across all requests.
**Rule**: any workflow that calls the model more than twice should hit
the server, not `mtplx ask`.

### 5.2 The memory-bandwidth ceiling

Decode is bandwidth-bound, not compute-bound. Theoretical AR ceiling:

```
tok/s_max ≈ memory_bandwidth_GBps / model_weights_GB
```

For Qwen 27B at ~5-bit quant (16.4 GB on disk) on M4 base (~120 GB/s):

```
ceiling = 120 / 16.4 ≈ 7.5 tok/s    (autoregressive single-token decode)
```

MTP / speculative decoding can exceed this because verification batches
re-use the same weight read for multiple tokens. The 11.7 tok/s
observed at MTP depth=3 ≈ 7.5 × 4 × 0.39 corresponds to ~39 % MTP
acceptance rate, typical for Qwen 27B on chat-style prompts.

**Implication for model choice**: if 7-11 tok/s feels slow, the bottleneck
is M4 base's bandwidth, not configuration. The two real escape routes are:

- Switch to a smaller model (14B → roughly 14-22 tok/s, 8B → 22-30 tok/s).
- Switch hardware (M4 Pro doubles bandwidth, M4 Max quadruples).

### 5.3 Thermal envelope

Fanless M4 Air sustains peak performance for ~15-20 minutes of continuous
inference, then thermally throttles by 15-25 %. Practical implication:
expect server throughput to settle from ~11 tok/s (cold burst) to
~6-8 tok/s (long session).

This is exactly the curve the lab project is designed to *measure* —
see DESIGN.md §1.2 research question 1.

---

## 6. Known gotchas

### 6.1 `mtplx tune` cannot run on fanless Macs (0.3.6)

`mtplx tune` requires "verified max-fan mode" via ThermalForge or TG Pro
before benchmarking. On a fanless MBA Air it's impossible to verify max-fan
mode (no fans), so tune always refuses. `--unsafe-force-unverified --yes`
does **not** bypass this check.

Workaround: skip tune entirely. MTPLX's default `mtp_depth=3` is already
at or near optimal on M4 base — measured 7.27 / 11.7 tok/s achieves
97 % of the AR bandwidth ceiling. The tuning choice space is only
{1, 2, 3}, so even worst-case misconfiguration costs ≤ 10 %.

### 6.2 Chat UI auth UX

The built-in chat UI HTML has no API-key input field. It expects the
`Authorization: Bearer` header to already be set, which a browser
cannot do on direct URL navigation. Use `--host 127.0.0.1` (no
API-key) for solo use — see §4.2 for LAN workarounds.

### 6.3 Reasoning mode is launch-time only

Per-request `"reasoning": "off"` in JSON body or `/no_think` in
system prompt are **ignored** in 0.3.6. If you need both modes
in one deployment, run two server instances on different ports
(e.g. 6767 with `--reasoning on`, 6768 with `--reasoning off`).

### 6.4 Stats footer pollutes message content

Without `--no-stats-footer`, every assistant message ends with
`⚡ MTPLX TPS: …`. Always include the flag in production.

---

## 7. Programmatic clients

The server speaks OpenAI v1. Model id is `mtplx-qwen36-27b-optimized-speed`
(derived from the HF name).

### 7.1 curl

```bash
curl http://127.0.0.1:6767/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mtplx-qwen36-27b-optimized-speed",
    "messages": [{"role":"user","content":"Hello"}],
    "max_tokens": 200
  }'
```

(no `Authorization` header needed in `--host 127.0.0.1` mode)

### 7.2 OpenAI Python SDK

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:6767/v1", api_key="dummy")

resp = client.chat.completions.create(
    model="mtplx-qwen36-27b-optimized-speed",
    messages=[{"role": "user", "content": "Hello"}],
    max_tokens=200,
)
print(resp.choices[0].message.content)
```

### 7.3 Browser / Vite frontend (this project)

```ts
const resp = await fetch("http://127.0.0.1:6767/v1/chat/completions", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "mtplx-qwen36-27b-optimized-speed",
    messages: [{ role: "user", content: prompt }],
    max_tokens: 200,
    stream: true,
  }),
});
// Parse SSE stream; each `data: {...}` line carries an OpenAI-format delta.
```

The structured `mtplx_stats` field on non-streaming responses contains
fine-grained perf counters (`forward_ar_hidden_calls`, `mtp_forward_calls`,
`prefill_chunk_size`, etc.) — useful for the thermal-decay analysis this
lab project is built around.

---

## 8. Health endpoints

| Path | Auth | Returns |
| --- | --- | --- |
| `GET /` | Required (or loopback) | Chat UI (HTML) |
| `GET /v1/models` | Required (or loopback) | OpenAI-format model list |
| `POST /v1/chat/completions` | Required (or loopback) | OpenAI-format completion |
| `POST /v1/messages` | Required (or loopback) | **Anthropic-format completion** (Messages API) |
| `GET /health` | Required (or loopback) | Health JSON |

The same server speaks **both OpenAI and Anthropic protocols** on the same
port — pick whichever endpoint your client library expects. See §9 for how
to point Anthropic-ecosystem tools (Claude Code, Anthropic SDK) at this server.

Note: even `/health` enforces auth when `--api-key` is set. This means
"is the server up?" probes from external monitoring tools need the key.

---

## 9. Anthropic / Claude Code integration

The MTPLX server speaks the Anthropic Messages API at `/v1/messages` on the
same port as the OpenAI endpoints. Any tool that uses the official Anthropic
SDK (Claude Code, custom Anthropic-SDK clients, OpenCode, etc.) can be
re-pointed at the local server via two environment variables.

### 9.1 Recommended server config for Claude Code

Claude Code does **long-context** work (multi-file reads, long conversations,
big diffs). For this workload use `--profile sustained` despite its slower
decode — the prefill fast path matters more for long inputs than burst tok/s.

```bash
mtplx serve \
  --model Youssofal/Qwen3.6-27B-MTPLX-Optimized-Speed \
  --profile sustained \
  --host 127.0.0.1 --port 6767 \
  --reasoning on \
  --no-stats-footer
```

### 9.2 Point Claude Code (or any Anthropic SDK client) at it

In the terminal you intend to run Claude Code from:

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:6767
export ANTHROPIC_API_KEY=dummy                                    # loopback skips auth, SDK needs non-empty
export ANTHROPIC_MODEL=mtplx-qwen36-27b-optimized-speed           # override default claude-* model name
export ANTHROPIC_SMALL_FAST_MODEL=mtplx-qwen36-27b-optimized-speed # same for haiku-class background tasks
claude                                                             # now talking to local Qwen, not Anthropic cloud
```

Why all four env vars are required:

- **`ANTHROPIC_BASE_URL`** — official Anthropic SDK redirect; every tool
  built on the Anthropic SDK respects it (Claude Code, `anthropic` Python
  package, TS SDK, etc).
- **`ANTHROPIC_API_KEY`** — loopback `--host 127.0.0.1` skips actual auth,
  but the SDK refuses to start with an empty key. Any non-empty placeholder
  works.
- **`ANTHROPIC_MODEL`** — Claude Code's default is `claude-opus-*` /
  `claude-sonnet-*`, which the MTPLX server's `/v1/models` does not list.
  Without this override Claude Code fails with `model missing` during
  startup. Set it to the model id MTPLX serves (`mtplx-qwen36-27b-optimized-speed`).
- **`ANTHROPIC_SMALL_FAST_MODEL`** — Claude Code internally runs background
  tasks (conversation compaction, quick tool routing) against a "small fast"
  model that defaults to a `claude-haiku-*` id. Same problem; point it at
  the same MTPLX model since you only have one local model loaded.

`mtplx-qwen36-27b-optimized-speed` is the **server's** model id, derived from
the HF model name. Confirm with: `curl -s http://127.0.0.1:6767/v1/models`.

**⚠️ Do NOT export these globally** (i.e. don't put them in `~/.zshrc`)
unless you are sure you want *every* Anthropic SDK call on your machine
to go to local Qwen 27B. Capability gap is large: cloud Claude (Opus 4.7,
Sonnet 4.6) vs local Qwen 27B differs by ~5-10× on tool calling, long-context
reasoning, and code quality. Keep this in a dedicated terminal session.

### 9.3 Verify the Anthropic endpoint

```bash
curl http://127.0.0.1:6767/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: dummy" \
  -d '{
    "model": "mtplx-qwen36-27b-optimized-speed",
    "max_tokens": 50,
    "messages": [{"role":"user","content":"Reply with exactly: PONG"}]
  }'
```

Expected response shape: `{"id":"msg_…", "type":"message", "role":"assistant",
"content":[{"type":"text","text":"…"}], "stop_reason":"…", "usage":{"input_tokens":N,"output_tokens":N}}`.

### 9.4 Real-world test result: Claude Code + local Qwen 27B is too slow

Test run 2026-05-16 on this exact hardware/software stack
(MBA M4, 32 GB, MTPLX 0.3.6, Qwen3.6-27B-MTPLX-Optimized-Speed, sustained
profile, `--reasoning on`):

- **Prompt**: a single `hi`
- **Response**: `嗨！有什么需要帮忙的？` (one Mandarin line)
- **Time**: **4 minutes 28 seconds** (server log `elapsed_s: 268.029911`)

Why so slow even for a one-word prompt — three compounding factors:

1. **Claude Code injects a massive system prompt.** Tool definitions,
   environment metadata, and the user's global `~/.claude/CLAUDE.md`
   are all sent on every turn. Estimated 15-25K tokens of prefill on
   the very first message. On M4 base (~120 GB/s bandwidth) prefilling
   ~20K tokens of a 16.4 GB model takes 3-4 minutes alone.
2. **Reasoning mode adds thinking tokens.** With `--reasoning on`, Qwen
   produces several hundred to thousand `<think>…</think>` tokens before
   the first user-visible output. Each token is ~0.1 s of decode in
   sustained mode.
3. **No streaming feedback during prefill.** During the 3-4 minute
   prefill phase the server emits `mtplx_stream_silence` events with
   `completion_tokens: 0`. Claude Code shows a `Wrangling…` spinner
   the whole time — looks like a hang but the system is working.

#### Thinking-block leak

Claude Code's UI rendered Qwen's *entire* thinking block as visible
assistant content, e.g. lines like
"Per my global config, I should reply in Mandarin..." appeared in the
chat transcript before the actual answer. Root cause:

| Layer | What it does | What it expects |
| --- | --- | --- |
| Anthropic protocol | Has a dedicated `thinking_block` content type Claude Code knows to fold away | `{"type":"thinking", "thinking":"…"}` blocks |
| MTPLX 0.3.6 server | Streams Qwen's raw output as plain `{"type":"text","text":"…"}` | (no thinking-block translation) |
| Qwen 3.6 model | Emits thinking inline using `<think>…</think>` text markers | (model-level convention) |

Three protocols, no agreement on where the thinking boundary is — so the
client renders it as content. There is no per-request fix in 0.3.6; mitigate
by launching server with `--reasoning off` for any Claude Code session.

#### Protocol mismatch layers

Beyond the thinking leak, three more compatibility gaps make
Claude Code + local Qwen significantly weaker than Claude Code + cloud:

1. **Tool calls** — Claude Code expects strict Anthropic tool_use JSON
   blocks. Qwen 27B has to be prompt-engineered into producing them;
   accuracy and reliability drop vs cloud Claude's native training.
2. **Long-context attention** — Claude Code workflows commonly cross
   50K+ tokens (multi-file reads, long histories). Qwen 27B's
   effective working context degrades well before its nominal 262K
   limit on M4 base due to prefill cost.
3. **Background "small fast" tasks** — Claude Code's compaction and
   tool-routing default to a Haiku-class model. Pointing both
   `ANTHROPIC_MODEL` and `ANTHROPIC_SMALL_FAST_MODEL` at the same
   27B model means even tiny housekeeping calls pay full 27B latency.

#### Verdict

Claude Code + local Qwen 27B on M4 base is **suitable only for**:

- Curiosity / capability-gap exploration (one-shot Q&A to compare against
  cloud Claude responses).
- Single-turn chat-style queries with short context (use the
  built-in chat UI at `http://127.0.0.1:6767/` instead of Claude Code —
  it's the same backend without the heavyweight system prompt).

**Not suitable for**:

- Real coding work (multi-file edits, debugging, refactoring).
  Each turn at 1-4 minutes makes the loop unusable.
- Agentic tasks involving tool calls or planning.

#### What to use instead

- **Lighter coding agent**: `aider` or `continue.dev` send much smaller
  system prompts than Claude Code (~1-3K tokens vs 20K+), so prefill
  cost drops 5-10×.
- **Smaller model**: a 7B-14B model on M4 base prefills 2-3× faster
  than 27B, and the lower quality is often fine for code completion-style
  workflows.
- **Different hardware**: M4 Pro or Max with 2-4× the memory bandwidth
  brings prefill of 20K tokens down to under a minute, which makes
  Claude Code + local LLM start to be usable.

### 9.5 Profile choice per workload

| Workload | Profile | Reasoning | Notes |
| --- | --- | --- | --- |
| Browser chat, single-turn Q&A | `performance-cold` | on/off | Short context, max burst speed |
| **Claude Code, code editing** | **`sustained`** | **on** | Long context > 8K, stable prefill |
| RAG over large docs | `sustained` | off | Long prompt, don't waste tokens on thinking |
| Math / multi-step reasoning | `performance-cold` | on | Short prompt, value thinking quality |

If you need two workloads in parallel, run two server instances on different
ports (e.g. 6767 sustained for Claude Code, 6768 performance-cold for chat).

---

## 10. Quick reference

```bash
# Install
pipx install mtplx

# Download Qwen 27B (~16.4 GB)
mtplx pull Youssofal/Qwen3.6-27B-MTPLX-Optimized-Speed

# Start server — daily driver (chat / single-turn)
mtplx serve \
  --model Youssofal/Qwen3.6-27B-MTPLX-Optimized-Speed \
  --profile performance-cold \
  --host 127.0.0.1 --port 6767 \
  --reasoning on --no-stats-footer --open-browser

# Start server — Claude Code backend (long-context coding)
mtplx serve \
  --model Youssofal/Qwen3.6-27B-MTPLX-Optimized-Speed \
  --profile sustained \
  --host 127.0.0.1 --port 6767 \
  --reasoning on --no-stats-footer

# Use chat UI       → http://127.0.0.1:6767/
# OpenAI API        → http://127.0.0.1:6767/v1/chat/completions
# Anthropic API     → http://127.0.0.1:6767/v1/messages
# Claude Code       → ANTHROPIC_BASE_URL=http://127.0.0.1:6767 \
#                     ANTHROPIC_API_KEY=dummy \
#                     ANTHROPIC_MODEL=mtplx-qwen36-27b-optimized-speed \
#                     ANTHROPIC_SMALL_FAST_MODEL=mtplx-qwen36-27b-optimized-speed \
#                     claude
```
