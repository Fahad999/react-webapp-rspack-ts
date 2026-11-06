# Figma-to-React — Setup Guide

> A step-by-step guide to install the Figma MCP server, add the `@FigmaToReact` custom agent and its skills to a repo, and use it from GitHub Copilot Chat in VS Code.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Install & Run the Figma MCP Server](#2-install--run-the-figma-mcp-server)
3. [(Optional) Install & Run CodeGraph](#3-optional-install--run-codegraph)
4. [Add the Custom Agents & Skills](#4-add-the-custom-agents--skills)
5. [Using @FigmaToReact in Chat](#5-using-figmatoreact-in-chat)
6. [Troubleshooting](#6-troubleshooting)

---

## 1. Overview

This repo's Figma-to-code workflow has three required parts, plus one optional one:

| Part                  | What it is                                                             | Where it lives                      |
| --------------------- | ----------------------------------------------------------------------- | ------------------------------------ |
| **Figma MCP server**  | Gives Copilot Chat tools to read Figma files (`get_design_context`, `get_screenshot`, etc.) | Configured once per machine/workspace |
| **Custom agents**     | `@FigmaToReact`, `@FigmaSnapshot`, `@FigmaDev` — the orchestrator + workers | `.github/agents/*.agent.md`          |
| **Skills**            | Step-by-step mechanics each agent phase follows                        | `.github/skills/**/SKILL.md`         |
| **CodeGraph MCP server** _(optional)_ | Gives `@FigmaDev` a `codegraph_explore` tool for mapping existing files' call paths and verifying reused symbols' contracts — opt-in per task, see [Section 3](#3-optional-install--run-codegraph) | Configured once per machine; indexed once per project |

Without the MCP server connected, the agents can't fetch anything from Figma. Without the agent files, `@FigmaToReact` won't show up in the chat agent picker. CodeGraph is optional — everything works without it, falling back to the built-in search/read tools.

---

## 2. Install & Run the Figma MCP Server

You only need to do this once per machine. This uses Figma's **remote/hosted MCP server** — you do **not** need the Figma desktop app running.

1. Open the Command Palette:
   - **Mac:** `Cmd+Shift+P`
   - **Windows:** `Ctrl+Shift+P`
2. Search for and run **MCP: Open User Configuration**.
3. Inside the `mcp.json` file that opens, paste the following:
   ```json
   {
     "inputs": [],
     "servers": {
       "figma": {
         "url": "https://mcp.figma.com/mcp",
         "type": "http"
       }
     }
   }
   ```
4. Save the file:
   - **Mac:** `Cmd+S`
   - **Windows:** `Ctrl+S`
5. VS Code will show a **Start** button above the `figma` server entry. Click it to start the server and begin the authentication flow (this opens a browser window to sign in to Figma and authorize VS Code).

> Alternatively, follow Figma's official installation guide for this exact flow: [Remote MCP Server installation — VS Code](https://developers.figma.com/docs/figma-mcp-server/remote-server-installation/#vs-code).

### If you already have Figma MCP configured under a different key

The agent files reference Figma tools as `figma/...` (e.g. `'figma/*'`,
`figma/get_design_context`). That prefix comes from whatever key *you* gave
the server in your own `mcp.json` — not from anything in the Figma MCP server
itself. If your existing config already has it keyed as `figma-mcp`,
`figma-design`, or anything else, those hardcoded references won't resolve.

You don't need to rename your server or hand-edit any `.agent.md` file for
this. Either:

- Add a second server entry in your own `mcp.json` keyed exactly `figma`,
  pointing at the same URL (both resolve to the same remote server, one auth
  session, no conflict), **or**
- Ask `@FigmaToolDiscovery` to fix it for you — it detects your real server
  key from the currently available tools and repoints the three agent files
  automatically. It only acts when a Figma tool is actually missing, so it's
  safe to invoke directly if you're troubleshooting: `@FigmaToolDiscovery`.

### Verifying it's working

- Open Copilot Chat, click the **Tools** icon, and check that Figma tools are enabled (not greyed out).
- Ask `@FigmaToReact` to fetch a Figma URL you have access to — if the server isn't connected, you'll get a tool error instead of design data.

---

## 3. (Optional) Install & Run CodeGraph

CodeGraph gives `@FigmaDev` a `codegraph_explore` tool that maps an existing file's call paths (hooks, selectors, handlers) and verifies a reused symbol's exact contract in one shot, instead of a grep-then-read loop. **This is entirely optional** — `@FigmaToReact` asks you per task (its Phase 2.5 gate) whether to enable it, explaining which specific scenarios it would help with for that task, and everything falls back automatically to the built-in search/read tools if you say no or if CodeGraph isn't installed/indexed.

### 3.1 — Install the CLI

Install the `codegraph` CLI binary for your platform (check with your team for the current distribution channel if you don't already have one). Once installed, you should have a `codegraph` executable on your `PATH` (e.g. `~/.local/bin/codegraph`).

Verify it's installed:

```bash
codegraph --version
```

### 3.2 — Wire it into GitHub Copilot (VS Code)

Run:

```bash
codegraph install --target copilot-vscode
```

This writes (or merges) the following into your user `mcp.json` — equivalent to opening **MCP: Open User Configuration** and pasting it yourself, the same way you did for the Figma server in [Section 2](#2-install--run-the-figma-mcp-server):

```json
{
  "servers": {
    "codegraph": {
      "type": "stdio",
      "command": "codegraph",
      "args": ["serve", "--mcp"]
    }
  }
}
```

Reload the VS Code window afterward so the new MCP server is picked up.

### 3.3 — Index a project

CodeGraph only works on projects that have been indexed. From the repo root:

```bash
codegraph init
```

This creates a local `.codegraph/` folder (already gitignored — see `.codegraph/.gitignore`) containing the index database. A freshly built or updated index is picked up live by the MCP server — no restart needed. Check status any time with:

```bash
codegraph status
```

### 3.4 — Using it with @FigmaToReact

You don't need to do anything else in chat. Once the CLI is installed, wired into Copilot, and the project is indexed, `@FigmaToReact` will offer to use it during its **Phase 2.5 — Codegraph Usage Decision** on any Figma-to-code task, and will only rely on it after you explicitly reply yes.

### 3.5 — Useful commands

| Command                   | What it does                                                |
| -------------------------- | ------------------------------------------------------------- |
| `codegraph status`        | Show index stats and whether it's up to date                |
| `codegraph sync`          | Update the index after code changes (faster than a full re-index) |
| `codegraph explore <query>` | Manually run the same exploration the `codegraph_explore` MCP tool performs |
| `codegraph uninit`        | Remove CodeGraph from this project (deletes `.codegraph/`)  |
| `codegraph upgrade`       | Update the CLI to the latest version                         |

---

## 4. Add the Custom Agents & Skills

This repo already has everything wired up under `.github/`. To bring the same workflow into **another** repo, copy these files over, preserving the folder structure:

```
.github/
├── agents/
│   ├── figma-to-react.agent.md     ← the manager agent (@FigmaToReact)
│   ├── figma-snapshot.agent.md     ← capture worker (@FigmaSnapshot)
│   ├── figma-dev.agent.md          ← implementation worker (@FigmaDev)
│   └── figma-tool-discovery.agent.md ← reactive repair worker (@FigmaToolDiscovery)
├── skills/
│   ├── figma-snapshot/SKILL.md     ← Phase 1 capture mechanics
│   ├── figma-to-code/SKILL.md      ← Phase 3–6 code generation mechanics
│   └── figma-to-react-impl/SKILL.md
├── figma-project-profile.md        ← repo-specific paths/conventions
├── figma-token-map.md              ← Figma → design-token/component mapping
├── review-prompt.md                ← review rules used in Phase 6 validation
└── copilot-instructions.md         ← points Copilot at all of the above
```

Also copy/adapt:

- `GUIDELINES.md` (root) — the underlying React/TypeScript coding standards these agents enforce.
- The `copilot-instructions.md` block that lists the agents/skills (see this repo's file for the exact wording) — append it to the target repo's own `copilot-instructions.md` rather than overwriting it if one already exists.

Then:

1. Reload the VS Code window (or run **Developer: Reload Window**) so it picks up the new `.agent.md` files.
2. Open Copilot Chat and check the agent/mode picker — `FigmaToReact`, `FigmaSnapshot`, and `FigmaDev` should now be listed alongside the built-in `Ask`/`Edit`/`Agent` modes.

> Only `figma-to-react.agent.md` needs to be selected manually. `FigmaSnapshot` and `FigmaDev` are internal workers that `FigmaToReact` delegates to automatically — you shouldn't need to select them yourself.

---

## 5. Using @FigmaToReact in Chat

1. Open **Copilot Chat** in VS Code.
2. Click the **agent/mode dropdown** at the top of the chat input box (it normally shows `Agent`, `Edit`, `Ask`, or the last agent you used).
3. Select **FigmaToReact** from the list.
4. Paste a Figma design URL and describe what you want, e.g.:
   ```
   Implement this design from Figma.
   @https://www.figma.com/design/0IKAdU6vKnRh0xZLhul9GJ/CC-Acquisition--SOT-for-Current-Journey?node-id=402-30358&m=dev
   ```
   Where possible, also share any relevant details about the design (e.g. is it a new page or an update to an existing one, mobile/desktop/both, any feature spec) — `FigmaToReact` will still ask clarifying questions in Phase 2, but the more context you give upfront, the fewer follow-up questions it needs to ask.
5. Follow the prompts — `FigmaToReact` will:
   - Capture/reuse a snapshot of the frame (`.figma-snapshots/<slug>/` — local-only; make sure `.figma-snapshots/` is listed in your `.gitignore` so you never commit it)
   - Show you a screenshot and ask you to confirm it's the right design
   - Ask clarifying questions (new vs. update, page vs. component, feature spec, interaction/state coverage, routing)
   - Generate the code and run Prettier/ESLint/TypeScript validation before handing it back to you

You can also type `@FigmaToReact` directly in the chat box instead of using the dropdown — either way routes you to the same agent.

---

## 6. Troubleshooting

| Symptom                                             | Likely cause                                                                 |
| ---------------------------------------------------- | ----------------------------------------------------------------------------- |
| `FigmaToReact` doesn't appear in the agent picker    | `.agent.md` files aren't under `.github/agents/`, or the window needs a reload |
| Figma tool calls fail immediately                    | The `figma` server in your user `mcp.json` hasn't been started, or the auth flow wasn't completed |
| Tools are visible but calls hang or time out         | You don't have access to the Figma file/node, or the browser auth session expired — re-run **MCP: Open User Configuration** and click **Start** again |
| Agent generates code without asking for confirmation | You're on an older/edited copy of `figma-to-react.agent.md` — Phase 1.5 (the visual confirmation gate) must not be skipped |
| Figma tools silently unavailable even though the server shows "Running" | Your server key in `mcp.json` isn't `figma` — ask `@FigmaToolDiscovery` to detect your real key and repoint the agent files, or add a second `figma`-keyed entry yourself (see [Section 2](#2-install--run-the-figma-mcp-server)) |
| `@FigmaToReact` never offers to use CodeGraph in Phase 2.5, or `@FigmaDev` always falls back to grep/read | The `codegraph` CLI isn't installed, isn't wired into your user `mcp.json` (run `codegraph install --target copilot-vscode`), or the project hasn't been indexed yet (run `codegraph init` — see [Section 3](#3-optional-install--run-codegraph)). This is expected/harmless — the workflow still works without it |
| Agents disappear from the picker / `agent not found` errors mid-session | Editing any `.github/agents/*.agent.md` file deregisters **all** custom agents for the current session, not just the edited one. Reload the VS Code window, then you can continue in the same chat session (e.g. reply "continue implementing") — no new session needed |
