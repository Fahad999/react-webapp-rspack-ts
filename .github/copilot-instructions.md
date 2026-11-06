Refer to the following files for all coding standards, review rules, and project conventions:

- `GUIDELINES.md` — Repository-wide React, TypeScript, state, styling, API, and asset standards.
- `.github/figma-project-profile.md` — Repository-specific paths, conventions, and validation commands for Figma-generated work.
- `.github/review-prompt.md` — Review rules, severity definitions, and output format for code audits.
- `.github/agents/figma-to-react.agent.md` — Figma-to-React agent: orchestrates snapshot capture, feature spec, clarifying questions, and code generation via sub-agents.
- `.github/figma-token-map.md` — Figma-to-token and Figma-to-component mapping for AI code generation from Figma MCP.
- `.github/skills/figma-to-code/SKILL.md` — Step-by-step process for generating or updating React UI from a Figma design link.
- `.github/skills/figma-snapshot/SKILL.md` — Process for capturing and storing Figma design data locally as snapshots.

## Figma Sub-Agents

- `@FigmaToReact` — **Recommended entry point.** Full Figma-to-React workflow: orchestrates snapshot capture, asks clarifying questions (including feature spec), and delegates code generation. Use for all new Figma-to-code requests.
- `@FigmaSnapshot` — Internal capture worker for `@FigmaToReact`; it stores approved Figma data in `.figma-snapshots/`.
- `@FigmaDev` — Internal implementation worker for `@FigmaToReact`; it only generates from a manager-approved handoff and snapshot.
- `@FigmaToolDiscovery` — Reactive-only repair worker; fixes the `figma/...` tool references in the agents above if your local MCP server is keyed differently. Only runs when a Figma tool is genuinely missing — never proactively.
