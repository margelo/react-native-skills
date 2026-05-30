---
title: Repository Structure and Workflow
impact: HIGH
tags: repo-structure, monorepo, apps, packages, docs, config, ci, workflow
---

# Skill: Repository Structure and Workflow

Use this when creating a new repo, reorganizing a repo, adding examples/docs/CI, or making a large feature that changes package layout.

## Recommended Layout

```
<root>/
├── README.md
├── package.json
├── bun.lock
├── packages/
│   └── react-native-math/
├── apps/
│   └── example/
├── docs/
├── scripts/
├── config/
└── .github/
    └── workflows/
```

## Rules

- Keep the root small so `README.md` stays visible in GitHub's file list. Root files should mostly be `README.md`, `package.json`, lockfiles, `packages/`, one example-app location, optional `docs/`, optional `scripts/`, `config/`, `.github/`, and root-only config files required by tools.
- Put publishable libraries in `packages/<package-name>/`.
- Use `apps/<example-name>/` when the repo may need multiple example apps, including optional native dependencies, feature variants, or integration demos.
- Use top-level `example/` only for intentionally small single-example repos that should preserve React Native's generated paths.
- Keep example apps close to the official React Native template. Use official APIs, generated configs, and template-supported extension points before custom setup.
- Add `docs/` only when `README.md` is not enough. Use Fumadocs for a full docs site.
- Add `scripts/` only for reusable repo automation.
- Put shared config under `config/` when tools can reference it, including TypeScript configs, lint/format configs, `.swift-format`, `.clang-format`, and `.editorconfig`. If a tool requires a root config file, keep the root file minimal and delegate to `config/`.
- Prefer Bun commands: `bun install`, `bun run`, `bun --cwd`, and `bunx`. Use `npx` only when Bun cannot run the tool.
- Add `.github/workflows/` for CI validation. Start with TypeScript/build checks, then add JS/native lint jobs when the codebase is mature enough to justify them.
- Do not add Husky, commitlint, lint-staged, pre-commit hooks, pre-push hooks, or `prepare` scripts that install hooks. Validation belongs in CI.
- Avoid `patch-package`, postinstall rewrites, monkeypatching, and workaround layers. Fix package layout, autolinking, generated config, official extension points, or upstream issues first.
- Work on a separate branch and open a draft PR early so CI can run while local work continues.
- Use squash merges for `main`.
- After initial release or after major rewrites settle, keep PRs atomic unless changes are tightly coupled.

## Root Workspace

For `apps/example`:

```json
{
  "name": "react-native-math-root",
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ],
  "scripts": {
    "specs": "bun --cwd packages/react-native-math run specs",
    "example": "bun --cwd apps/example"
  }
}
```

For top-level `example/`, use:

```json
{
  "workspaces": [
    "packages/*",
    "example"
  ],
  "scripts": {
    "example": "bun --cwd example"
  }
}
```

## CI Baseline

Start with:

- TypeScript/build: `bun install --frozen-lockfile`, `bun run typecheck`, `bun run build`, `bun run specs`
- JS lint/format: Biome, or ESLint/Prettier if already used
- Native lint/format once mature: SwiftLint/SwiftFormat, clang-format/clang-tidy, ktlint, or Detekt

## Common Pitfalls

- **Root pollution** — Move config into `config/` when tools support it.
- **Two example locations** — Use either `apps/example` or `example/`, not both.
- **Hook frameworks** — Do not add commit/push hooks; use CI.
- **Patch layers** — Avoid patches and postinstall rewrites; fix the root cause.
- **Late PRs** — Open a draft PR early so CI runs while implementation continues.
