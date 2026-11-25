# Agent Forge — Design

Date: 2026-11-25 · Owner: Scott · Status: draft for approval

## Intent / why
A personal power tool for building and deploying Microsoft 365 Copilot **declarative agents** fast. It sits between Copilot Agent Builder (quick but exposes only a subset of the manifest) and Copilot Studio (powerful but heavy). Describe an agent in plain language → get a complete, valid, deployable agent that uses every manifest feature that fits → refine in a UI → deploy to the tenant in one click.

## Goals
- Prompt → full ATK project draft (declarative agent manifest, app manifest, icons, API plugins, MCP actions) in under a minute.
- Expose the **entire** declarative agent schema (v1.6–v1.9, version selectable per agent): instructions, conversation starters, all capabilities and their scoping (e.g. SharePoint `items_by_url`/`items_by_sharepoint_ids`, Graph connector `connections`, Email/Teams scoping, Dataverse, People, Meetings, CodeInterpreter, GraphicArt, WebSearch `sites`, ScenarioModels, EmbeddedKnowledge), actions, `behavior_overrides`, `disclaimer`, `worker_agents`, `sensitivity_label`, user overrides — whatever the chosen schema version allows.
- API plugins: generate plugin manifest + OpenAPI spec from a description or an imported OpenAPI file; configure auth (none / API key / OAuth via ATK registration), response semantics and adaptive cards.
- MCP connectors: add an MCP server URL, discover its tools, generate the corresponding plugin/action entries.
- Skills: author reusable instruction/knowledge "skills" in the app's library and compose them into an agent (merged into instructions/knowledge/actions at build time).
- Validate continuously against the official JSON schema; show errors inline.
- Lifecycle via `atk` CLI: validate, package, provision, sideload/preview, publish. Show CLI output in the UI.
- Library of all agents (local projects) with list, open, duplicate, delete, re-deploy.

## Adaptive cards (Scott's requirement)
Agents should use well-designed Adaptive Cards to be much more functional.
- Every API plugin and MCP function gets cards by default: `response_semantics` (`data_path`, `properties` title/subtitle/url, `static_template`) for results, and `confirmation` cards for any function that writes or changes data.
- Claude generates the card from the function's response schema (or a sample response), following a built-in design guide: clear hierarchy, ColumnSets/FactSets for records, a list layout for arrays, `Action.OpenUrl` links back to the source, and nothing that Copilot won't render. Templates use `${}` binding and target the card schema version Copilot supports (1.5).
- Card editor tab: live preview (the `adaptivecards` + `adaptivecards-templating` libraries, rendered against sample data), raw JSON, regenerate-with-prompt, and a reusable card template library (detail, list, approval, status, KPI).
- Validated against the Adaptive Card schema; unsupported elements are flagged.

## Create & Share (Scott's requirement)
- One **Create** button: validate → package → provision → install in **Shared** scope (`atk install --scope Shared`, which is documented) → the share popup opens straight away, as in Agent Builder.
- The share popup: add people or security groups; role **Can chat** / **Can edit** (groups can only be Can chat); org-wide sharing toggle; "send notification" option; **Copy chat link**; list of everyone with access and their role, with role changes and removal. The popup can be reopened later from the Library.
- **Risk / first spike:** there is no confirmed public mechanism yet for sharing an ATK-provisioned agent with named users. Check in this order: an `atk share` command in the installed CLI → the Shared-scope install plus Teams Developer Portal / Graph app-catalog APIs → fall back to org catalog publish. The popup's behaviour depends on what the spike proves; whatever can't be automated is shown honestly in the popup (for example, a deep link that finishes the step in M365).

## Non-goals
- Copilot Studio topics, flows, custom-engine agents.
- Multi-user, hosting, auth to the app itself (localhost only).
- TypeSpec authoring (possible later export).
- Work IQ or any paid API beyond the LLM used for drafting.

## Anti-goals
- Becoming as heavy as Copilot Studio: the default path must be prompt → review → deploy, with advanced fields collapsed.
- Generating manifests that pass the UI but fail `atk validate` / tenant upload.

## Constraints
- Windows 11, Node.js. Must call the installed `atk` CLI (Microsoft 365 Agents Toolkit) and reuse its sign-in to the M365 tenant.
- Source of truth = ATK project files on disk (JSON); each agent is a normal ATK project openable in VS Code.
- LLM: Claude API (key from KeyVault per Scott's standard), model `claude-opus-5-5` default, configurable.
- Preview/tenant-gated features must be flagged in the UI, not silently emitted.

## Approaches considered
- **A. Local web app, JSON manifest as source of truth — CHOSEN.** React UI + Node backend on localhost; Claude drafts JSON; JSON Schema drives validation and forms; `atk` handles lifecycle. Clean UI↔file round-trip.
- B. TypeSpec as source of truth — nicer API plugin authoring, but form edits can't round-trip reliably; UI degrades into a code editor.
- C. VS Code extension — less to build, but boxed into VS Code panels; no real management dashboard.

## Decided how
- Architecture: Vite + React + TypeScript frontend; Node (Express or Fastify) backend; one repo, `npm run dev` starts both.
- Data: agents live in a configurable workspace folder (default `C:\AIA-Code\CopilotAgents\agents\<agent-name>\`), each a standard ATK declarative-agent project. App metadata (skills library, settings) in a local JSON/SQLite store.
- Schemas: vendored official declarative agent / plugin / app manifest JSON schemas by version; Ajv validation server- and client-side.
- Generation: Claude produces a structured spec (tool-use / JSON output) → deterministic code writes files → schema validate → if invalid, one auto-repair round with errors fed back.
- UI screens: Library · New Agent (prompt) · Agent Editor (tabs: Overview, Instructions, Starters, Capabilities, Actions/API plugins, MCP, Skills, Raw JSON, Deploy) · Skills Library · Settings.
- Deploy: backend spawns `atk` commands and streams logs to the UI via SSE.

## Calls made by Claude (review these)
- **Feature gating:** hardcoded capability table per schema version (GA / preview / license-required), plus manual per-tenant toggles in Settings. No live tenant check.
- **External edits:** the app manages projects it created, round-trips losslessly (unknown fields preserved), and detects files changed outside the app, prompting before overwriting. Importing foreign ATK/TypeSpec projects is out of v1.
- **Skills:** agents keep *references* to library skills (with optional version pinning); a rebuild picks up changes. Merged content is tagged so it stays traceable; the build fails with a clear message if instructions exceed the 8,000-char limit.
- **Secrets:** stored in KeyVault and injected into the `atk` process environment at run time; never written to `.env` files.

## Decided by Scott
- **MCP:** runtime MCP actions (plugin with MCP runtime; app discovers tools from the server to populate the action and its auth). Flagged as preview; when the selected schema/tenant doesn't support it, offer fallback conversion to an OpenAPI plugin.
- **Deploy:** both. One-click default = sideload/preview for Scott; separate "Publish to org catalog" action (requires admin approval in the M365 admin center) with status shown.

## Left to the builder
- Exact component library (e.g. Fluent UI React v9 recommended for M365 look).
- Express vs Fastify; JSON vs SQLite for metadata.
- Form generation approach (hand-built per section vs schema-driven).
- Exact `atk` command mapping and how to detect install/sign-in state.
- Test strategy details (schema-validation golden tests strongly encouraged).
