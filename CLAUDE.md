# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Authoritative guide: read `AGENTS.md` before any work.** Scoped `AGENTS.md` files exist in `extensions/`, `src/{plugin-sdk,channels,plugins,gateway,gateway/protocol,agents}/`, `test/helpers*/`, `docs/`, `ui/`, and `scripts/` — read the relevant one before subtree work.

## Commands

```sh
pnpm install                                  # install deps (pnpm workspace, Node 22+)
pnpm dev / pnpm openclaw ...                  # run CLI locally
pnpm build                                    # production build

# Testing
pnpm test <path-or-filter> [vitest args...]   # targeted test run (never raw vitest)
pnpm test:changed                             # tests for changed lanes only
pnpm test:serial                              # serial run (or OPENCLAW_VITEST_MAX_WORKERS=1)
pnpm test extensions/<id>                     # single extension tests
pnpm test:contracts:channels                  # channel contract tests
pnpm test:contracts:plugins                   # plugin contract tests
OPENCLAW_LIVE_TEST=1 pnpm test:live           # live integration tests

# Type checking — use tsgo lanes only, never tsc --noEmit
pnpm tsgo*                                    # typecheck lanes
pnpm check:test-types

# Format / Lint
pnpm format / pnpm format:check               # oxfmt (not Prettier)
pnpm exec oxfmt --write --threads=1 <files>   # targeted format
pnpm lint:*                                   # repo oxlint wrappers only

# Validation gates
pnpm check:changed                            # smart changed-lane gate (sparse-safe)
pnpm check                                    # full prod sweep — run in Testbox, not locally
pnpm check:architecture                       # import cycles + madge
pnpm check:import-cycles

# Commit
scripts/committer "<msg>" <file...>           # formats staged files; run gates separately
```

**Broad gates** (`pnpm check`, full `pnpm test`, Docker/E2E/live/package/build) belong in **Testbox** by default, not local. Run targeted local commands (`pnpm test <file>`, narrow `pnpm test:changed`) locally.

Do not run multiple concurrent `pnpm test`/Vitest processes in the same worktree — they race on the Vitest cache. Use one grouped invocation or set distinct `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH` values.

## Architecture

**OpenClaw** is a TypeScript ESM monorepo (strict mode) — a multi-channel personal AI assistant gateway with 150+ bundled plugins.

```
src/           # core: agents, gateway, channels, cli, config, cron, plugin-sdk, acp, …
extensions/    # bundled plugins: providers (anthropic, openai), channels (telegram, discord, …), capabilities
packages/      # published packages: plugin-sdk, memory-host-sdk, sdk, plugin-package-contract
ui/            # web dashboard
apps/          # native apps: ios, android, macos, windows
docs/          # mdx docs + generated API references
```

### Key boundaries

- **Core stays extension-agnostic.** Extensions cross into core only via `openclaw/plugin-sdk/*`, manifest metadata, and documented barrels (`api.ts`, `runtime-api.ts`). No `src/**` or `src/plugin-sdk-internal/**` imports from extensions; no deep plugin internals from core/tests.
- **Extension-owned behavior stays in extensions** — repair, detection, onboarding, auth/provider defaults, provider tools. Fix owner-specific bugs in the owner module; add a generic core seam only when multiple owners need it.
- **Dependency ownership follows runtime ownership**: extension-only deps stay plugin-local; root deps for core or intentionally internalized bundled plugin runtime.
- **Request-time runtime resolution**: carry prepared facts (`AgentRuntimePlan`, `ProviderRuntimePluginHandle`, scoped model helpers) through context from startup/dispatch. Hot reply/tool/outbound paths must not call broad plugin/provider/channel loaders (`loadOpenClawPlugins`, `resolveProviderPluginsForHooks`, etc.) to answer questions the caller already knows.
- **No legacy compat in runtime paths.** Use `openclaw doctor --fix` for config repair. Retired config keys stay retired.
- **Prompt cache**: deterministic ordering for registries/plugin lists/maps/sets/files/network results before model/tool payloads.
- **Dynamic imports**: no static + dynamic import of the same prod module. Use `*.runtime.ts` lazy boundaries. After edits run `pnpm build` and check `[INEFFECTIVE_DYNAMIC_IMPORT]`.
- **File size**: split around ~700 LOC when clarity/testability improves.
- **Cycles**: keep `pnpm check:import-cycles` + architecture/madge green.

### Testing conventions

- Vitest only; colocated `*.test.ts`; E2E as `*.e2e.test.ts`.
- Example model names: `sonnet-4.6`, `gpt-5.5` (no GPT-4.x agent-smoke defaults).
- Clean timers/env/globals/mocks/sockets/temp dirs per test; `--isolate=false` safe.
- Mock expensive seams directly (scanners, manifests, registries, fs crawls, provider SDKs, network).
- Plugin registry tests need both manifest-registry and metadata-snapshot exports.
- Thread-bound subagent tests that don't create a requester transcript: set `context: "isolated"`.
- Prefer injection over module mocking; if mocking, mock narrow local `*.runtime.ts`, not broad barrels.
- Do not edit baseline/snapshot/expected-failure files without explicit approval.
- Test guide: `docs/help/testing.md`.

### Code conventions

- **Imports**: `.js` extension on all ESM cross-package imports; `import type { X }` for type-only. No re-export wrapper files — import directly from the source.
- **Build**: `tsdown` outputs to `dist/`. CLI uses Commander + clack/prompts. After touching lazy/module boundaries run `pnpm build` and check `[INEFFECTIVE_DYNAMIC_IMPORT]`.
- **Types**: Avoid `any`; prefer `unknown` or narrow adapters. `zod` at external boundaries. Discriminated unions over freeform strings for runtime branching.
- **No `@ts-nocheck`**; lint suppressions must be intentional and explained.

### Key utilities — do not duplicate

Search before creating helpers. Authoritative locations:

| Need | Module |
|---|---|
| Time/duration formatting | `src/infra/format-time` |
| Terminal tables | `src/terminal/table.ts` (`renderTable`) |
| Terminal colors/themes | `src/terminal/theme.ts` (`theme.success`, `theme.muted`, …) |
| CLI spinners / progress bars | `src/cli/progress.ts` |
| CLI option wiring | `src/cli/` |
| CLI commands | `src/commands/` |

### Mac gateway dev

- Dev watch: `pnpm gateway:watch` (tmux session `openclaw-gateway-watch-main`; auto-attaches).
- Non-interactive: `OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch`.
- Logs: `./scripts/clawlog.sh`.

### Naming & language

- Product/docs/UI/changelog: **OpenClaw**, "plugin/plugins".
- CLI/package/path/config: `openclaw`. Internal directory: `extensions/`.
- American English spelling.
