PROJECT ADJUTANT

Arcology Copilot: Local, Toggleable AI Operator (Design Target)

0) Purpose (what this copilot must make effortless)
	•	Command & filesystem operations on the local machine via natural language (and hotkeys).
	•	Continuous context from the active workspace (focused window, selected files, recent shell history, logs) with explicit consent gates.
	•	LLM inference through oobabooga (Text‑Generation WebUI) using a llama engine, with easy model switching and grammar/custom stopping rules.
	•	Full speech loop: optional push‑to‑talk → STT (later) → LLM → TTS output (via TTS‑WebUI).
	•	Hard toggle: ON/OFF (and “read‑only” mode) with visible state, integrated with the Arcology Ops interface you designed.

The end state below is non‑negotiable; the implementation is free.

⸻

1) Core Concepts (objects & behaviors)

1.1 Copilot
	•	A long‑lived agent that can receive user prompts, system events, and task assignments.
	•	Modes: off | listen-only | propose-only | execute-with-confirmation | execute-auto (permitted commands only)
	•	State: current LLM preset, tool permissions, working directory, active project context.

1.2 Tools (local capabilities)
	•	Each tool is a declared command interface the copilot may call.
	•	Examples: fs.read, fs.write, fs.diff, shell.exec, git.status, proc.list, sys.info, net.check_port, editor.patch, search.grep, doc.summarize, compress.zip, schedule.task.
	•	Contract: JSON schema for arguments, safety policy, dry‑run string, execution function.

1.3 Sandbox & Policy
	•	Sandbox: executes commands in a controlled environment (user namespace, working dir cage, allowlist binaries).
	•	Policy: allow/deny patterns, escalation rules (e.g., forbid sudo, block writes outside allowlisted paths unless explicitly granted per‑session).

1.4 Context Providers
	•	Passive, opt‑in sources: focused window path, selected files, clipboard, recent shell history (last N commands), recent logs from known services, active Git repo status, current project metadata.
	•	Each provider returns a compact bulletin; copilot can request details.

1.5 Orchestrator Binding
	•	The copilot appears in Arcology Ops as an Agent with status, latency, last tasks, memory usage, Policy chips (e.g., “Propose‑only”).
	•	Scheduler respects Copilot Persistent policy (won’t unload its model; can block GPU‑exclusive jobs unless forced).

1.6 Dialogue & Speech
	•	Input: keyboard (command palette) and optional push‑to‑talk.
	•	Output: on‑screen response + TTS audio via TTS‑WebUI.
	•	Interrupts: space/esc to stop speaking; “show plan” to reveal tool call sequence.

⸻

2) Views (UI surfaces)

2.1 Copilot Dock (always‑available panel)
	•	Mode toggle (off/listen/propose/confirm/auto)
	•	Mic button (push‑to‑talk)
	•	Current preset (LLM name, context window)
	•	Policy chips (no‑sudo, read‑only FS, project: X)
	•	Quick actions: “Summarize current folder”, “Explain last error”, “Draft commit message”

2.2 Plan & Permissions Drawer
	•	Shows the structured plan (sequence of tool calls) before execution.
	•	Per‑step grant/deny and edit arguments.
	•	One‑click Execute Plan or Run Step‑by‑Step.

2.3 Action Log & Replay
	•	Timeline of (prompt → plan → tool calls → results).
	•	Replay or Dry‑run any past action; Export bug bundle.

2.4 Settings (per‑profile)
	•	Model preset (oobabooga API target, grammar/stops, temperature, system prompt).
	•	Tool allowlist and path cages.
	•	Auto‑TTS on/off, response verbosity, token budget.
	•	Context providers: on/off per source.

⸻

3) Behavioral Rules (must hold true)
	1.	Propose → Confirm → Execute default.
	•	The copilot always drafts a plan (tool call sequence) for non‑trivial tasks.
	•	User can approve/deny/edit before execution.
	2.	Least privilege by default.
	•	Start in read‑only FS mode; writes require elevation via a visible switch or per‑action grant.
	•	No sudo in MVP; future: gated escalation with passphrase and time‑boxed token.
	3.	Dry‑run first.
	•	Every destructive tool supports dry_run() returning the exact shell/file diffs it would perform.
	4.	Deterministic logs.
	•	Each tool call is logged with args, working dir, env diff, stdout/stderr; logs are immutable and exportable.
	5.	Human in the loop for multi‑step changes.
	•	For plans >1 step that involve writes, require explicit Plan approval unless in execute-auto mode and within a safe command allowlist.
	6.	Context is explicit.
	•	Copilot may ask to import context (e.g., “Include these 3 files?”) and shows a context summary before using it.

⸻

4) Minimal Data Contracts (interfaces)

4.1 Tool declaration (registry entry)

{
  "name": "fs.write",
  "version": "1.0",
  "description": "Write text to a file (creates or overwrites).",
  "args_schema": {
    "type": "object",
    "properties": {
      "path": {"type":"string"},
      "text": {"type":"string"},
      "create_dirs": {"type":"boolean","default":false}
    },
    "required": ["path","text"]
  },
  "safety": {
    "writes": true,
    "allowed_paths": ["~/projects", "/tmp/arcology"],
    "forbid_patterns": ["/etc/**","~/.ssh/**"]
  },
  "supports_dry_run": true,
  "examples": [{"path": "~/projects/README.md", "text": "Hello"}]
}

4.2 Tool call (proposed step)

{
  "tool": "fs.write",
  "args": {"path": "~/projects/notes/todo.md", "text": "- [ ] Wire adapter"},
  "dry_run": true,
  "explain": "Create/update todo list in project notes."
}

4.3 Execution result (log line)

{
  "tool": "fs.write",
  "args": {"path":".../todo.md","text":"..."},
  "dry_run_diff": "CREATE .../todo.md (102 bytes)",
  "status": "ok",
  "stdout": "",
  "stderr": "",
  "duration_ms": 28,
  "artifacts": [{"type":"file","path":".../todo.md"}]
}

4.4 Context bulletin

{
  "cwd": "/home/user/projects/arcology",
  "git": {"branch":"main","dirty": true, "ahead": 1},
  "focus": {"window_title": "VSCode - orchestrator.py", "path": ".../orchestrator.py"},
  "selected_files": [".../orchestrator.py",".../locks.py"],
  "recent_shell": ["make start","python smoke.py"],
  "logs": [{"name":"orchestrator","tail":"[INFO] Scheduler started..."}]
}


⸻

5) LLM Integration (oobabooga)
	•	Inference host: Text‑Generation WebUI (oobabooga) running a llama engine model (Qwen2.5‑7B/Mistral‑7B quant).
	•	API: use its generation endpoint; grammar/stopping rules enforce JSON tool calls when required.
	•	Prompts:
	•	System prompt defines: role, tool calling contract, safety, confirmation policy.
	•	Assistant style: concise; prefers proposed plan + rationale + tool JSON.
	•	Switching models: through orchestrator /models/load; Copilot respects persistent policy.
	•	Streaming: show tokens live; allow interrupt.

Minimal “tool‑calling JSON” grammar (LLM must output one of):

{"type":"plan","steps":[{"tool":"shell.exec","args":{"cmd":"ls -la"}}]}

or

{"type":"response","text":"<concise answer>"}

Stop sequences: end of JSON } plus a sentinel (e.g., \n<END>).

⸻

6) TTS Output (TTS‑WebUI)
	•	After LLM response (or plan approval), send renderable text to TTS‑WebUI.
	•	Auto‑TTS toggle (on by default for short responses; off for long logs).
	•	Playback controls: play/pause/stop; “send transcript to clipboard”.

⸻

7) Tooling Surface (initial allowlist)

MVP tools with tight scopes; each must support dry_run().

	•	shell.exec — run a command; no pipes by default (args array form); timeouts; capture output.
	•	fs.read / fs.write / fs.append / fs.diff — path‑caged.
	•	git.status / git.diff / git.commit (message only, no amend in MVP).
	•	search.grep — ripgrep wrapper with path allowlist.
	•	editor.patch — apply a unified diff patch to a file; validates patch.
	•	proc.list / proc.kill (confirm; kill SIGTERM only in MVP).
	•	doc.summarize — calls Arcology task to summarize PDFs in cwd (bridges to Workflow Forge).
	•	schedule.task — creates a queued Arcology task with capability + inputs.

⸻

8) Golden Flows (must be smooth)

8.1 “Fix this error” (shell → edit → verify)
	1.	User: “The orchestrator crashes on start; help.”
	2.	Copilot: reads last 200 lines of orchestrator logs → proposes plan:
	•	fs.read(log) → search.grep("Traceback") → editor.patch(file, diff) → shell.exec("make start")
	3.	User approves; copilot runs dry‑run for patch; executes; shows result; reads logs again; confirms healthy.

8.2 “Summarize these PDFs and read it out”
	1.	Copilot ingests selected files; creates Arcology workflow task (ocr.pdf → text.summarize → audio.tts) via schedule.task.
	2.	Waits for completion (subscribes to task events).
	3.	Plays summary via TTS; offers to save summary.md.

8.3 “Create a feature branch and commit changes”
	1.	Proposes: git.status → show diff → git.checkout -b → editor.patch → git.commit -m.
	2.	Dry‑runs patch; executes; returns commit hash; no push in MVP.

8.4 “Run command and explain output”
	1.	Runs shell.exec(["nvidia-smi","--query-gpu=..."]).
	2.	Summarizes; logs artifacts; speaks brief.

⸻

9) Safety & UX Guarantees
	•	Mode indicator always visible; color‑coded (Off=gray, Propose=blue, Confirm=amber, Auto=green).
	•	Write previews (diff) before applying.
	•	Time‑boxed sessions: Auto‑downgrade from execute-auto to propose-only after N minutes idle.
	•	Path cages default to project workspace and /tmp/arcology; expanding needs a one‑click grant with path picker.
	•	No network actions in MVP (other than local APIs); outbound calls must be explicit future tools.

⸻

10) Eventing & IPC (flexible but clear)
	•	Internal event bus: copilot.events channel publishes prompt, plan, tool_call, result, error.
	•	IPC to tools: local Unix domain socket or localhost HTTP; both acceptable.
	•	Subscriptions: copilot subscribes to Arcology task events for jobs it spawned.

⸻

11) Persistence
	•	Action log: append‑only JSONL per day (timestamped).
	•	Artifacts: saved under runs/YYYY‑MM‑DD/copilot/<task-id>/.
	•	Profiles: JSON presets (model, temp, grammar, tools allowlist, cages, context providers).
	•	Golden prompts for regression: stored under copilot/golden/.

⸻

12) MVP Scope
	•	Modes: propose‑only, execute‑with‑confirmation.
	•	LLM: one preset (e.g., Qwen2.5‑7B‑Instruct Q4_K_M) via oobabooga; streaming; JSON tool‑call grammar.
	•	TTS: TTS‑WebUI integration; play/pause.
	•	Tools: shell.exec, fs.read/write/diff, search.grep, git.status/diff/commit, editor.patch, schedule.task.
	•	Context providers: cwd, focused file, git status, recent shell, orchestrator logs.
	•	UI: Dock + Plan Drawer + Log Viewer.
	•	Safety: read‑only default; dry‑run for writes; no sudo/network.

⸻

13) Acceptance Criteria
	•	I can toggle copilot mode and see state persist.
	•	Given “List Python processes,” copilot proposes and executes proc.list (read‑only), returns a summary, and speaks it.
	•	Given “Append a TODO to notes.md,” copilot proposes fs.write (diff preview), executes on approval, and logs the artifact.
	•	Given “Summarize PDFs here,” copilot schedules a workflow task via Arcology and plays back the summary when done.
	•	A malformed plan yields a clear error and a retry with simplified plan; all steps are visible in the log.
	•	Switching the model preset takes effect without restarting the copilot process (or via a single restart button).

⸻

14) Prompting Skeletons (for Codex to use)

System prompt (sketch):

You are Arcology Copilot, a local operator. You can propose multi‑step plans and call tools using JSON. Prefer minimal steps. Obey safety: read‑only default; no sudo; write only within allowed paths after user approval. Always respond in one of two forms:
	1.	{"type":"plan","steps":[...]}
	2.	{"type":"response","text":"..."}
Use the provided context bulletin when helpful. If a tool call fails, propose a revised plan.

Few‑shot (tool call):

{"type":"plan","steps":[
  {"tool":"shell.exec","args":{"cmd":["ls","-la"],"cwd":"~/projects/arcology"}},
  {"tool":"fs.read","args":{"path":"~/projects/arcology/README.md"}}
]}


⸻

15) Seed Files to include in repo
	•	/copilot/registry/tools/*.json — tool declarations (as above).
	•	/copilot/presets/default.json — model + grammar + stops + policy.
	•	/docs/copilot_examples/ — sample prompts → plans → results.
	•	/samples/context_bulletin.json — mock bulletin for testing parser.
	•	/tests/golden/ — small regression set (prompts + expected tool sequences).

⸻

16) Roadmap (post‑MVP, optional)
	•	STT for voice input (push‑to‑talk), with VAD and on‑device models.
	•	Safe sudo: ephemeral elevation token with narrowed commands (apt install, systemctl restart for specific services).
	•	Editor deep link: open at line/col for diffs (VSCode protocol).
	•	Learning memory: per‑project notes that the copilot can update under approval.
	•	RAG for codebase: local embeddings to improve explanations (still offline).

⸻

