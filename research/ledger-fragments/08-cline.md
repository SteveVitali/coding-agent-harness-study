# Ledger — 08 cline/cline @ 64993e78d562

| claim | cline | path:lines | "snippet" | class | confidence |
|---|---|---|---|---|---|
| Pinned commit verified | 64993e78d562... | (git rev-parse HEAD) | "64993e78d5623920e1fb549fb20b6d15ed9b710a" | commit-history | high |
| Monorepo with SDK + apps workspaces | yes | package.json:1-12 | "workspaces": ["sdk/packages/*", "apps/*", ...] | config | high |
| SDK packages are publishable, ship dist only | yes | sdk/packages/agents/package.json | "files": ["dist"] ... "description": "Browser-safe agent runtime for the next-generation Cline SDK" | config | high |
| @cline/core etc. actually published to npm at repo version | yes | external: npm registry, 2026-08-04 | @cline/core "latest": "0.0.69"; cline CLI 3.0.49 | external | high |
| Unified release script publishes SDK + CLI | yes | sdk/scripts/release.ts:5-17 | "bun release sdk # auto-increment patch, publish SDK ... bun release cli" | config | high |
| CI publish workflows exist for sdk, cli, desktop, vscode ext | yes | .github/workflows (ls) | sdk-publish.yml, cli-publish.yml, desktop-publish.yml, ext-vscode-publish-stable.yml | config | high |
| Legacy extension loop symbols removed | yes | apps/vscode/src (grep) | grep recursivelyMakeClineRequests / parseAssistantMessage → zero hits | impl | high |
| Extension is a host adapter over SDK | yes | apps/vscode/src/sdk (ls, ~90 files) | SdkController.ts, cline-session-factory.ts, message-translator.ts, vscode-runtime-builder.ts | impl | high |
| Extension declares SDK deps | yes | apps/vscode/package.json | ['@cline/agents', '@cline/core', '@cline/llms', '@cline/shared', ...] | config | high |
| HostProvider abstraction spans VS Code and cline-core | yes | apps/vscode/src/hosts/host-provider.ts:5-9 | "This system runs on two different platforms (VSCode extension and cline-core)" | impl | high |
| Legacy gRPC hostbridge proto surface in-tree | yes | apps/vscode/proto/host (ls) | diff.proto, env.proto, testing.proto, window.proto, workspace.proto | impl | high |
| Agent loop: iterate until no tool calls or maxIterations | yes | sdk/packages/agents/src/agent-runtime.ts:686-802 | while (this.config.maxIterations === undefined \|\| this.state.iteration < this.config.maxIterations) | impl | high |
| Tool calls are structured parts, not XML | yes | agent-runtime.ts:711-714 | message.content.filter((part): part is AgentToolCallPart => part.type === "tool-call") | impl | high |
| Completion via lifecycle.completesRun tool (submit_and_exit) | yes | agent-runtime.ts:1492-1512; core/src/extensions/tools/definitions.ts:816-818 | if (this.tools.get(toolCall.toolName)?.lifecycle?.completesRun !== true) continue; / lifecycle: { completesRun: true } | impl | high |
| submit_and_exit is attempt_completion analog | yes | sdk/ARCHITECTURE.md:129-136 | "a successful `submit_and_exit` (the SDK analog of original Cline's `attempt_completion`)" | doc | med |
| Tool approval: policy + host callback; no callback = deny | yes | agent-runtime.ts:1576-1614 | policy.autoApprove === false → requestToolApproval; "requires approval but no approval callback is configured" → approved: false | impl | high |
| Parallel tool execution supported | yes | agent-runtime.ts:1479-1483 | if (this.config.toolExecution === "parallel") return Promise.all(...) | impl | high |
| 9 built-in tools, schema'd | yes | core/src/extensions/tools/definitions.ts:254-831 | name: "read_files" ... "search_codebase" ... "run_commands" ... "fetch_web_content" ... "apply_patch" ... "editor" ... "skills" ... "ask_question" ... "submit_and_exit" | impl | high |
| Tools use JSON Schema from zod | yes | definitions.ts:621,675 | inputSchema: zodToJsonSchema(ApplyPatchInputSchema) | impl | high |
| apply_patch uses Codex-style patch envelope | yes | definitions.ts:594-604 | "*** Begin Patch\n*** Update File: src/page.tsx" | impl | high |
| editor = exact unique-match replace, no fuzzy | yes | core/src/extensions/tools/executors/editor.ts:181-195 | if (occurrences === 0) throw ...; if (occurrences > 1) "multiple occurrences of text found" | impl | high |
| Shell executor = child_process.spawn with timeout kill | yes | executors/bash.ts:4-7,127-227 | "Built-in implementation for running shell commands using Node.js spawn" ... TimeoutError(`Command timed out after ${timeoutMs}ms`) | impl | high |
| Search uses ripgrep with fallback | yes | executors/search.ts:4,123-139 | "using ripgrep (if available) or regex" ... spawn("rg", ["--version"]) | impl | high |
| Tool output truncation caps at 48k chars | yes | executors/output-limits.ts:18-50 | MAX_COMMAND_OUTPUT_CHARS = 48_000 ... MAX_READ_LINES = 2_000 | impl | high |
| VS Code replaces run_commands with terminal-integrated tool | yes | apps/vscode/src/sdk/vscode-run-commands-tool.ts:1-12; vscode-session-host.ts:102-116,164 | "Custom `run_commands` tool that replaces the SDK's built-in version ... Foreground (vscodeTerminal) ... Background (backgroundExec)" | impl | high |
| VS Code replaces editor executor with diff-view pipeline | yes | vscode-session-host.ts:58-61 | "Custom `editor` tool executor (diff-view edit pipeline). Fully replaces the SDK's" | impl | high |
| System prompt is tiny with rules slots | yes | sdk/packages/shared/src/prompt/system.ts:1-35 (file 6636 bytes) | "You are Cline, an AI coding agent..." ... {{CLINE_RULES}}{{CLINE_METADATA}} | impl | high |
| Separate YOLO system prompt for unattended runs | yes | system.ts (YOLO block) | "You should only end the task when all the requirements are met by calling the 'submit_and_exit' tool" | impl | high |
| Plan/act/yolo are tool presets (tool gating) | yes | core/src/extensions/tools/presets.ts:20-126 | plan: { ...enableEditor: false ... }; resolveToolPresetName({mode}) | impl | high |
| Plan mode keeps bash enabled, prompt-enforced only | yes | presets.ts:43-46; shared/src/prompt/cline.ts:26-30 | plan comment "read-only, no shell access" yet enableBash: true; "the mitigation for plan-mode mutations is prompting plus mode-switch notices, not tool removal" | impl | high |
| Mode provenance via user_input mode tags | yes | cline.ts:21-23 | `User messages arrive wrapped in a <user_input mode="..."> tag` | impl | high |
| Yolo preset auto-approves all tools incl. wildcard | yes | presets.ts:137-158 | policies["*"] = { enabled: true, autoApprove: true } | impl | high |
| SDK defaults unlisted tools to auto-approved; extension forces callbacks | yes | apps/vscode/src/sdk/sdk-tool-policies.ts:7-40 | "The SDK defaults unlisted tools to auto-approved." ... set(["run_commands", "execute_command"]) | impl | high |
| Per-task auto-approval settings flow from webview proto | yes | apps/vscode/src/core/controller/task/newTask.ts:16-45 | NewTaskRequest ... autoApprovalSettings ... actions: {...globalSettings.actions, ...incomingSettings.actions} | impl | high |
| Webview↔extension is gRPC-shaped over postMessage | yes | apps/vscode/src/core/controller/grpc-handler.ts:23-66 | if (response?.grpc_response) ... handleStreamingRequest(controller, postMessageWithRecording, request) | impl | high |
| Providers: ~150 generated IDs | yes | llms/src/providers/provider-ids.generated.ts | "anthropic" "bedrock" ... "openrouter" ... "zhipuai-coding-plan" (163-line file) | impl | high |
| ~9 vendor implementation families | yes | llms/src/providers/vendors (ls) | anthropic.ts, bedrock.ts, google.ts, mistral.ts, ollama.ts, openai.ts, openai-compatible.ts, vertex.ts, community.ts | impl | high |
| Execution via Vercel AI SDK streamText | yes | llms/src/providers/ai-sdk.ts:23-25,1247 | import { streamText } from "ai" ... stream = streamText({ | impl | high |
| Cost computed incl. BYOK upstream inference cost | yes | ai-sdk.ts:740-760 | const totalCost = marketCost ?? (shouldAddUpstreamCost ? baseCost + upstreamInferenceCost : costOrUpstream) | impl | high |
| Prompt caching handled in anthropic-compatible routing | yes | llms/src/providers/routing/anthropic-compatible.ts:132,222 | "cache_control remains on the content part" / "OpenAI-compatible cache_control shape used by the new routing path" | impl | med |
| Reasoning stream + token accounting | yes | agent-runtime.ts:222,361-383 | case "reasoning": ... reasoningTokenCount = Math.max(...) | impl | high |
| Context overflow: compact once, retry, else terminal | yes | agent-runtime.ts:58-73,664,699-700 | CONTEXT_WINDOW_OVERFLOW_RECOVERY_FAILED_MESSAGE ... overflowRecoveryAttempted = false ... generateAssistantMessageWithOverflowRecovery() | impl | high |
| Compaction default "agentic" with preserveRecentTokens + deterministic fallback | yes | core/src/extensions/context/compaction.ts:48,167-170,285-290 | "deterministic basic strategy — recovery must not depend on another" ... strategy = userCompaction?.strategy ?? "agentic" | impl | high |
| Compaction state persisted separately from canonical transcript | yes | core/src/services/session-artifacts.ts:85 | `${sessionId}.compaction.json` | impl | high |
| Sessions persisted via SQLite manifests + file artifacts | yes | core/src/session/stores/session-manifest-store.ts:45-52; shared/src/db/sqlite-db.ts:144 | sessionRowFromManifest(...) / wrapBunDb(new Database(filePath, { create: true })) | impl | high |
| Checkpoints = real-repo git plumbing under private refs | yes | core/src/hooks/checkpoint-hooks.ts:111-153,250-263 | runGitWithIndex(cwd, indexFile, ["write-tree"]) ... "refs/cline/checkpoints/${sessionId}/" | impl | high |
| Checkpoint restore destructive w/ stash transaction | yes | core/src/session/checkpoint-restore.ts:43-113 | "checkpoint restoration runs `git clean -fd`" ... refs/cline/restore-transactions/ ... "Workspace snapshot and rollback both failed" | impl | high |
| SDK core MCP client: stdio-only, protocol 2024-11-05, hand-rolled | yes | core/src/extensions/mcp/client.ts:44,195-204 | MCP_PROTOCOL_VERSION = "2024-11-05" ... `Unsupported MCP transport ... ${this.registration.transport.type}` | impl | high |
| Extension MCP: official SDK, stdio/SSE/streamableHTTP | yes | apps/vscode/src/services/mcp/McpHub.ts:7-9,474-484 | import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js" | impl | high |
| MCP config = mcpServers JSON, atomic writes | yes | core/src/extensions/mcp/config-loader.ts:207-212,264 | mcpServers: z.record(z.string(), mcpRegistrationBodySchema) ... "Atomically write the MCP settings file" | impl | high |
| ACP agent implemented in CLI | yes | apps/cli/src/acp/acpAgent.ts:24-25; apps/cli/src/main.ts:799-801 | from "@agentclientprotocol/sdk" ... if (args.acpMode) { const { runAcpMode } = await import("./acp/index") | impl | high |
| CLI headless one-shot with yolo/json/piped stdin | yes | apps/cli/src/main.ts:780-791,958-969 | "In headless mode (yolo / json / piped stdin without --tui)" ... isHeadless | impl | high |
| Evals run against headless CLI on $PATH | yes | evals/README.md:5 | "still works against whatever `cline` is on `$PATH` (install with `npm i -g cline`)" | doc | high |
| AGENTS.md + legacy .clinerules discovery | yes | core/src/extensions/config/user-instruction-config-loader.ts:483-501,569; shared/src/storage/paths.ts:34-44 | const agentsPath = join(directoryPath, "AGENTS.md") ... fileName === ".clinerules" \|\| ... DEPRECATED_CONFIG_DIR = ".clinerules" | impl | high |
| Skills = SKILL.md + YAML frontmatter, exposed via skills tool | yes | user-instruction-config-loader.ts:24,206; definitions.ts:737-767 | SKILL_FILE_NAME = "SKILL.md"; frontmatterRegex = /^---\r?\n.../ ; "Available skills: ..." dynamic description | impl | high |
| Subprocess hook files can mutate prompt/messages | yes | core/src/hooks/hook-file-hooks.ts:17-35 | runSubprocessEvent ... HookCommandControl { systemPrompt?; appendMessages? } | impl | high |
| spawn_agent subagent tool exists | yes | core/src/extensions/tools/team/spawn-agent-tool.ts:121 | name: "spawn_agent" | impl | high |
| Focus chain = per-task markdown todo file | yes | apps/vscode/src/core/task/focus-chain/file-utils.ts:9-11 | path.join(taskDir, `focus_chain_taskid_${taskId}.md`) | impl | high |
| ARCHITECTURE.md references nonexistent loop files (doc drift) | contradiction | sdk/ARCHITECTURE.md:497-515 vs agents/src (ls) | doc: "Start: `packages/agents/src/agent.ts`"; ls shows only agent-runtime.ts, index.ts + tests | doc | high |
| Standalone Agent construction from provider IDs (embeddability) | yes | agent-runtime.ts:108-127 | "This is the entry point most standalone users want." providerId/modelId/apiKey/baseUrl | impl | high |
| Hub daemon auto-spawned by CLI; sessions shareable | yes | AGENTS.md (root, CLI section); core/src/hub/server (ls) | "auto-spawns the `@cline/cline-hub` daemon" / browser-websocket.ts, command-transport.ts, handlers/ | doc+impl | med |
| No OS sandbox for agent actions | absent | sdk/, apps/ (greps: no seccomp/sandbox-exec/container exec path) | approval callbacks + ToolPolicy only | inference | med |
| Repo age and cadence | active | git log | first commit 2024-07-05; 2,460 commits since 2026-01-01 | commit-history | high |
