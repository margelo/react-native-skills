---
title: Monorepo Setup and Nitrogen Scaffold
impact: CRITICAL
tags: monorepo, nitrogen, init, workspace, bun, scaffold, package-json
---

# Skill: Monorepo Setup and Nitrogen Scaffold

Covers new-repo structure, getting the library name, scaffolding with Nitrogen, CI placement, and branch/PR workflow.

## Quick Commands

```bash
# After collecting answers to all upfront questions, scaffold:
bunx nitrogen@latest init react-native-math
# This places the library in packages/react-native-math/

# After scaffold, install from root:
bun install
```

## When to Use

- **Only for new libraries** — if the user is adding a HybridObject to an existing library, skip this file entirely and go to [spec-hybrid-object.md](spec-hybrid-object.md)
- Starting a new Nitro module library from scratch
- Setting up the monorepo workspace before writing specs

## Prerequisites

- Node.js and Bun installed
- Answers collected from the user (see Ask First section in SKILL.md)

## Step-by-Step

### 1. Collect answers before doing anything

Ask the user all of the following before running any command:

| Question | Default |
|----------|---------|
| What is the library name? (e.g. `react-native-math`) | — required |
| Use monorepo with `packages/<name>` folder? | **yes** |
| Create an example app to test the module? | **yes, usually `apps/example`** |
| iOS language: `swift` or `cpp`? | **swift** |
| Android language: `kotlin` or `cpp`? | **kotlin** |
| What does this module do? (brief description) | — required |

Only proceed once all questions are answered.

### 2. Set up the monorepo structure

This skill defaults to placing publishable libraries in `packages/<name>` inside a clean workspace root. Nitro does not require this layout, but it keeps package code, apps, docs, scripts, config, and CI separated:

```
<root>/
├── README.md                    ← keep visible in GitHub's root file list
├── package.json                 ← root workspace config and CI/manual scripts
├── bun.lock
├── packages/
│   └── react-native-math/       ← publishable library package
├── apps/
│   └── example/                 ← preferred when more examples may be needed
├── example/                     ← alternative only; do not use with apps/example
├── docs/                        ← optional; prefer Fumadocs when a docs site is needed
├── scripts/                     ← optional reusable repo automation
├── config/                      ← shared tool config when tools can reference it
└── .github/
    └── workflows/               ← CI validation
```

Choose one example app location, not both. Prefer `apps/example` when multiple examples are already needed or likely, such as optional native dependencies, feature variants, or separate integration demos. It is fine to start with only `apps/example` even when future examples are speculative because moving from `example/` to `apps/` later is needless churn. Use a shallower `example/` app only when the repo is intentionally small and keeping React Native's generated config closer to default is more valuable.

Keep root pollution low so users do not have to scroll past config files to find `README.md`. Put shared configs such as `tsconfig.json`, `.swift-format`, `.clang-format`, `.editorconfig`, and lint/format configs under `config/` when the tool can reliably reference them. If a tool or editor requires discovery from the root, keep the root file minimal and delegate to `config/`.

Stay close to official templates and APIs. Avoid `patch-package`, postinstall rewrites, monkeypatching, and hacky workaround layers unless there is no reasonable official or upstreamable path. Workarounds tend to spread into more setup code, raise the maintenance burden, and make the repo harder for new contributors to understand. When something requires ugly manual plumbing, fix the underlying package layout, autolinking, generated config, or upstream issue first.

If a root `package.json` does not exist yet, create one:

```json
{
  "name": "react-native-math-root",
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

For the shallower layout, use `"example"` instead of `"apps/*"` in `workspaces`.

### 3. Confirm the library name

The library name should:
- Follow npm naming: `react-native-<domain>` (e.g. `react-native-math`, `react-native-camera`)
- Be lowercase, hyphen-separated
- Reflect the module's purpose

### 4. Scaffold with Nitrogen

Run from the monorepo root:

```bash
bunx nitrogen@latest init react-native-math
```

This creates `packages/react-native-math/` when using the monorepo layout. If the user explicitly chose a non-monorepo layout, run the same command from the target library parent and adjust later paths accordingly.

### 5. Verify the generated folder structure

```
packages/react-native-math/
├── NitroMath.podspec
├── android/
│   ├── src/main/java/com/margelo/nitro/<namespace>/
│   └── CMakeLists.txt
├── ios/
│   └── HybridMath.swift        ← later implementation
├── src/
│   └── specs/
│       └── Example.nitro.ts    ← delete this
├── nitrogen/
│   └── generated/              ← populated after running nitrogen
├── nitro.json
└── package.json
```

### 6. Add to root workspace

In the monorepo root `package.json`:

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

If the example app lives at `example/`, use `"workspaces": ["packages/*", "example"]` and `"example": "bun --cwd example"` instead.

### 7. Install from root

```bash
bun install
```

### 8. Add CI without local commit hooks

Use `.github/workflows/` for validation instead of commit-time or push-time enforcement. Start with TypeScript/build checks and add lint jobs as the codebase matures:

- TypeScript/build: `bun install --frozen-lockfile`, `bun run typecheck`, `bun run build`, `bun run specs`
- JS lint/format: Biome, or ESLint/Prettier if the repo already uses them
- Native lint/format when mature enough to justify it: SwiftLint/SwiftFormat, clang-format/clang-tidy, ktlint, or Detekt

Do not add Husky, commitlint, lint-staged, pre-commit hooks, pre-push hooks, or `prepare` scripts that install hooks. Local scripts may exist for manual use, but CI is the source of truth.

### 9. Use branch and PR workflow

Work on a separate branch and open a draft PR early so CI can run while local development continues. Use squash merges for a clean `main` history. After the first release or once major rewrites settle, keep PRs atomic and decoupled so review and later reverts stay straightforward.

## Code Examples

### Root `package.json`

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
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

### Package `package.json` (generated, key fields)

```json
{
  "name": "react-native-math",
  "version": "0.1.0",
  "description": "A React Native Math module built with Nitro",
  "main": "lib/index",
  "module": "lib/index",
  "react-native": "src/index",
  "source": "src/index",
  "types": "lib/index.d.ts",
  "peerDependencies": {
    "react": "*",
    "react-native": "*",
    "react-native-nitro-modules": "*"
  }
}
```

## Common Pitfalls

- **Running `nitrogen init` from wrong directory** — Run from the monorepo root, not inside `packages/`
- **Forgetting to add package to workspaces** — Without this, `bun install` won't link the package
- **Adding hook frameworks** — Do not add Husky, commitlint, lint-staged, pre-commit, or pre-push hooks; use CI instead
- **Polluting the root** — Keep configs in `config/` when supported, and keep the root focused on README, workspace files, package folders, apps, docs, scripts, and CI
- **Normalizing workaround layers** — Avoid patches, postinstall rewrites, monkeypatching, and manual plumbing; fix the root cause or use official extension points
- **Name collisions** — Check npm before choosing a name (`npm info react-native-<name>`)
- **Not running `bun install` after scaffold** — Dependencies won't be linked until you do

## Related Skills

- [spec-hybrid-object.md](spec-hybrid-object.md) — Next step: write the TypeScript spec
- [spec-nitro-json.md](spec-nitro-json.md) — Configure autolinking before running nitrogen
