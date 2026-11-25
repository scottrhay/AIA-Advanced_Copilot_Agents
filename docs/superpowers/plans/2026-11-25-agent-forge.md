# Agent Forge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Local web app that turns a prompt into a full M365 Copilot declarative agent (ATK JSON project) with capabilities, API plugins, runtime MCP actions, skills and Adaptive Cards, and creates + shares it in one click.

**Architecture:** npm-workspaces monorepo. `server/` (Fastify, TypeScript) owns files on disk, schema validation, Claude calls, MCP discovery and `atk` process execution. `web/` (Vite + React + Fluent UI v9) is the UI and talks to the server over REST plus SSE for streaming logs. `shared/` holds the types both sides use.

**Tech Stack:** Node 22, TypeScript 5, Fastify 5, Ajv 8 (draft 2020-12 + formats), `@anthropic-ai/sdk`, `@modelcontextprotocol/sdk`, `adaptivecards` + `adaptivecards-templating`, React 19, Fluent UI React v9, Monaco editor, Vitest, Playwright (one smoke test).

**Spec:** `docs/superpowers/specs/2026-11-25-agent-forge-design.md`

## Global Constraints
- Localhost only; the server binds to `127.0.0.1:5179`, and the web dev server is `5173`, proxying `/api`.
- Agents workspace default: `C:\AIA-Code\CopilotAgents\agents\<agent-slug>\`; each agent is a standard ATK declarative-agent project (`appPackage/declarativeAgent.json`, `appPackage/manifest.json`, `m365agents.yml`, `env/`).
- Supported declarative agent schema versions: `v1.6`, `v1.7`, `v1.8`, `v1.9`; default `v1.9`. Skills require `v1.9`.
- Instructions max length: 8,000 characters.
- Adaptive Card schema version: `1.5`.
- Default LLM model `claude-opus-5-5`, configurable in Settings.
- Secrets (Anthropic key, plugin API keys, OAuth client secrets) come from KeyVault via the `keyvault` skill's CLI path; they are injected only into the child-process env and are never written to `.env*` files.
- Unknown JSON fields must survive a read→write round-trip.
- Preview or license-gated features are always labelled in the UI.

## Review Focus
1. Prompt that asks for a feature the selected schema version lacks (e.g. skills on v1.8) → generator must bump the version or drop the feature with a visible warning, never emit invalid JSON. *Test in Task 5.*
2. A file edited in VS Code after the app loaded it → save must refuse with a 409 conflict, not overwrite. *Test in Task 3.*
3. `atk` not installed or not signed in → Create shows an actionable message and doesn't hang. *Test in Task 10.*
4. MCP server unreachable or requires OAuth → discovery returns a typed error the UI shows; no half-written plugin file. *Test in Task 7.*
5. A card template that references fields missing from the response schema → validator flags the unbound `${...}` paths. *Test in Task 6.*

---

### Task 0: Spike — sharing and MCP card support (throwaway, findings only)

**Files:** Create `docs/spikes/2026-11-25-share-and-mcp.md`

- [ ] Run `atk -h`, `atk install -h` and check for `atk share`. Record the exact commands and flags.
- [ ] Provision a hello-world DA to the dev tenant with `atk install --scope Shared`. Determine how to add named users/groups: CLI, then Graph `/appCatalogs` / Teams Dev Portal API, then a deep link. Record which calls worked, the required permissions, and how to get the chat link (`https://m365.cloud.microsoft/chat/?titleId=...` or similar).
- [ ] Check the Learn docs and samples for whether `RemoteMCPServer` runtime functions support `response_semantics` / `confirmation`. Record yes/no with the source URL.
- [ ] Write findings: for each popup feature (add user, add group, role, org-wide, notify, copy link, list access), mark it "automatable via X" or "deep link". Tasks 7 and 11 follow this file.

### Task 1: Scaffold monorepo

**Files:** `package.json` (workspaces `shared`,`server`,`web`), `tsconfig.base.json`, `server/src/index.ts`, `web/` (Vite React TS), `shared/src/types.ts`, `.gitignore` (`node_modules`, `dist`, `agents/*/env/.env.*.user`)

- [ ] `git init`; create the workspaces. `npm run dev` starts both with `concurrently`. `npm test` runs Vitest in all workspaces.
- [ ] Test `server/test/health.test.ts`: `GET /api/health` → `200 {ok:true}`. Run it, see it fail, implement it, see it pass.
- [ ] Commit `chore: scaffold agent forge`.

### Task 2: Schemas + validator

**Files:** `server/schemas/declarative-agent/v1.6..v1.9.json`, `server/schemas/plugin/v2.2.json`, `v2.3.json` (latest available), `server/schemas/app-manifest/v1.22.json` (+latest), `server/schemas/adaptive-card/1.5.json`, `server/src/validate.ts`, `server/test/validate.test.ts`

**Produces:**
- `type SchemaKind = 'declarativeAgent'|'plugin'|'appManifest'|'adaptiveCard'`
- `validate(kind: SchemaKind, version: string, doc: unknown): ValidationIssue[]` where `ValidationIssue = {path: string; message: string; severity: 'error'|'warning'}` (in `shared/src/types.ts`)

- [ ] Download the official schemas (URLs are linked from each Learn manifest reference page) and vendor them. Record each source URL in `server/schemas/SOURCES.md`.
- [ ] Tests: a minimal valid v1.9 DA → `[]`. A DA missing `instructions` → one error at path `/instructions`. A v1.8 DA containing a skills entry → an error. Instructions of 8,001 characters → an error with message containing `8000`.
- [ ] Implement with Ajv 2020 and compiled-validator caching. Add the 8,000-char check as a custom rule if the schema lacks it.
- [ ] Commit.

### Task 3: Project store (lossless, conflict-safe)

**Files:** `server/src/projectStore.ts`, `server/src/routes/agents.ts`, `server/test/projectStore.test.ts`

**Produces:**
- `listAgents(): Promise<AgentSummary[]>` (`{slug,name,schemaVersion,updatedAt,deployed:boolean}`)
- `readAgent(slug): Promise<AgentProject>` where `AgentProject = {slug; files: Record<string, unknown>; hashes: Record<string,string>; meta: AgentMeta}`, keyed by relative path (`appPackage/declarativeAgent.json`, ...)
- `writeAgent(slug, files, baseHashes): Promise<{hashes}>` throws `ConflictError` when the on-disk sha256 ≠ baseHash
- `createAgentSkeleton(slug, name, schemaVersion): Promise<AgentProject>`, which writes the ATK DA template (`m365agents.yml` with `teamsApp/create`, `teamsApp/zipAppPackage`, `teamsApp/validateAppPackage`, `teamsApp/update`, `teamsApp/extendToM365` steps)
- `AgentMeta` stored at `<agent>/.forge/meta.json`: `{skillRefs: SkillRef[], sharing?: ShareState, lastCreatedAt?}`
- REST: `GET /api/agents`, `GET /api/agents/:slug`, `PUT /api/agents/:slug` (409 on conflict), `POST /api/agents/:slug/duplicate`, `DELETE /api/agents/:slug` (moves to `agents/.trash/`)

- [ ] Tests: round-trip keeps the unknown field `{"x_custom":1}`. Writing with a stale hash after an external `fs.writeFile` → `ConflictError` (Review Focus 2). Delete moves to `.trash`.
- [ ] Implement. JSON is written with 2-space indent and a trailing newline.
- [ ] Commit.

### Task 4: Capability catalog

**Files:** `shared/src/capabilities.ts`, `server/test/capabilities.test.ts`

**Produces:** `CAPABILITIES: CapabilityDef[]`, where `CapabilityDef = {name: string; minVersion: SchemaVersion; status:'ga'|'preview'; license:'none'|'copilot'; fields: FieldDef[]; docUrl: string}`. Also `availableCapabilities(version, tenantToggles): CapabilityDef[]`.

- [ ] Fill it from the v1.6–v1.9 manifest reference pages: WebSearch, OneDriveAndSharePoint, GraphConnectors, GraphicArt, CodeInterpreter, Dataverse, TeamsMessages, Email, People, ScenarioModels, Meetings, EmbeddedKnowledge, plus any others the pages list. `fields` drives the editor forms (e.g. `items_by_url[].url`).
- [ ] Test: `availableCapabilities('v1.6', {})` excludes every capability whose `minVersion > v1.6`. Preview items are excluded unless the toggle is on.
- [ ] Commit.

### Task 5: Prompt → agent generator

**Files:** `server/src/secrets.ts`, `server/src/llm.ts`, `server/src/generate/spec.ts`, `server/src/generate/emit.ts`, `server/src/routes/generate.ts`, `server/test/generate.test.ts`

**Produces:**
- `getSecret(name: string): Promise<string>`, the KeyVault wrapper, with an in-memory cache only
- `AgentSpec` (in shared): `{name, description, instructions, conversationStarters[], capabilities: {name, config}[], apiPlugins: ApiPluginSpec[], mcpServers: McpServerSpec[], skills: string[], schemaVersion, behaviorOverrides?, disclaimer?}`
- `draftSpec(prompt: string, ctx: {version, toggles, skillLibrary}): Promise<AgentSpec>`, a Claude tool-use call with the tool `emit_agent_spec`, whose input_schema is AgentSpec's JSON schema
- `emitProject(spec): Record<string, unknown>`, deterministic files from the spec
- `generateAgent(prompt, opts): Promise<{project: AgentProject; warnings: string[]}>`: draft → emit → validate → if errors, one repair call with the issues → if still invalid, return the project with the issues (never silently drop)
- `POST /api/generate` `{prompt, schemaVersion}` → SSE of progress events, then the final `{slug, warnings}`

- [ ] Tests with a mocked LLM: a spec asking for skills on v1.8 → the emitted version is v1.9 plus a warning "schema bumped to v1.9 for skills" (Review Focus 1). An invalid first draft → a repair call is made exactly once. `emitProject` is a pure function, covered by a snapshot test.
- [ ] Implement. The system prompt includes the capability catalog for the selected version and the card design guide (Task 6).
- [ ] Commit.

### Task 6: Adaptive Cards

**Files:** `server/src/cards/designGuide.md`, `server/src/cards/templates/{detail,list,approval,status,kpi}.json`, `server/src/cards/generate.ts`, `server/src/cards/check.ts`, `web/src/components/CardPreview.tsx`, tests

**Produces:**
- `generateCard(fn: {name, responseSchema?: JSONSchema, sample?: unknown, kind:'result'|'confirmation'}): Promise<AdaptiveCardTemplate>`
- `checkBindings(template, responseSchema): ValidationIssue[]` lists `${path}` references missing from the schema (Review Focus 5) and elements not in AC 1.5
- `<CardPreview template data />` renders with `adaptivecards-templating` → `adaptivecards`
- Response semantics builder: `toResponseSemantics(template, dataPath, props): object`

- [ ] Tests: `${title}` bound against a schema without `title` → a warning at that path. An `Action.Execute` element → flagged. A generated list card for an array schema uses `$data`.
- [ ] Implement. The design guide lists the allowed elements and layouts from the spec.
- [ ] Commit.

### Task 7: MCP runtime actions

**Files:** `server/src/mcp/discover.ts`, `server/src/mcp/emitPlugin.ts`, `server/src/routes/mcp.ts`, tests with a local stub MCP server

**Produces:**
- `discoverMcp(url: string, auth?: {type:'none'|'oauth'|'apiKey'; ...}): Promise<{tools: McpTool[]} | {error: 'unreachable'|'auth_required'|'protocol', detail}>`, trying streamable HTTP first, then SSE
- `emitMcpPlugin(server: McpServerSpec, tools: McpTool[], cards): {pluginFile: object; pluginPath: string}`, a plugin manifest with a `RemoteMCPServer` runtime (exact shape from the Learn reference; record the URL in code) plus `functions[]`. Cards are attached if Task 0 says it's supported; otherwise the UI offers "Convert to OpenAPI plugin" (`mcpToOpenApi(tools)`).
- `POST /api/mcp/discover`

- [ ] Tests: the stub server with 2 tools → 2 functions. An unreachable URL → `{error:'unreachable'}` and no file written (Review Focus 4). A 401 → `auth_required`.
- [ ] Commit.

### Task 8: API plugins from OpenAPI

**Files:** `server/src/apiPlugin/import.ts`, `server/src/apiPlugin/emit.ts`, tests

**Produces:** `importOpenApi(specText: string): ApiPluginSpec` (operations, schemas, auth), and `emitApiPlugin(spec, cards): Record<string, unknown>`, which writes `appPackage/<name>-plugin.json` and `appPackage/<name>-openapi.yaml`, adds an `actions[]` entry to the DA, and sets the auth `reference_id` to `${{<NAME>_REGISTRATION_ID}}`. Auth registration goes through an `oauth/register` / `apiKey/register` step in `m365agents.yml`, with the secret supplied via the env of the `atk` process.

- [ ] Tests: a petstore spec → one function per operation, each with `response_semantics`. POST/PUT/DELETE operations get `confirmation` cards. Write operations are flagged.
- [ ] Commit.

### Task 9: Skills library

**Files:** `server/src/skills/library.ts`, `server/src/skills/compose.ts`, `server/src/routes/skills.ts`, tests

**Produces:**
- Library at `C:\AIA-Code\CopilotAgents\skills\<skill>\SKILL.md` (+ resources); `listSkills()`, `saveSkill()`, with versions as sha256 of the folder
- `SkillRef = {name, pinnedVersion?}`
- `composeSkills(project, refs): Record<string, unknown>` copies each referenced (or pinned) skill into `appPackage/skills/<name>/` and registers it in the DA the way `atk add skill` does (match its output; verify by running `atk add skill` once in a temp project). Requires v1.9.
- The build fails with `ValidationIssue` if the composed instructions exceed 8,000 characters.

- [ ] Tests: an unpinned ref picks up a library edit on the next compose. A pinned ref doesn't. A v1.8 project → error.
- [ ] Commit.

### Task 10: ATK runner + Create

**Files:** `server/src/atk/run.ts`, `server/src/atk/status.ts`, `server/src/routes/create.ts`, tests with a fake `atk` shim on PATH

**Produces:**
- `atkStatus(): Promise<{installed: boolean; version?: string; signedIn: boolean; account?: string}>` (parses `atk -v`, `atk auth list`)
- `runAtk(args: string[], cwd, env): AsyncIterable<LogLine>` with a timeout of 10 min and kill on cancel
- `POST /api/agents/:slug/create` → SSE: steps `validate` → `compose skills` → `provision` (`atk provision --env dev`) → `install shared` (per Task 0) → `done {titleId, chatLink}`, then saves `meta.lastCreatedAt`
- `POST /api/agents/:slug/publish` → `atk publish --env dev`, status `submitted`

- [ ] Tests: the shim reports not installed → the first event is an `error` with `code:'atk_missing'` and an install command (Review Focus 3). The shim is not signed in → `code:'atk_signin'` and the `atk auth login m365` hint. A successful shim run → the step events arrive in order.
- [ ] Commit.

### Task 11: Share

**Files:** `server/src/share/*.ts` (mechanism per Task 0), `server/src/routes/share.ts`, `web/src/components/SharePopup.tsx`, tests

**Produces:**
- `ShareState = {entries: {id, displayName, kind:'user'|'group', role:'chat'|'edit'}[]; orgWide: boolean; chatLink: string}`
- `GET/PUT /api/agents/:slug/share`, `POST /api/agents/:slug/share/people` `{principals, role, notify}`
- The popup matches the spec. Groups can't be set to `edit` (disabled in the UI and rejected by the server with 400). Every feature Task 0 marked "deep link" renders as a labelled button opening that link.

- [ ] Tests: adding a group with `edit` → 400. Copy link returns `chatLink`. UI test: the popup opens automatically after a `done` create event.
- [ ] Commit.

### Task 12: UI screens

**Files:** `web/src/pages/{Library,NewAgent,Editor,Skills,Settings}.tsx`, `web/src/editor/tabs/{Overview,Instructions,Starters,Capabilities,Actions,Mcp,Skills,Cards,RawJson,Deploy}.tsx`, `web/src/api.ts`, `web/e2e/smoke.spec.ts`

- [ ] Library: cards per agent with Open / Duplicate / Delete / Share / Create.
- [ ] NewAgent: prompt box, version picker, streamed progress, then the Editor.
- [ ] Editor: form tabs driven by `CAPABILITIES[].fields`, plus live `validate` issues inline (debounced server call). Advanced fields collapsed. Preview badges. Raw JSON in Monaco. On a 409 it shows a "file changed on disk — reload / overwrite" dialog.
- [ ] Settings: workspace path, model, preview toggles, `atkStatus` display.
- [ ] Playwright smoke test: create from a mocked prompt → editor shows instructions → Create (shim) → share popup visible.
- [ ] Commit.
