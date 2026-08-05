# mini-swe-agent (SWE-agent/mini-swe-agent) @ a83fcae82d2a

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04 · citations pinned to the commit below.*

Follow-up work (traces not performed in this study): cross-check swebench_backticks.yaml/swebench_xml.yaml variants against the "text-based" claim; read models/utils/anthropic_utils.py and cache_control.py in full if prompt-caching behavior becomes a matrix scoring input; spot-check `mini-extra swebench-single` output against a live run if reproducibility needs first-hand verification.

Repo: `~/mini-swe-agent`, pinned `git rev-parse HEAD` = `a83fcae82d2a08f0ee0c688f9d137b3566c097f8` (commit-history, high confidence).

## A. Orientation

**The "100 lines" claim does not hold at this pinned commit.** README.md:26 states: "Minimal: Just some 100 lines of python for the agent class ... (and a bit more for the environment, model, and run script)". Actual line counts read directly:

- `src/minisweagent/agents/default.py` — **190 lines** (the canonical minimal agent loop; `wc -l` verified).
- `src/minisweagent/environments/local.py` — 92 lines.
- `src/minisweagent/models/litellm_model.py` — 164 lines.
- `src/minisweagent/run/hello_world.py` — 43 lines (the truly minimal run script; explicitly labeled "the simplest possible example," `hello_world.py:1-2`). `run/mini.py`, the actual packaged CLI entry point, is 109 lines and pulls in typer/rich/config plumbing.

Sum of the four files README points to: 489 lines, not "100." The core loop (`default.py`) alone is ~1.9x the claimed figure. This is a real drift from a much smaller v1 (the repo carries a parallel `v1` branch/tag referenced in README.md:17); the codebase has grown considerably (tool-calling support, multimodal, caching, retry, multiple providers/environments) while the "100 lines" marketing line was not updated. Still, in the landscape of harnesses in this study, `default.py` is genuinely tiny and readable in one sitting — the claim is directionally true, numerically wrong by ~2x.

**Package layout** (`src/minisweagent/`):
- `agents/` — `default.py` (DefaultAgent/AgentConfig), `interactive.py` (InteractiveAgent, REPL confirm/yolo/human modes), `utils/prompt_user.py`.
- `environments/` — `local.py`, `docker.py`, `singularity.py`, `extra/{bubblewrap,contree,swerex_docker,swerex_modal}.py`.
- `models/` — `litellm_model.py` (default), `litellm_textbased_model.py`, `litellm_response_model.py`, `openrouter_*`, `portkey_*`, `requesty_model.py`, `utils/{actions_text,actions_toolcall,actions_toolcall_response,anthropic_utils,cache_control,content_string,openai_multimodal,retry}.py`, `extra/roulette.py`.
- `run/` — `mini.py` (CLI `mini`), `hello_world.py`, `utilities/{config,inspector,mini_extra}.py`, `benchmarks/{swebench,swebench_single,programbench}.py` + `benchmarks/utils/{batch_progress,common}.py`.
- `config/` — YAML templates: `default.yaml`, `mini.yaml`, `mini_textbased.yaml`, `benchmarks/{swebench,swebench_backticks,swebench_modal,swebench_xml,programbench}.yaml`.
- `exceptions.py`, `utils/{log,serialize}.py`.

**Entry points** (`pyproject.toml:89-93`): `mini` / `mini-swe-agent` → `minisweagent.run.mini:app`; `mini-extra` / `mini-e` → `minisweagent.run.utilities.mini_extra:main`, which dispatches to `config`, `inspect`/`i`/`inspector`, `swebench`, `swebench-single`, `programbench` subcommands (`run/utilities/mini_extra.py:13-19`).

**Config format**: YAML with Jinja2 templates for prompts, merged via `recursive_merge` across `-c` flags and CLI overrides (`config/__init__.py:56-61`, `run/mini.py:63,71-92`).

**Tests**: `tests/` — 53 Python files across `agents/` (5), `config/` (3), `environments/` (5) + `environments/extra/` (5), `models/` (15), `run/` (11), `utils/` (2), plus `test_data/`. pytest-based, codecov badge in README.md:32 (`impl`/`config` evidence, not independently re-run here).

**Docs**: `docs/` mkdocs site (mkdocs.yml), deployed to mini-swe-agent.com; includes `faq.md`, `advanced/v2_migration.md`, `advanced/control_flow.md`, `usage/{mini,swebench,programbench,inspector}.md`, `models/local_models.md`.

**License**: MIT (`LICENSE.md:1-3`, `pyproject.toml:12,28`).

**Maturity**: 1019 commits total, first commit 2025-06-28, HEAD commit 2026-07-22 (`git log`, commit-history). Commit cadence in the run-up to the pin is near-daily/multi-per-day (7 commits 2026-06-29 alone). This is an actively maintained project, not an abandoned research snapshot — contradicts a "toy baseline, unmaintained" framing.

**Relationship to full SWE-agent / SWE-bench**: same organization (SWE-agent GitHub org), same original authors (Kilian Lieret, Carlos E. Jimenez — `pyproject.toml:14-17`). README.md:19-21,83-103 positions `mini` as the default recommended tool over the older, more heavyweight `swe-agent` for most use cases, explicitly for lower overfitting risk in fine-tuning/RL and simpler benchmark eval. It functions as SWE-bench's reference "bash-only" evaluation harness — README.md:54 references a "SWE-bench (bash only)" leaderboard, and `run/benchmarks/swebench.py` is a first-class, actively maintained batch runner against the official SWE-bench/SWE-bench_Verified/SWE-bench_Lite/SWE-smith/SWE-rebench HF datasets (`run/benchmarks/swebench.py:53-62`).

## B. The agent loop, fully

Everything below is `default.py` end to end, augmented with the model/environment classes it delegates to.

**Construction & first turn** (`agents/default.py:88-95`): `DefaultAgent.run(task)` resets `self.messages = []` and appends two messages built from Jinja2 templates: `system_template` and `instance_template` (`AgentConfig`, `default.py:19-35`), rendered via `_render_template` (`default.py:66-67`) against `get_template_vars()` (`default.py:52-64`), which recursively merges agent config, environment vars, model config, run stats (`n_model_calls`, `model_cost`, `elapsed_seconds`), and any extra kwargs (e.g. `task`).

**Main loop** (`default.py:96-124`): `while True: self.step()` where `step()` = `self.execute_actions(self.query())` (`default.py:126-128`). Breaks when `self.messages[-1].get("role") == "exit"` (`default.py:122-123`), returning `self.messages[-1].get("extra", {})` — the caller sees `exit_status`/`submission`.

**Model call** (`query()`, `default.py:130-152`): checks `step_limit`/`cost_limit` and `wall_time_limit_seconds` first, raising `LimitsExceeded`/`TimeExceeded` (both subclass `InterruptAgentFlow`, `exceptions.py:13-18`) if exceeded — these are checked *before* the call, so they gate the next step rather than interrupt mid-call. Then `self.model.query(self.messages)`.

`LitellmModel.query()` (`models/litellm_model.py:81-106`) is the default model client (`models/__init__.py:110-113`; `litellm >= 1.75.5` pinned in `pyproject.toml:38`). It calls `litellm.completion(model=..., messages=..., tools=[BASH_TOOL], **model_kwargs)` (`litellm_model.py:64-71`) wrapped in a tenacity retry loop (`models/utils/retry.py:9-25`: `stop_after_attempt` env-configurable, default 10; `wait_exponential(multiplier=1, min=4, max=60)`; aborts immediately on `UnsupportedParamsError`, `NotFoundError`, `PermissionDeniedError`, `ContextWindowExceededError`, `AuthenticationError`, `KeyboardInterrupt` — `litellm_model.py:50-57`). Cost is computed via `litellm.cost_calculator.completion_cost` and tracked both per-agent (`self.cost`) and globally (`GLOBAL_MODEL_STATS`, `models/__init__.py:13-42`, itself capped by `MSWEA_GLOBAL_COST_LIMIT`/`MSWEA_GLOBAL_CALL_LIMIT` env vars).

**Response parsing / action extraction — tool-calling by default, NOT bash-only text parsing.** `LitellmModel._parse_actions` (`litellm_model.py:128-135`) calls `parse_toolcall_actions` (`models/utils/actions_toolcall.py:30-76`) against `response.choices[0].message.tool_calls`. A single `BASH_TOOL` function schema is registered (`actions_toolcall.py:11-27`: name `bash`, one string parameter `command`). `parse_toolcall_actions` raises `FormatError` (subclass of `InterruptAgentFlow`, `exceptions.py:25-26`) if `tool_calls` is empty/`None` ("No tool calls found... Every response MUST include at least one tool call," `actions_toolcall.py:40-52`), or if a tool call names something other than `bash`, has unparsable JSON arguments, or lacks a `command` key (`actions_toolcall.py:53-76`). Unlike the historical (v1) design, **multiple** tool calls per turn are accepted and executed — `execute_actions` iterates `message["extra"]["actions"]` (`default.py:156`), and `swebench.yaml:183` explicitly turns on `parallel_tool_calls: true`.

A **separate, non-default** text-based path still exists: `LitellmTextbasedModel` (`models/litellm_textbased_model.py`) subclasses `LitellmModel`, omits `tools=` from the completion call, and parses exactly one ```` ```mswea_bash_command ```` fenced block via regex (`action_regex` default `r"```mswea_bash_command\s*\n(.*?)\n```"`, `litellm_textbased_model.py:8`); `parse_regex_actions` (`models/utils/actions_text.py:15-40`) raises `FormatError` unless `len(actions) == 1`. This is wired up only via `model_class: litellm_textbased` or the `mini_textbased.yaml` config (`config/mini_textbased.yaml:1-16` uses this fenced-block system prompt). **`mini.yaml` and `swebench.yaml` — the actual default configs shipped for `mini` and `mini-extra swebench` — do not set `model_class`, so they get tool-calling `LitellmModel` by default.**

This directly **contradicts** the project's own README and FAQ, both still current at the pin: README.md:42,71 and `docs/faq.md:30-31,42` state "Does not have any tools other than bash — it doesn't even need to use the tool-calling interface of the LMs." But `docs/advanced/v2_migration.md:9` — the same repo's own v2 migration doc — states plainly: "**Tool calls**: Native tool calling API support (**now the default**)." The FAQ file was last touched 2026-01-27 (`git log -1 --date=short -- docs/faq.md`) for a typo fix, well after the v2 tool-calling default shipped; it appears stale relative to the shipped default. **CONFIRMED contradiction, doc vs. impl** (see ledger).

**Execution** (`execute_actions`, `default.py:154-157`): `outputs = [self.env.execute(action) for action in message["extra"]["actions"]]`. `LocalEnvironment.execute` (`environments/local.py:24-43`) calls a private `_run()` helper (`local.py:72-92`) that wraps `subprocess.Popen(command, shell=True, cwd=..., env=os.environ|self.config.env, start_new_session=True)` and `process.communicate(timeout=...)`; on timeout it kills the whole process group via `os.killpg(process.pid, SIGKILL)` (`local.py:88-91`) — every action is a **fresh subprocess**, confirming the "stateless shell" claim. `DockerEnvironment.execute` (`environments/docker.py:101-138`) shells out to `docker exec -w <cwd> <container> bash -lc <command>` per action against a long-lived container started once via `docker run -d ... sleep <container_timeout>` (`docker.py:74-99`). `SingularityEnvironment` similarly does `singularity exec --writable <sandbox> bash -c <command>` per action against a pre-built sandbox (`environments/singularity.py:80-119`, sandbox built via `singularity build --sandbox`, `singularity.py:46-65`).

**Environment classes enumerated** (`environments/__init__.py:8-16`): `docker` → `DockerEnvironment`, `singularity` → `SingularityEnvironment`, `local` → `LocalEnvironment`, `swerex_docker`/`swerex_modal` → SWE-ReX-backed remote execution (`environments/extra/swerex_docker.py`, 80 lines; `swerex_modal.py`, 124 lines — Modal cloud sandboxes), `bubblewrap` → `BubblewrapEnvironment` (`environments/extra/bubblewrap.py`, 151 lines, unprivileged Linux namespace sandboxing), `contree` → `ContreeEnvironment` (`environments/extra/contree.py`, 148 lines, third-party sandbox SDK).

**Observation formatting**: `format_toolcall_observation_messages` (`actions_toolcall.py:79-113`) renders each output through the model config's `observation_template` (a Jinja2 string) into `{role: "tool", tool_call_id, content, extra: {raw_output, returncode, exception_info, ...}}`. Default template embeds `<returncode>`/`<output>` XML-ish tags (`litellm_model.py:40-43`); the shipped `mini.yaml`/`swebench.yaml` override with JSON- or XML-tagged variants that additionally **truncate**: when `output.output | length >= 10000`, the template keeps only `output[:5000]` head + `output[-5000:]` tail and reports `elided_chars` (`config/mini.yaml:112-128`, `config/benchmarks/swebench.yaml:131-158`). This is the entirety of "context management" — a fixed per-observation head/tail truncation with no cross-turn summarization.

**Termination / submit mechanism**: no dedicated "finish" tool. Every environment's `_check_finished(output)` (`environments/local.py:45-56`, `docker.py:140-151`, `singularity.py:121-132` — identical logic duplicated three times) inspects the **first line of command stdout**; if it strictly equals `COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` and `returncode == 0`, it raises `Submitted` (subclass of `InterruptAgentFlow`, `exceptions.py:9-10`) carrying `{"role": "exit", "extra": {"exit_status": "Submitted", "submission": <rest of stdout>}}`. The agent's `run()` loop catches `InterruptAgentFlow` generically and appends its messages (`default.py:115-116`), and the loop-exit check on `role == "exit"` fires. The system/instance templates instruct the model to `echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` (optionally followed by `&& cat patch.txt` for SWE-bench submission) as a **standalone command, not combined with anything else** (`config/mini.yaml:18-19,41-42`; `config/benchmarks/swebench.yaml:97-104`). This sentinel-in-stdout mechanism is itself a form of convention rather than an API primitive — the v1→v2 migration doc records the sentinel text changed once already (`docs/advanced/v2_migration.md:47-49`, from `MINI_SWE_AGENT_FINAL_OUTPUT` to `COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT`), i.e., it is a soft contract the model must echo verbatim.

**Limits**: `step_limit` (default `0` = unlimited, `AgentConfig`, `default.py:26-27`), `cost_limit` (default `3.0` USD, `default.py:28-29`), `wall_time_limit_seconds` (default `0` = unlimited, `default.py:30-31`), `max_consecutive_format_errors` (default `3`, exits with `RepeatedFormatError` exit status, `default.py:32-33,100-114`). `swebench.yaml:112` overrides `step_limit: 250`. A separate **global** cost/call cap exists across concurrent agents in a process via `GLOBAL_MODEL_STATS` (env `MSWEA_GLOBAL_COST_LIMIT`/`MSWEA_GLOBAL_CALL_LIMIT`, `models/__init__.py:20-31`) — relevant for the batch runner's thread pool.

**State / persistence**: `self.messages` is the single source of truth — "completely linear history," no separate trajectory structure (README.md:45,77 "Has a completely linear history"). `Agent.serialize()`/`save()` (`default.py:159-190`) write the full message list plus model/env/agent config and cost stats to a JSON trajectory file tagged `"trajectory_format": "mini-swe-agent-1.1"` (`default.py:178`) if `config.output_path` is set; saved after **every** step via `finally: self.save(...)` (`default.py:120-121`), not just at the end.

**Confirmed**: no tool-calling API is used *only* if you opt into `LitellmTextbasedModel`/`mini_textbased.yaml`; the shipped default is tool-calling with a single `bash` tool (see contradiction above). Stateless shell confirmed (`local.py:72-92`, fresh `Popen` per action, no persisted cwd/env across actions — templates explicitly tell the model to prefix `cd`/`export` per command, `mini.yaml:39-40`). No compaction beyond fixed 5000/5000-char head/tail truncation per observation (`mini.yaml:112-128`); full message history grows unbounded otherwise and is resent every turn.

## C. Matrix inputs

| Dimension | Label | Note |
|---|---|---|
| TUI | **native** | `textual`-based `TrajectoryInspector` app for browsing saved trajectories (`run/utilities/inspector.py:16-19,79`, 316 lines); `mini` itself is a `rich`-rendered REPL, not a full TUI. |
| Headless/batch | **native** | `mini-extra swebench` batch runner with `ThreadPoolExecutor(max_workers=workers)` (`run/benchmarks/swebench.py:257-263`), plus `swebench-single` and `programbench` (193 lines) for other benchmarks. |
| Server API | **absent** | No HTTP/RPC server found anywhere in `src/`; only CLI entry points. |
| Provider abstraction | **native** | litellm (default), openrouter, portkey, requesty each with dedicated model classes (`models/__init__.py:78-89`); litellm alone covers most providers/models by proxy. |
| Local models | **partial** | Supported via litellm `model_kwargs` (`api_base`, `custom_llm_provider`) and `litellm_model_registry` for cost metadata (`litellm_model.py:32-33`, `docs/models/local_models.md:16-30`); requires manual config, no auto-discovery. |
| Token/cost accounting | **native** | Per-call cost via `litellm.cost_calculator.completion_cost`, accumulated per-agent (`self.cost`) and globally (`GLOBAL_MODEL_STATS`); `cost_tracking: ignore_errors` config escape hatch when a model isn't in litellm's registry (`litellm_model.py:108-126`). No token-count field is separately surfaced (cost only). |
| Retry | **native** | `tenacity`-based exponential backoff, 10 attempts default, explicit abort-exception allowlist (`models/utils/retry.py`). |
| MCP | **absent** | No occurrences of "mcp"/"model context protocol" anywhere in `src/` or `docs/` (grep verified). |
| ACP | **absent** | No occurrences of "agent client protocol"/"ACP" anywhere in `src/` or `docs/` (grep verified). |
| File editing | **via bash only** | No dedicated edit/patch tool; system templates explicitly teach `sed -i`, heredoc `cat <<'EOF' > file`, and `nl -ba | sed -n` for viewing (`config/mini.yaml:57-100`). |
| Permission model | **confirm-prompt (interactive) / none (batch)** | `InteractiveAgent` has three modes — `human`, `confirm` (default), `yolo` — switchable at runtime via `/u`/`/c`/`/y`; `confirm` mode prompts per-action unless the command matches a `whitelist_actions` regex (`agents/interactive.py:24-31,162-183`). `DefaultAgent` (used by the batch/SWE-bench runner) has **no confirmation step at all** — every parsed action executes unconditionally. |
| Sandboxing | **native, OS/runtime-enforced when selected** | Docker (`docker.py`), Singularity/Apptainer (`singularity.py`), Bubblewrap (unprivileged namespaces, `extra/bubblewrap.py`), Contree (`extra/contree.py`), SWE-ReX-backed Docker/Modal (`extra/swerex_docker.py`, `extra/swerex_modal.py`). `LocalEnvironment` is unsandboxed by design — raw `subprocess` on the host. |
| Compaction | **absent** | Only fixed per-observation head/tail truncation (`mini.yaml:112-128`); no summarization, no context-window-aware pruning of history. |
| Subagents | **absent** | No spawning/delegation mechanism found; single linear agent per task. Parallelism exists only at the batch-runner level (independent agent instances per SWE-bench task, `swebench.py:257-263`). |
| Config portability | **YAML, highly portable** | Single/merged YAML files fully describe system/instance templates, limits, environment, and model config; `-c file1.yaml -c file2.yaml -c key.path=value` composition (`run/mini.py:63,71-92`; `config/__init__.py:56-61`). |
| Effort to change the loop | **trivially forkable** | `default.py` (190 lines) is the entire control flow; Protocol-based interfaces (`Model`, `Environment`, `Agent` in `__init__.py:43-81`) mean a custom model/environment/agent subclass integrates by duck typing alone — no plugin registration required beyond the name→class dict in each package's `__init__.py`. |

## D. Governing philosophy

README.md:21: "We now ask: **What if our agent was 100x simpler, and still worked nearly as well?**" The stated design principles (README.md:42-50,71-79; `docs/faq.md:27-33`):
- No tools beyond bash (per the docs, though contradicted by the shipped tool-calling default — see B).
- "Has a completely linear history — every step of the agent just appends to the messages and that's it. ... there's no difference between the trajectory and the messages that you pass on to the LM. Great for debugging & fine-tuning" (README.md:45-47).
- "Executes actions with `subprocess.run`... every action is completely independent... This makes it trivial to execute the actions in sandboxes (literally just switch out `subprocess.run` with `docker exec`)" (README.md:48-50). The FAQ elaborates a specific, reasoned argument against persistent shell sessions: ambiguous command-termination detection, single bad commands killing the whole session, and interrupt handling corrupting subsequent output (`docs/faq.md:104-118`, "why no shell session").
- Explicit refusal to own task-specific tooling: "Want it to do something specific like opening a PR? Just tell the LM to figure it out rather than spending time to implement it in the agent" (README.md:73-74).
- Positioned deliberately as a **baseline/control** for research: "This makes it perfect as a baseline system and for a system that puts the language model (rather than the agent scaffold) in the middle of our attention" (README.md:52-53), with its own SWE-bench leaderboard track ("SWE-bench (bash only)").

## E. Hidden tradeoffs of minimalism

- **RISK — stale documentation actively misrepresents the current default.** README/FAQ claim no tool-calling; the shipped default (`LitellmModel`) uses it. A downstream consumer relying on the docs to justify "works with any model, no function-calling support needed" would be wrong for the default config; the text-based fallback exists but requires explicit opt-in. CONFIRMED by direct code read (`litellm_model.py:69`) vs. `docs/faq.md:30-31` vs. `docs/advanced/v2_migration.md:9`.
- **CONFIRMED — no mid-command cancellation.** `LocalEnvironment`/`DockerEnvironment`/`SingularityEnvironment.execute()` blocks on `process.communicate(timeout=...)`; the only interrupt path is a hard `timeout` kill (`local.py:86-91`) or, in `InteractiveAgent`, `KeyboardInterrupt` caught at the `step()` boundary (`interactive.py:109-122`) — which can only interrupt *between* actions, not mid-subprocess, since `Popen.communicate` is a blocking call in the main thread with no other input channel. A stuck long-running command in `DefaultAgent` (non-interactive/batch mode) can only be ended by the per-command `timeout` config, not by the caller.
- **CONFIRMED — observation truncation loses information silently-ish.** Beyond 10,000 chars, the model sees only head 5000 + tail 5000 chars, with `elided_chars` reported but no way to retrieve the middle short of re-running a more targeted command (`mini.yaml:112-128`). This is a hard, fixed limit with no adaptive sizing based on remaining context budget.
- **RISK — unbounded message history with no compaction.** Because "completely linear history" is a *design feature* (README.md:45-47), long-running tasks accumulate every system/user/tool message and resend all of them every turn; the only bound is the model's own context window (surfaced only reactively via `ContextWindowExceededError` aborting retries, `litellm_model.py:54`) or the `cost_limit`/`step_limit`. No token-budget-aware pruning exists.
- **CONFIRMED — no prompt-caching by default beyond a single opt-in flag.** `set_cache_control: "default_end"` is auto-enabled only for Anthropic-family model names (`models/__init__.py:55-61`) and otherwise must be set manually per model (`litellm_model.py:34-35`); combined with unbounded linear history, non-cached long tasks resend and re-bill the full transcript every step (cost impact: `cost_limit` default is only $3.00, `default.py:28`, suggesting the maintainers are aware this adds up fast).
- **CONFIRMED — stateless shell forces per-command prefix boilerplate onto every model turn** (`cd`/`export` must be repeated each time, `mini.yaml:39-40`); the FAQ frames this as a deliberate stability tradeoff, but it does add latent per-turn token overhead and a class of "forgot to `cd`" failures that a stateful shell wouldn't have.
- **CONFIRMED — no confirmation/whitelist in the default (non-interactive) agent path.** `DefaultAgent.execute_actions` runs every parsed action unconditionally (`default.py:154-157`); the human-in-the-loop confirm/whitelist mechanism lives only in `InteractiveAgent` (`interactive.py:124-139,162-183`), which the batch/SWE-bench runner does not use. Any approval-posture guarantee has to be built by the operator (e.g., choice of sandbox), not the harness.

## F. Interop

No MCP, no ACP anywhere in source or docs (grep-verified, see matrix). This is consistent with the stated philosophy: the harness deliberately refuses to define a tool/protocol surface at all — "bash is the only tool" — so there is no natural MCP server-registration point, and no ACP client/agent handshake exists for editor integration. Other harnesses could still consume mini-swe-agent as:
- A **batch runner reference**: `run/benchmarks/swebench.py` is a clean example of parallel-instance dispatch (`ThreadPoolExecutor`), per-instance trajectory + `preds.json` output, and Docker image resolution from SWE-bench instance metadata (`swebench.py:68-94,122-177`) — directly reusable as a template for evaluating *other* harnesses against the same benchmark, since the harness itself is swappable behind `get_agent`/`get_model`/`get_environment`.
- A **trajectory format reference**: `"trajectory_format": "mini-swe-agent-1.1"` (`default.py:178`) is a flat JSON of `{info: {...}, messages: [...]}`; simple enough that any other harness's adapter could translate its own trajectory into this shape for downstream tooling (the `textual` inspector) without needing mini-swe-agent's runtime.

## G. Benchmark inputs (control-arm invocation)

**Single task (interactive CLI)**:
```bash
mini -t "Fix the bug in foo.py" -m anthropic/claude-sonnet-4-5-20250929 -y -l 3.0
```
(`-t`/`--task`, `-m`/`--model`, `-y`/`--yolo` disables confirmation, `-l`/`--cost-limit`; `run/mini.py:56-65`.)

**Single SWE-bench instance (non-interactive, for debugging)**:
```bash
mini-extra swebench-single --subset verified --split test \
  --model anthropic/claude-sonnet-4-5-20250929 -i sympy__sympy-15599
```
(`docs/usage/swebench.md`; entry via `run/benchmarks/swebench_single.py`.)

**Batch (SWE-bench, parallel)**:
```bash
mini-extra swebench --model anthropic/claude-sonnet-4-5-20250929 \
  --subset verified --split test --workers 4 -o results/
```
Parallelism is thread-based (`ThreadPoolExecutor(max_workers=workers)`, `swebench.py:257`), one Docker/Singularity container per instance (`get_sb_environment`, `swebench.py:79-94`), default config `config/benchmarks/swebench.yaml` (`swebench.py:51`). `--subset` maps to HF datasets (`full`/`verified`/`lite`/`multimodal`/`multilingual`/`smith`/`rebench`, `swebench.py:53-62`); `--slice`/`--filter`/`--shuffle` control instance selection (`swebench.py:180-197`).

**Pinning the model**: litellm model string via `-m`/`--model` or config `model.model_name` (e.g. `anthropic/claude-sonnet-4-5-20250929`, `gemini/gemini-3-pro-preview`, `openai/gpt-5.4`); permanently via `mini-extra config set MSWEA_MODEL_NAME <model>` (`run/utilities/config.py:73-78`, writes to a global `.env`) or the `MSWEA_MODEL_NAME` env var (`models/__init__.py:73-74`). Base URL / local models: set `model.model_kwargs.api_base` and `model.model_kwargs.custom_llm_provider` in YAML (passed straight through to `litellm.completion(**model_kwargs)`, `litellm_model.py:64-71`; `docs/models/local_models.md:16-45`).

**Capping cost/steps**: `-l`/`--cost-limit` CLI flag (0 disables, `run/mini.py:62`) or config `agent.cost_limit` (default $3.00) / `agent.step_limit` (default 0 = unlimited; `swebench.yaml:112` sets 250) / `agent.wall_time_limit_seconds`; global cross-agent caps via `MSWEA_GLOBAL_COST_LIMIT`/`MSWEA_GLOBAL_CALL_LIMIT` env vars (`models/__init__.py:20-21`).

**Trajectory/output format**: per-instance `<instance_id>.traj.json` (`swebench.py:163`) containing `{"info": {"model_stats": {...}, "config": {...}, "exit_status": ..., "submission": ...}, "messages": [...], "trajectory_format": "mini-swe-agent-1.1"}` (`default.py:159-180`); batch predictions aggregated into `preds.json` keyed by `instance_id` with `model_name_or_path`/`model_patch` (`swebench.py:97-108`) — this is the SWE-bench-official submission format, directly consumable by SWE-bench's own scoring harness.

**Approval-posture flags**: `-y`/`--yolo` (no confirmation) vs. default `confirm` mode vs. `human` mode, only meaningful for `mini`/`InteractiveAgent` (`run/mini.py:61`, `agents/interactive.py:24-31`). The batch runner (`DefaultAgent`) has no approval posture — it is unconditionally "yolo" by construction, so containment must come from the environment sandbox choice, not a flag.

## H. Portability-adapter inputs

With no plugin system, the entire "adapter surface" is a YAML config file plus the two Jinja2 templates it carries:

- `agent.system_template` — first message; sets the model's role framing (`config/mini.yaml:2-3`: one line, "You are a helpful assistant that can interact with a computer").
- `agent.instance_template` — second message; carries `{{task}}`, workflow instructions, submission-command instructions, `{{system}}/{{release}}/{{version}}/{{machine}}` platform vars, and few-shot examples of tool-call formatting (`config/mini.yaml:4-100`). An adapter implementing a spec→plan→tests→implement→review→verify→PR methodology would rewrite this template almost entirely — it's the only place task-specific process instructions live.
- `model.observation_template` / `model.format_error_template` — control how command output and parse errors are shown back to the model (`config/mini.yaml:112-149`); an adapter tuning verbosity/format for its own review conventions would edit these.
- `agent.step_limit` / `cost_limit` / `wall_time_limit_seconds` / `max_consecutive_format_errors`, `environment.cwd`/`timeout`/`env`, `model.model_name`/`model_kwargs` — straightforward numeric/env knobs (`AgentConfig`, `default.py:19-35`; `LocalEnvironmentConfig`, `local.py:13-16`).

**What an adapter would fight**: almost nothing structurally — there is no built-in TODO list, no compaction, no subagent routing, no permission gate in the default path to work around. The flip side is that an adapter gets **no help either**: no built-in support for structured plan tracking, no automatic verification-loop scaffolding, no git/PR helper beyond "tell the model to run `git diff`/`gh pr create` in bash" (which the swebench.yaml template already does for patch generation, `swebench.yaml:77-110`). A spec→plan→tests→implement→review→verify→PR methodology is entirely achievable **unattended**, because the model can run any bash it wants (create planning files, run test suites, `git commit`, invoke `gh`) and the harness will faithfully execute and report back — but "achievable" here means "the model has to invent and hold the entire process in its own reasoning and in scratch files it writes," since the harness supplies no structural support (no step/plan object, no built-in review gate, no test-result parser). This is honestly closer to "possible but crude": every piece of process discipline (a persistent plan file, checking off steps, invoking a linter/formatter, opening a PR) has to be bash-commands-and-prose the adapter's system/instance template teaches the model to perform, with zero harness-level enforcement that any step actually happened before the model calls the submit sentinel. Nothing stops a model from echoing `COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` having skipped review or tests; only the JSON trajectory + PR diff is observable after the fact for a human/automated grader.

## Strengths

- Genuinely small, single-file control flow (`default.py`, 190 lines) that is easy to read completely and reason about exactly — rare among the harnesses in this study.
- Environment abstraction is extremely clean: swap `LocalEnvironment` for `DockerEnvironment`/`SingularityEnvironment`/etc. by changing one config key; every environment class implements the identical 3-method surface (`execute`, `get_template_vars`, `serialize`).
- Because every action is `subprocess.run`/`docker exec`, execution is trivially parallelizable — the SWE-bench batch runner is a straightforward `ThreadPoolExecutor` over independent agent+environment instances.
- Config-as-YAML plus Jinja2 templates gives real portability without needing to fork the Python at all for prompt/behavior changes.
- Actively maintained (near-daily commits, 1019 commits, MIT license, used as the reference SWE-bench baseline) — not an abandoned research artifact.

## Weaknesses

- Stale docs (README/FAQ) misstate the current default behavior on the single most identity-defining claim ("no tool-calling") — a real trust gap for anyone auditing the project from its public description alone.
- Zero built-in guardrails in the default (batch/non-interactive) path: no confirmation, no whitelist, no partial-output preservation beyond what the model chooses to do — containment is 100% delegated to the chosen `environment_class`.
- No compaction/summarization; long tasks either fit in context or hit `ContextWindowExceededError`/`cost_limit`, with no graceful degradation in between beyond per-observation truncation.
- No mid-command cancellation; a hung command can only be ended by a fixed timeout, not user/orchestrator intervention, in the batch path.
- No server/API mode and no MCP/ACP — cannot be embedded as a subprocess tool inside another agent's protocol-native tool-calling loop without a bespoke wrapper.

## Best-fit / poor-fit use cases

**Best fit**: SWE-bench-style batch evaluation and leaderboards; fine-tuning/RL data generation where a minimal, model-attributable scaffold is explicitly desired (README.md:91: "doing FT or RL and don't want to overfit to a specific agent scaffold"); a base to fork for a bespoke internal agent where the operator wants to own 100% of the prompt/process logic; a portability-adapter test subject specifically *because* it has no competing structure to fight.

**Poor fit**: interactive daily-driver coding assistant needing rich UI, mid-task cancellation of long commands, or built-in review/approval workflows; any setting requiring MCP/ACP interop with existing agent ecosystems; long-horizon tasks that would benefit from context compaction rather than truncation; any use case where the operator cannot trust the model to self-police process discipline (spec/plan/test/review), since the harness enforces none of it.
