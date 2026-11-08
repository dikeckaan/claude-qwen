# claude-qwen

A single-file launcher that runs [Claude Code](https://code.claude.com) against a **local GGUF model** (Qwen3.8-27B by default) on an Apple Silicon Mac. No cloud API needed: the model runs on your machine with [llama.cpp](https://github.com/ggml-org/llama.cpp), whose `llama-server` exposes an Anthropic-compatible `/v1/messages` endpoint.

## Features

- Type `claude-qwen`, pick a model, and Claude Code opens. The server starts if needed.
- **Model picker:** finds every GGUF file in the `models/` directory.
- **Model downloader:** `claude-qwen download <hf-repo>` lists the repo's GGUF files with their sizes and whether they fit in your RAM. Interrupted downloads resume where they stopped.
- **Auto shutdown:** when the last session closes, the server stops and frees its RAM.
- **Template fix:** Qwen3.8's stock chat template fails with `System message must be at the beginning` on the system messages Claude Code sends mid-conversation. `chat_template.jinja` is a patched version that accepts them.
- **Optional GPU memory tuning:** raises macOS's GPU wired-memory limit (`iogpu.wired_limit_mb`) while the server runs and restores the default when it stops. If the limit is high enough, MTP speculative decoding turns on automatically (~60% faster generation).

## Installation

### Homebrew

```bash
brew install dikeckaan/claude-qwen/claude-qwen   # also installs llama.cpp
claude-qwen download unsloth/Qwen3.8-27B-GGUF    # pick a quant from the list
claude-qwen
```

Models and logs are stored in `~/.claude-qwen`. Set `CLAUDE_QWEN_HOME` to use a different directory.

### Git

```bash
git clone https://github.com/dikeckaan/claude-qwen.git ~/claude-qwen
~/claude-qwen/claude-qwen setup                # installs llama.cpp (brew), links the command into ~/.local/bin
claude-qwen download unsloth/Qwen3.8-27B-GGUF  # pick a quant from the list
claude-qwen
```

Requirements: an Apple Silicon Mac, [Homebrew](https://brew.sh), [Claude Code](https://code.claude.com), and `~/.local/bin` on your `PATH`.

### Which quant?

Rough sizes for a 27B dense model:

| RAM | Recommended | File size |
|---|---|---|
| 24 GB | IQ4_XS / IQ3_M | 13–14 GB |
| 32 GB | Q4_K_XL / Q5_K_M | 17–20 GB |
| 48 GB+ | Q6_K / Q8_0 | 22–30 GB |

Leave about 6 GB for the context (KV cache) and the OS.

## Usage

```bash
claude-qwen                     # pick a model and open Claude Code
claude-qwen -p "question"       # any other arguments are passed through to claude
claude-qwen download [repo]     # download a GGUF
claude-qwen models              # list downloaded models
claude-qwen start | stop | restart  # manage the server without opening Claude Code
claude-qwen status | log
claude-qwen version
```

| Variable | Default | Description |
|---|---|---|
| `QWEN_MODEL` | (menu) | Skip the menu: a number or a file name |
| `QWEN_CTX` | `102400` | Context size. Claude Code needs at least 100K for auto-compaction |
| `QWEN_EFFORT` | `medium` | Reasoning effort: `xhigh`, `medium`, `low`, `none` |
| `QWEN_MTP` | `auto` | Speculative decoding with the model's built-in MTP head. `auto`: on when the GPU limit allows. `0`/`1`: force off/on |
| `QWEN_KEEP` | `0` | `1`: keep the server running after the session closes |
| `QWEN_GPU_MB` | RAM × 7/8 | GPU wired-memory limit to use while the server runs |
| `QWEN_EXTRA` | | Extra `llama-server` arguments |
| `CLAUDE_QWEN_HOME` | `~/.claude-qwen` | Data directory for models, logs and sessions (a git clone uses its own directory) |

### GPU memory limit (optional)

By default macOS gives the GPU about 75% of RAM. On a 24 GB Mac that is not enough for a 27B model plus 100K context plus MTP. `claude-qwen setup` prints a command that installs a narrow sudoers rule allowing exactly two commands without a password: `sysctl iogpu.wired_limit_mb=<N>` and `sysctl iogpu.wired_limit_mb=0`. The command writes the rule as root directly into `/etc/sudoers.d` and validates it with `visudo` before activating it. With the rule installed, the script raises the limit while the server runs and restores the default when it stops. Without it everything still works; MTP just stays off.

## Performance (Apple M6, 24 GB, 100K context)

Generation speed (tokens per second):

| Model | MTP off | MTP on |
|---|---|---|
| Qwen3.8-27B UD-IQ4_XS (14.3 GB) | 9.2 | 14.9 (~16 inside Claude Code) |
| Qwen3.8-27B Uncensored IQ3_M (12.8 GB) | 10.1 | 15.5–18.5 (~16 inside Claude Code) |

- MTP uses the multi-token prediction head embedded in the model. It needs no extra file and does not change output quality. `--spec-draft-n-max 3` was slightly faster than 2.
- Prefill runs at about 210–300 tokens/s. Claude Code's first request is about 14K tokens, so the first reply in a session takes around a minute; later requests reuse the cache.
- To go faster: set `QWEN_EFFORT=low`, and disable MCP servers or plugins you don't need in claude-qwen sessions.

## Notes

- A local model is less capable than Claude in the cloud. Be careful with `--dangerously-skip-permissions`.
- The "unrecognized model" and "connectors disabled" warnings at startup are expected.

## License

MIT. `chat_template.jinja` is derived from the Qwen3.8 chat template (Apache 2.0).
