# learn-herness

A step-by-step project for building an **AI coding agent harness** from scratch on top of the
Anthropic Messages API.

The idea is incremental: every `src/sNN.py` file is a self-contained lesson that takes the
previous step and adds exactly one new capability. You can read each file top to bottom and see
what the "agent" actually is — a loop, a system prompt, and some tools.

## The core pattern

Every lesson shares the same skeleton:

1. Send a prompt + tool definitions to `client.messages.create()`.
2. Append the assistant turn to the message history.
3. Look for `tool_use` blocks in the response. If there are none, the turn is done.
4. Otherwise, execute each tool locally, collect `tool_result` blocks, append them as a user
   turn, and **loop back to step 1**.

That while-loop *is* the agent. Everything else in the repo is scaffolding around it.

## Lessons

| Step | File | What it adds |
|------|------|--------------|
| s01 | [`src/s01.py`](src/s01.py) | **Agent loop** — the minimal working agent: one `bash` tool, `subprocess` execution, a tool-use loop, and an interactive REPL. |
| s02 | [`src/s02.py`](src/s02.py) | **Tool use** — four more file tools (`read_file`, `write_file`, `edit_file`, `glob`) dispatched through a handler map, plus workspace path sandboxing. |
| s03 | _not written yet_ | Next capability (context management / permissions / sub-agents). |

### s01 — Agent Loop

- System prompt pins the agent to the current working directory and instructs it to
  *"Use bash to solve tasks. Act, don't explain."*
- Single tool: `bash`, with a `command` string parameter.
- `run_bash()` guards against obviously destructive commands (`rm -rf /`, `sudo`, `shutdown`,
  `reboot`, `> /dev/`), enforces a 120s timeout, and truncates output at 50k chars.
- `agent_loop()` runs until the model replies without calling a tool.
- Entry point is a REPL (`s01 >> `) that keeps a `history` list across turns, so the agent has
  conversational memory within a session. Type `q` to quit.

### s02 — Tool Use

Adds to s01:

- `read_file` (with optional `limit`, reporting how many lines were elided), `write_file`
  (creates parent directories), `edit_file` (single literal `old_text` → `new_text` replace),
  and `glob` (sorted matches, capped at 200, `root_dir`-scoped).
- `TOOL_HANDLERS` maps tool names to Python callables; the loop does
  `handler(**block.input)` instead of hard-coding `run_bash`, so adding a tool means adding a
  schema and a function and nothing else.
- `safe_path()` resolves every requested path and raises if it isn't relative to `WORKDIR` —
  file tools cannot escape the workspace. (Note: the `bash` tool is still unsandboxed.)
- Each model turn is executed *before* the loop feeds results back, so parallel tool calls from
  one assistant message work naturally.

## Requirements

- Python >= 3.12
- [uv](https://docs.astral.sh/uv/)
- An Anthropic-compatible API key

Dependencies: `anthropic`, `python-dotenv`, `pyyaml`.

There is no `requirements.txt` — dependencies are declared in `pyproject.toml` and pinned in
`uv.lock`, and installed with `uv sync`.

> `pyyaml` is declared in `pyproject.toml` but not yet used in `src/` — it is staged for a later
> lesson (config/prompts as data rather than string literals).

## Setup

```bash
uv sync
cp .env.example .env   # or create .env by hand, see keys below
```

`.env` is git-ignored and read via `python-dotenv`. Required keys:

| Key | Purpose |
|-----|---------|
| `ANTHROPIC_API_KEY` | API key for the client. |
| `ANTHROPIC_BASE_URL` | Optional endpoint override — point this at any Anthropic-compatible gateway or proxy. |
| `MODEL_ID` | Model sent to `messages.create()`. Read with `os.environ[...]`, so it is mandatory. |

## Run

Each lesson is executed as a script (the `learn-herness` console entry point is still the uv
boilerplate `Hello` stub):

```bash
uv run python src/s01.py
uv run python src/s02.py
```

You get a colored prompt; type a task and press Enter, or `q` to quit. Tool calls are echoed to
the terminal (`$ command` in s01, `> tool_name` in s02) with the first 200 chars of each result,
followed by the model's final text answer.

## Cross-platform notes

The system prompt branches on `os.name`:

- On Windows it tells the model the "bash" tool actually runs through `cmd.exe`, so it must emit
  cmd syntax rather than Bash/PowerShell.
- On Unix-like systems it says the tool runs the system shell.

ANSI color escapes in the REPL prompt are wrapped in `\001`/`\002` so Readline treats them as
zero-width and cursor editing stays sane.

## Layout

```
.
├── pyproject.toml          # uv project metadata + dependencies
├── uv.lock                 # pinned dependency lockfile
├── SUMMARY.md              # short-form project summary
├── src/
│   ├── s01.py              # lesson 1: agent loop
│   ├── s02.py              # lesson 2: tool use
│   └── learn_herness/      # package stub (unused entry point)
├── .env.example            # tracked config template
├── .env                    # API key / base URL / model id (git-ignored)
└── README.md
```

## Learning goals

By the end you should be able to answer, concretely:

- Why an agent loop terminates, and who decides when the model is "done".
- How tool schemas shape model behavior, and why a handler map beats a switch statement.
- What a harness has to enforce that the model cannot: timeouts, output caps, path sandboxing,
  command blocklists.
- Where message history, token cost, and context limits start to bite.

## Disclaimer

These are teaching examples, not a production agent. Command blocklists are pattern-matched and
trivially bypassable, history is never compacted, and there is no permission layer — run them in
a directory you're comfortable having files written to.
