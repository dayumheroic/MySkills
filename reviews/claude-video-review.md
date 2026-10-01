# Review: bradautomates/claude-video ("watch" skill), v0.3.2

Reviewed at commit `03ceb42` (tag v0.3.2, 2026-09-25). MIT licensed. Author: Bradley Bonanno.

## What it is

An Agent Skills package that gives a coding agent a video input. The agent runs a bundled
Python script against a URL or local file and gets back either a timestamped frame set plus
a transcript (local engine) or a timestamped answer from Google's Gemini video model
(gemini engine). It installs as a Claude Code plugin, a Codex plugin, or a generic skill
(`npx skills add`) for Cursor, Copilot and other hosts.

## Capabilities

### Two engines
| Engine | How it works | Needs |
|---|---|---|
| `local` (default without a key) | yt-dlp downloads media and one caption track; ffmpeg extracts frames; captions become the transcript, with optional WhisperX/Groq/OpenAI speech fallback. The agent reads the frames itself. | Python 3.10+, ffmpeg/ffprobe, yt-dlp (plus Deno for YouTube) |
| `gemini` (auto-selected when `GEMINI_API_KEY` resolves) | YouTube URLs are sent to Google directly. Other URLs and local files are uploaded to Google's Files API, then deleted. Gemini answers the `--question` with timestamps. | Google AI Studio key; ffmpeg/yt-dlp only for non-YouTube sources |

No silent fallback between engines. A Gemini failure is reported with a category (auth, quota, rejected, upload, service, network, response) and the user decides whether to rerun locally.

### Local engine detail levels
| Detail | Frame selection | Cap |
|---|---|---|
| `transcript` | none unless `--timestamps` cues | 0 |
| `efficient` | keyframes, uniform fallback | 50 |
| `balanced` (default) | scene changes, uniform fallback | 100 |
| `token-burner` | scene changes, uncapped | warns over 250 |

Other local options: `--start/--end` focus window, `--timestamps` cue frames, `--max-frames`,
`--resolution` (512 default, 1024 for on-screen text), `--fps` (max 2), `--no-dedup`
(16x16 RGB thumbnail near-duplicate removal is on by default), `--sub-lang`, `--cookies`,
`--cookies-from-browser`, `--out-dir`.

### Transcription fallbacks
`whisperx` (managed local install: uv + Python 3.12 venv + WhisperX 3.8.6, ~1.5 GB download, 3 GB disk, 8 GB RAM, CPU int8 small model), `groq` (whisper-large-v3), `openai` (whisper-1), or `none`. Captions always come first. Cloud uploads are chunked under a 24 MB budget with source-time offsets restored.

### Setup wizard and config
First run asks engine, then (local only) detail and fallback. Settings live in `~/.config/watch/.env` (mode 0600). Key lookup order: environment, user file, then a `.env` in the current working directory. A SessionStart hook runs `setup.py --check` and prints an advisory line if setup needs attention.

### Output
A markdown report on stdout: metadata header, a `## Frames` list of absolute JPEG paths with `t=MM:SS` and selection reason, a `## Transcript` code fence of `[MM:SS] text` lines, an optional `## Unavailable evidence` section, and a footer naming the disposable work dir. Gemini runs produce `## Answer (from Gemini)` instead of frames.

## Verification done in this session
- Full pytest suite: 248 passed, 1 skipped (needs yt-dlp, not installed here). Python 3.11, FFmpeg 6.1.
- Smoke test of the local engine on a synthesized 8 s clip: 12 frames extracted with correct timestamps, report well-formed, frame images render.
- setup.py --json correctly reported yt-dlp missing and `can_proceed: false`.

## Engineering quality
- Pure standard-library Python (argparse, json, urllib, subprocess, ssl, etc.). No pip dependencies for the base runtime.
- All subprocess calls use list argv, no `shell=True`. URLs go after `--`, sources starting with `-` are rejected, local paths are resolved before use. These were hardened after a reported option-injection issue in 0.1.3.
- Keys travel only in HTTP headers, are scrubbed from Gemini diagnostics, and never appear in reports or argv. Config writes are atomic (mkstemp + replace).
- CI matrix: Ubuntu 22.04/latest, macOS, Windows, real ffmpeg, latest yt-dlp. Tag push builds and attaches `dist/watch.skill` to a GitHub release.
- Thorough docs: README, CHANGELOG, AGENTS.md with structure and release rules.

## Concerns and findings

Security and privacy
1. **cwd `.env` key fallback.** A project `.env` containing `GEMINI_API_KEY` silently switches the default `auto` engine to Gemini, so a local or cookie-authenticated private video gets uploaded to Google. Mitigation: set `WATCH_ENGINE=local` or always pass `--engine local` for private material.
2. **Prompt-injection surface.** Captions, Whisper output, video title/uploader, Gemini's answer, and up to 3000 chars of yt-dlp stderr are placed in the report verbatim. A transcript segment containing three backticks can break out of the code fence. SKILL.md tells the agent to treat this as untrusted evidence, but the report itself carries no framing.
3. **Unprompted installs.** A bare `python3 setup.py` on macOS runs `brew install ffmpeg yt-dlp` with no confirmation. `--install-whisperx` runs `curl | sh` (Linux) or `irm | iex` (Windows) to install uv, then pulls torch and WhisperX. No sudo anywhere. `--check` and `--json` never install.
4. **Standalone frames.py CLI** deletes any `frame_*.jpg` / `cue_*.jpg` in the supplied output directory before extracting. Safe via watch.py (fresh work dir), risky if invoked directly on a user folder.
5. Cookie jar passed to yt-dlp is read/write, as documented.

Code quality (minor)
- Transcript timestamps print `[MM:SS]` with unbounded minutes (`[75:12]`) while frame timestamps use `H:MM:SS`. Inconsistent for videos over an hour.
- One ffmpeg failure path (frames.py:281) dumps unbounded stderr into the report where siblings truncate.
- A plain `setup.py` run without `--backend`/`--engine` never marks setup complete, so `first_run` stays true.
- A cue timestamp past end of file is silently dropped.
- Small dead code: unused `VIDEO_EXTS`, duplicate branches in frames.py, redundant exception clauses in whisper.py, a Homebrew hint shown on all platforms.

Environment limits (documented by the author)
- Not supported in Claude Chat or Cowork. Cloud sandboxes (including Claude Code on the web) usually block yt-dlp downloads and lose the Gemini key between tasks. Works best in local Claude Code, Codex, or other local agents.
- Only the latest yt-dlp is supported; a 403 triggers one auto-update-and-retry.
- WhisperX is verified only on Apple Silicon.

## Verdict
A well-built, honest, stdlib-only skill with solid tests and hardening. Fit for use in a local agent session. Before relying on it with private footage, pin `WATCH_ENGINE=local` and keep project `.env` files free of a Gemini key. The prompt-injection framing of the report is the main thing I would improve if adopting it.

## Mirror status
Forking or creating a repository was denied by this session's GitHub App scope (only `dayumheroic/fantastic-ride-studio` is allowed). A full git bundle of the clone with all tags is saved at `claude-video-mirror.bundle` in this session's scratchpad. To restore: `git clone claude-video-mirror.bundle claude-video`.
