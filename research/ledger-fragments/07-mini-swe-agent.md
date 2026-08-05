# Ledger: SWE-agent/mini-swe-agent @ a83fcae82d2a

| claim | mini-swe-agent | path:lines | "snippet" | class | confidence |
|---|---|---|---|---|---|
| Pinned commit verified | a83fcae82d2a08f0ee0c688f9d137b3566c097f8 | git rev-parse HEAD | `a83fcae82d2a08f0ee0c688f9d137b3566c097f8` | commit-history | high |
| "100 lines" agent-class claim is inaccurate at this pin; actual is 190 | contradicted | README.md:26 vs agents/default.py (wc -l) | `"Just some 100 lines of python for the agent class"` vs 190 total lines | impl | high |
| Core loop file sizes: default.py 190, local.py 92, litellm_model.py 164, hello_world.py 43 | yes | wc -l (4 files) | `190 .../default.py` `92 .../local.py` `164 .../litellm_model.py` `43 .../hello_world.py` | impl | high |
| Main loop: step() until exit-role message | yes | agents/default.py:96-124 | `while True: ... if self.messages[-1].get("role") == "exit": break` | impl | high |
| step() = execute_actions(query()) | yes | agents/default.py:126-128 | `return self.execute_actions(self.query())` | impl | high |
| Default model class uses litellm tool-calling with a single bash tool | yes | models/litellm_model.py:64-71 | `litellm.completion(model=..., messages=messages, tools=[BASH_TOOL], **(self.config.model_kwargs \| kwargs))` | impl | high |
| BASH_TOOL schema: one function `bash`, one string param `command` | yes | models/utils/actions_toolcall.py:11-27 | `"name": "bash", ... "properties": {"command": {"type": "string", ...}}` | impl | high |
| No tool calls -> FormatError | yes | models/utils/actions_toolcall.py:40-52 | `"No tool calls found in the response. Every response MUST include at least one tool call."` | impl | high |
| Unknown tool name or missing command arg -> FormatError | yes | models/utils/actions_toolcall.py:53-76 | `if tool_call.function.name != "bash": error_msg += f"Unknown tool '{tool_call.function.name}'."` | impl | high |
| Separate legacy text-based (regex fenced-block) model path exists, not default | yes | models/litellm_textbased_model.py:1-18; models/utils/actions_text.py:15-40 | `action_regex: str = r"\`\`\`mswea_bash_command\s*\n(.*?)\n\`\`\`"` | impl | high |
| README/FAQ claim "does not even need tool-calling interface" contradicts shipped default | contradicted | README.md:42,71; docs/faq.md:30-31,42 | `"Does not have any tools other than bash — it doesn't even need to use the tool-calling interface of the LMs."` | doc | high |
| Project's own v2 migration doc admits tool calling is now the default | yes | docs/advanced/v2_migration.md:9 | `"Tool calls: Native tool calling API support (now the default)"` | doc | high |
| FAQ file last touched 2026-01-27 (typo fix), post-dating v2 tool-calling default | yes | git log -1 --date=short -- docs/faq.md | `Tue Jan 27 2026 ... Doc: Fix typo in FAQ (#716)` | commit-history | med |
| Local execution is a fresh subprocess per action (stateless shell) | yes | environments/local.py:72-92 | `process = subprocess.Popen(command, shell=True, ..., start_new_session=os.name == "posix")` | impl | high |
| Timeout kills whole process group, not just the process | yes | environments/local.py:88-91 | `os.killpg(process.pid, signal.SIGKILL) if os.name == "posix" else process.kill()` | impl | high |
| Docker environment execs per-action against a long-lived container | yes | environments/docker.py:74-99,101-138 | `cmd = [self.config.executable, "exec", "-w", cwd] ... cmd.extend([self.container_id, *self.config.interpreter, command])` | impl | high |
| Submission mechanism: sentinel string as first line of stdout, exit 0 | yes | environments/local.py:45-56 (identical logic docker.py:140-151, singularity.py:121-132) | `if lines and lines[0].strip() == "COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT" and output["returncode"] == 0: raise Submitted(...)` | impl | high |
| Submit sentinel text changed once already (v1 -> v2) | yes | docs/advanced/v2_migration.md:47-49 | `"From echo MINI_SWE_AGENT_FINAL_OUTPUT to echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT"` | doc | high |
| Exception hierarchy: Submitted/LimitsExceeded/TimeExceeded/UserInterruption/FormatError all subclass InterruptAgentFlow | yes | exceptions.py:1-27 | `class Submitted(InterruptAgentFlow): ...` | impl | high |
| Default limits: step_limit=0 (unlimited), cost_limit=$3.00, wall_time=0 (unlimited) | yes | agents/default.py:26-31 | `cost_limit: float = 3.0` `wall_time_limit_seconds: int = 0` | impl | high |
| SWE-bench config overrides step_limit to 250 | yes | config/benchmarks/swebench.yaml:112 | `step_limit: 250` | config | high |
| Observation truncation: head 5000 + tail 5000 chars beyond 10000 total | yes | config/mini.yaml:112-128; config/benchmarks/swebench.yaml:131-158 | `output_head: {{ output.output[:5000] }} ... output_tail: {{ output.output[-5000:] }}` | config | high |
| No compaction/summarization mechanism found beyond fixed truncation | yes (absence) | grep across src/ and config/*.yaml | (no summarization/compaction code found) | inference | med |
| Trajectory persisted to JSON after every step (not just at end) | yes | agents/default.py:120-121,159-190 | `finally: self.save(self.config.output_path)` ... `"trajectory_format": "mini-swe-agent-1.1"` | impl | high |
| Retry: tenacity exponential backoff, default 10 attempts, explicit abort-exception list | yes | models/utils/retry.py:9-25; models/litellm_model.py:50-57 | `stop=stop_after_attempt(int(os.getenv("MSWEA_MODEL_RETRY_STOP_AFTER_ATTEMPT", "10")))` | impl | high |
| Cost tracked per-call via litellm.cost_calculator, per-agent and globally | yes | models/litellm_model.py:108-126; models/__init__.py:13-42 | `cost = litellm.cost_calculator.completion_cost(response, model=self.config.model_name)` | impl | high |
| Global cross-agent cost/call caps via env vars | yes | models/__init__.py:20-31 | `self.cost_limit = float(os.getenv("MSWEA_GLOBAL_COST_LIMIT", "0"))` | impl | high |
| Environment classes enumerated: docker, singularity, local, swerex_docker, swerex_modal, bubblewrap, contree | yes | environments/__init__.py:8-16 | `"docker": "minisweagent.environments.docker.DockerEnvironment", ... "contree": "minisweagent.environments.extra.contree.ContreeEnvironment"` | impl | high |
| No MCP or ACP references anywhere in source or docs | yes (absence) | grep -rni "mcp\|model context protocol\|agent client protocol\|\bACP\b" src/ docs/ README.md | (zero matches) | inference | high |
| InteractiveAgent has three modes: human/confirm/yolo, whitelist regex skips confirmation | yes | agents/interactive.py:24-31,162-183 | `mode: Literal["human", "confirm", "yolo"] = "confirm"` ... `whitelist_actions: list[str] = []` | impl | high |
| Default (non-interactive) DefaultAgent has no confirmation step at all | yes | agents/default.py:154-157 | `outputs = [self.env.execute(action) for action in message.get("extra", {}).get("actions", [])]` | impl | high |
| Textual-based TUI trajectory inspector exists | yes | run/utilities/inspector.py:16-19,79 | `from textual.app import App, ComposeResult` ... `class TrajectoryInspector(App):` | impl | high |
| SWE-bench batch runner uses ThreadPoolExecutor with configurable worker count | yes | run/benchmarks/swebench.py:209,257-263 | `workers: int = typer.Option(1, "-w", "--workers", ...)` ... `ThreadPoolExecutor(max_workers=workers)` | impl | high |
| Batch runner writes per-instance trajectory + aggregated preds.json (SWE-bench submission format) | yes | run/benchmarks/swebench.py:97-108,163-176 | `output_data[instance_id] = {"model_name_or_path": model_name, ... "model_patch": result}` | impl | high |
| Entry points: mini, mini-swe-agent, mini-extra, mini-e | yes | pyproject.toml:89-93 | `mini = "minisweagent.run.mini:app"` `mini-extra = "minisweagent.run.utilities.mini_extra:main"` | config | high |
| License MIT | yes | LICENSE.md:1-3; pyproject.toml:28 | `MIT License` `Copyright (c) 2025 Kilian A. Lieret and Carlos E. Jimenez` | config | high |
| 1019 commits total, active from 2025-06-28 to 2026-07-22, near-daily cadence | yes | git log --oneline \| wc -l; git log --reverse | `1019` ... `2025-06-28 Initial commit` ... 7 commits on 2026-06-29 alone | commit-history | high |
| 53 test files across agents/config/environments/models/run/utils | yes | find tests -name "*.py" \| wc -l and per-dir counts | `15 tests/models` `11 tests/run` `5 tests/agents` etc. | test | high |
| Global config setup prompts for model + API key, stored in a .env file | yes | run/utilities/config.py:62-96 | `set_key(global_config_file, "MSWEA_MODEL_NAME", default_model)` | impl | high |
