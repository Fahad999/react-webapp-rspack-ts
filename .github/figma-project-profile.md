# Figma Project Profile

This file defines the repository-specific locations and conventions used by the
Figma agents. It is authoritative for this workspace.

For Figma work, follow the scoped Instruction Precedence policy in
`@FigmaToReact`. This profile controls repository locations and local patterns
after manager lifecycle gates have passed; it takes precedence over generic
agent or skill path assumptions.

## Standards and Validation

| Concern                       | Location or command                    |
| :---------------------------- | :------------------------------------- |
| Repository coding standards   | `GUIDELINES.md`                        |
| Code and review standards     | `.github/review-prompt.md`             |
| Figma token and component map | `.github/figma-token-map.md`           |
| Format                        | `npx prettier --write <changed-files>` |
| Lint                          | `npm run eslint`                       |
| Type check                    | `npm run tsc:lint`                     |
| Tests                         | `npm run test:coverage`                |

## Application Structure

| Concern                            | Location or convention                                                                      |
| :--------------------------------- | :------------------------------------------------------------------------------------------ |
| Page components                    | `src/Pages/<Domain>/`                                                                       |
| Page-specific components and hooks | `src/Pages/<Domain>/<Feature>/Components/` and `Hooks/`                                     |
| Reusable components                | `src/Components/`                                                                           |
| Route constants                    | `src/Constants/APP_ROUTES.ts`                                                               |
| Router configuration               | `src/Configurations/getAppRouter.tsx`                                                       |
| Shared utilities and hooks         | `src/Lib/`                                                                                  |
| Typed Redux dispatch hook          | `~/src/Lib/hooks.ts` (`useAppDispatch`)                                                     |
| Redux domains                      | `src/Redux/<Domain>/`                                                                       |
| Services                           | Inspect the target Redux domain first; legacy domains use either `services/` or `Services/` |
| Selectors                          | `src/Redux/<Domain>/Selectors.ts` when that domain follows the selector pattern             |
| Image assets                       | `src/AssetFiles/images/`                                                                    |
| Asset constants                    | `src/Constants/ASSET_MAP.ts`                                                                |

## Discovery Fallback

When this profile does not define a needed location or convention:

1. Inspect one existing equivalent in the target domain or feature.
2. Follow that local pattern.
3. Ask the user only when no reliable pattern exists.

Do not create a new top-level folder or import path based on a generic Figma
agent instruction when it conflicts with this profile or an established local
pattern.
