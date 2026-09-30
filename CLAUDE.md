# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Happy Friday Lite: an Electron + Vue 3 desktop knowledge assistant (AI agent "Friday", RAG knowledge base, TipTap notes, drawing canvas, calendar, automation tasks, and an embedded office suite). Everything is stored locally. Code comments, log messages, and many UI strings are in Chinese. Keep new comments in the same style as the file you're editing.

## Commands

```bash
npm install --legacy-peer-deps   # required flag; postinstall runs patch scripts in scripts/
npm run electron:dev             # Vite dev server on :5174 (strictPort) + Electron
npm run dev                      # Vite only (renderer without Electron; window.electronAPI is absent)
npm run build                    # Vite build -> dist/ (console/debugger are stripped)
npm run electron:build           # Linux x64 package -> release/
npm run electron:build:arm64
npm run office:build             # builds vendor/happyoffice (git submodule) into resources/office/
```

The repo has no test suite and no linter. To check a change, run `npm run build` for renderer compile errors and `npm run electron:dev` for runtime behavior. `OFFICE_SMOKE=<file> npm run electron:dev` opens that file in the office host at startup and logs a probe result.

`electron:build*` scripts target **Linux** only. CI (`.github/workflows/build.yml`, manual dispatch) builds Linux, Windows, and macOS. It deletes `.npmrc` (Chinese npm mirrors) and installs with `npm ci --legacy-peer-deps --force`.

In dev, app data (SQLite `friday.db`, `config.json`, logs, knowledge files, agent memory) lives in `./app-data/`. In packaged builds it lives in Electron `userData`.

## Architecture

### Process layout
- `main.js`: Electron main entry (ESM). Creates the window, calls `initDb()`, then `registerCommands()`. After that it starts subsystems without awaiting them: automation scheduler, office host, knowledge-dir watcher, Python env, RAG, backup and history-clean checks, LAN share server, and the local MCP server. It also has a headless office-export path (`isHeadlessExportRun`) that skips all normal init.
- `preload.cjs`: exposes `window.electronAPI` with **hard-coded allowlists** for `invoke`, `send`, and `on` channels. **Every new IPC channel or push event must be added here**, or it is rejected or silently ignored.
- `src-electron/`: main-process backend (plain JS, ESM).
- `src/`: Vue 3 renderer (Pinia stores in `src/store/modules`, routes in `src/router`, `@` → `src`). All IPC goes through `src/services/electron.js` (`electronService.invoke/listen/send`). `invoke` swallows errors and returns `null`.

### IPC
- `src-electron/commands.js` registers most `ipcMain.handle` channels and delegates to feature modules. Agent channels live in `agent/ipc.js` (`registerAgentCommands`), harness channels in `harness/index.js`, and office channels in `office/office-ipc.js`.
- Push-event names are constants in `src-electron/events.js`. Streaming chat uses `chat-chunk` / `chat-reasoning-chunk` / `chat-done` / `chat-error`. The agent reuses those channels and adds `agent-tool-call` / `agent-tool-result` / `agent-tool-approval`.

### Persistence
- `src-electron/db.js` uses **sql.js (WASM, in-memory)**, which is exported and written to `friday.db` on disk. Schema creation and migrations live in this file. Writes must be followed by a save or persist (see `queryAllRaw`, `flushDb`, `closeDb`).
- `src-electron/config.js` handles `config.json` (model provider settings etc.). Models are any OpenAI-compatible endpoint (`openaiUrl.js` normalizes base URLs, `llm.js` handles plain chat, note AI, and FIM).

### Agent (`src-electron/agent/`)
- Built on `deepagents` `createDeepAgent` + LangChain/LangGraph (`agent/index.js`). A shared `MemorySaver` checkpointer enables HITL pause and resume. `InMemoryStore` is synced to SQLite (`memory.js`). Memory markdown files (SOUL/USER/MEMORY/Agent.md, `memoryFiles.js`) are re-read from disk on each invoke rather than passed via the SDK `memory:` option.
- **Adding a tool:** create a file in `agent/tools/builtin/` that calls `registerTool({ name, description, schema (zod), handler, meta: { requireApproval } })`, then add an `import` line in `agent/tools/index.js`. `registry.js` wraps tools with logging and audit and builds `interruptOn` from `requireApproval`. Tools can carry a `riskAssessment` argument that is stripped before the handler runs.
- `humanInTheLoop.js` handles approval requests and resumes. `permissions.js` sets filesystem rules. `skills.js` loads SKILL.md directories (bundled examples in `public/skills/`). `subagents.js` defines preset subagents. `mcp.js` handles external MCP clients plus an optional local MCP server that exposes Friday's tools.

### RAG (`src-electron/rag/`)
Pipeline: `loaders.js` (pdf/docx/xlsx/epub/html/md/…) → `chunkers.js` (parent-child chunks; notes use structure-aware chunking) → `embeddings.js` → `vectorstore.js` (Zvec, one store split by `kb_type`). Parent chunks go in the SQLite `parent_docs` table. `queue.js` runs indexing tasks and `triggers.js` (`initRag`) wires in scheduled and startup updates. Updates are incremental against the `file_status` table. `fileWatcher.js` pushes `kb-directory-changed` to the renderer. On Linux the renderer must call `kb-watch-current-dir` because recursive watch isn't supported there.

### Other subsystems
- `harness/`: spawns a DeepSeek harness sidecar (`@deepseek-ai/dsh`) and bridges Friday's tools to it through the local MCP server. The Harness view is `src/views/harness/`.
- `office/`: hosts HappyOffice editors (docs/sheets/slides/pdf) as `WebContentsView`s attached to the main window. Prebuilt bundles are loaded from `resources/office/`, which `npm run office:build` generates from `vendor/happyoffice`. `office-ai-bridge.js` / `office-agent.js` connect the editors' AI sidebar to Friday.
- `automation.js`: scheduled agent tasks. `python-env.js` / `python.js`: user-provided Python for the `python_repl` tool (`python/requirements.txt`). `shareServer.js`: read-only LAN HTTP share of conversations, notes, and drawings. `backup.js` + `backup-worker.js`: backups in a worker thread.

### Renderer notes
- i18n: each feature has its own JSON under `src/i18n/locales/{zh-CN,en-US}/`. Add keys to both locales.
- `src/config/prompts.js` is also used by the main process and is listed explicitly in electron-builder `files`.

## Gotchas
- `package.json` `build.files` is an explicit allowlist with many exclusions (`asar: false`). New main-process dependencies or files outside `src-electron/` may need entries there, or the packaged app will be missing them.
- `scripts/fix-langgraph-sdk.mjs` and `scripts/fix-dsh-directory-picker-electron.mjs` patch `node_modules` and run on postinstall, dev, and start. Re-run them if you reinstall a single package.
- Native modules (`@zvec/*`, `sharp`) are platform-specific. The `prebuild-*` and `verify-packaged-native-modules` scripts handle arch-specific packaging.
