# Repository Guidelines

- Repo: https://github.com/openclaw/openclaw
- In chat replies, file references must be repo-root relative only (example: `extensions/bluebubbles/src/channel.ts:80`); never absolute paths or `~/...`.
- Do not edit files covered by security-focused `CODEOWNERS` rules unless a listed owner explicitly asked for the change or is already reviewing it with you. Treat those paths as restricted surfaces, not drive-by cleanup.

## Project Structure & Module Organization

- Source code: `src/` (CLI wiring in `src/cli`, commands in `src/commands`, web provider in `src/provider-web.ts`, infra in `src/infra`, media pipeline in `src/media`).
- Tests: colocated `*.test.ts`.
- Docs: `docs/` (images, queue, Pi config). Built output lives in `dist/`.
- Plugins/extensions: live under `extensions/*` (workspace packages). Keep plugin-only deps in the extension `package.json`; do not add them to the root `package.json` unless core uses them.
- Plugins: install runs `npm install --omit=dev` in plugin dir; runtime deps must live in `dependencies`. Avoid `workspace:*` in `dependencies` (npm install breaks); put `openclaw` in `devDependencies` or `peerDependencies` instead (runtime resolves `openclaw/plugin-sdk` via jiti alias).
- Import boundaries: extension production code should treat `openclaw/plugin-sdk/*` plus local `api.ts` / `runtime-api.ts` barrels as the public surface. Do not import core `src/**`, `src/plugin-sdk-internal/**`, or another extension's `src/**` directly.
- Installers served from `https://openclaw.ai/*`: live in the sibling repo `../openclaw.ai` (`public/install.sh`, `public/install-cli.sh`, `public/install.ps1`).
- Messaging channels: always consider **all** built-in + extension channels when refactoring shared logic (routing, allowlists, pairing, command gating, onboarding, docs).
  - Core channel docs: `docs/channels/`
  - Core channel code: `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web` (WhatsApp web), `src/channels`, `src/routing`
  - Extensions (channel plugins): `extensions/*` (e.g. `extensions/msteams`, `extensions/matrix`, `extensions/zalo`, `extensions/zalouser`, `extensions/voice-call`)
- When adding channels/extensions/apps/docs, update `.github/labeler.yml` and create matching GitHub labels (use existing channel/extension label colors).

## Docs Linking (Mintlify)

- Docs are hosted on Mintlify (docs.openclaw.ai).
- Internal doc links in `docs/**/*.md`: root-relative, no `.md`/`.mdx` (example: `[Config](/configuration)`).
- When working with documentation, read the mintlify skill.
- For docs, UI copy, and picker lists, order services/providers alphabetically unless the section is explicitly describing runtime behavior (for example auto-detection or execution order).
- Section cross-references: use anchors on root-relative paths (example: `[Hooks](/configuration#hooks)`).
- Doc headings and anchors: avoid em dashes and apostrophes in headings because they break Mintlify anchor links.
- When Peter asks for links, reply with full `https://docs.openclaw.ai/...` URLs (not root-relative).
- When you touch docs, end the reply with the `https://docs.openclaw.ai/...` URLs you referenced.
- README (GitHub): keep absolute docs URLs (`https://docs.openclaw.ai/...`) so links work on GitHub.
- Docs content must be generic: no personal device names/hostnames/paths; use placeholders like `user@gateway-host` and “gateway host”.

## Docs i18n (zh-CN)

- `docs/zh-CN/**` is generated; do not edit unless the user explicitly asks.
- Pipeline: update English docs → adjust glossary (`docs/.i18n/glossary.zh-CN.json`) → run `scripts/docs-i18n` → apply targeted fixes only if instructed.
- Before rerunning `scripts/docs-i18n`, add glossary entries for any new technical terms, page titles, or short nav labels that must stay in English or use a fixed translation (for example `Doctor` or `Polls`).
- `pnpm docs:check-i18n-glossary` enforces glossary coverage for changed English doc titles and short internal doc labels before translation reruns.
- Translation memory: `docs/.i18n/zh-CN.tm.jsonl` (generated).
- See `docs/.i18n/README.md`.
- The pipeline can be slow/inefficient; if it’s dragging, ping @jospalmbier on Discord instead of hacking around it.

## exe.dev VM ops (general)

- Access: stable path is `ssh exe.dev` then `ssh vm-name` (assume SSH key already set).
- SSH flaky: use exe.dev web terminal or Shelley (web agent); keep a tmux session for long ops.
- Update: `sudo npm i -g openclaw@latest` (global install needs root on `/usr/lib/node_modules`).
- Config: use `openclaw config set ...`; ensure `gateway.mode=local` is set.
- Discord: store raw token only (no `DISCORD_BOT_TOKEN=` prefix).
- Restart: stop old gateway and run:
  `pkill -9 -f openclaw-gateway || true; nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &`
- Verify: `openclaw channels status --probe`, `ss -ltnp | rg 18789`, `tail -n 120 /tmp/openclaw-gateway.log`.

## Build, Test, and Development Commands

- Runtime baseline: Node **22+** (keep Node + Bun paths working).
- Install deps: `pnpm install`
- If deps are missing (for example `node_modules` missing, `vitest not found`, or `command not found`), run the repo’s package-manager install command (prefer lockfile/README-defined PM), then rerun the exact requested command once. Apply this to test/build/lint/typecheck/dev commands; if retry still fails, report the command and first actionable error.
- Pre-commit hooks: `prek install` (runs same checks as CI)
- Also supported: `bun install` (keep `pnpm-lock.yaml` + Bun patching in sync when touching deps/patches).
- Prefer Bun for TypeScript execution (scripts, dev, tests): `bun <file.ts>` / `bunx <tool>`.
- Run CLI in dev: `pnpm openclaw ...` (bun) or `pnpm dev`.
- Node remains supported for running built output (`dist/*`) and production installs.
- Mac packaging (dev): `scripts/package-mac-app.sh` defaults to current arch.
- Type-check/build: `pnpm build`
- TypeScript checks: `pnpm tsgo`
- Lint/format: `pnpm check`
- Format check: `pnpm format` (oxfmt --check)
- Format fix: `pnpm format:fix` (oxfmt --write)
- Tests: `pnpm test` (vitest); coverage: `pnpm test:coverage`
- For narrowly scoped changes, prefer narrowly scoped tests that directly validate the touched behavior. If no meaningful scoped test exists, say so explicitly and use the next most direct validation available.
- Preferred landing bar for pushes to `main`: `pnpm check` and `pnpm test`, with a green result when feasible.
- Scoped tests prove the change itself. `pnpm test` remains the default `main` landing bar; scoped tests do not replace full-suite gates by default.
- Hard gate: if the change can affect build output, packaging, lazy-loading/module boundaries, or published surfaces, `pnpm build` MUST be run and MUST pass before pushing `main`.
- Default rule: do not commit or push with failing format, lint, type, build, or required test checks when those failures are caused by the change or plausibly related to the touched surface.
- For narrowly scoped changes, if unrelated failures already exist on latest `origin/main`, state that clearly, report the scoped tests you ran, and ask before broadening scope into unrelated fixes or landing despite those failures.
- Do not use scoped tests as permission to ignore plausibly related failures.

## Coding Style & Naming Conventions

- Language: TypeScript (ESM). Prefer strict typing; avoid `any`.
- Formatting/linting via Oxlint and Oxfmt.
- Never add `@ts-nocheck` and do not disable `no-explicit-any`; fix root causes and update Oxlint/Oxfmt config only when required.
- Dynamic import guardrail: do not mix `await import("x")` and static `import ... from "x"` for the same module in production code paths. If you need lazy loading, create a dedicated `*.runtime.ts` boundary (that re-exports from `x`) and dynamically import that boundary from lazy callers only.
- Dynamic import verification: after refactors that touch lazy-loading/module boundaries, run `pnpm build` and check for `[INEFFECTIVE_DYNAMIC_IMPORT]` warnings before submitting.
- Extension SDK self-import guardrail: inside an extension package, do not import that same extension via `openclaw/plugin-sdk/<extension>` from production files. Route internal imports through a local barrel such as `./api.ts` or `./runtime-api.ts`, and keep the `plugin-sdk/<extension>` path as the external contract only.
- Extension package boundary guardrail: inside `extensions/<id>/**`, do not use relative imports/exports that resolve outside that same `extensions/<id>` package root. If shared code belongs in the plugin SDK, import `openclaw/plugin-sdk/<subpath>` instead of reaching into `src/plugin-sdk/**` or other repo paths via `../`.
- Extension API surface rule: `openclaw/plugin-sdk/<subpath>` is the only public cross-package contract for extension-facing SDK code. If an extension needs a new seam, add a public subpath first; do not reach into `src/plugin-sdk/**` by relative path.
- Never share class behavior via prototype mutation (`applyPrototypeMixins`, `Object.defineProperty` on `.prototype`, or exporting `Class.prototype` for merges). Use explicit inheritance/composition (`A extends B extends C`) or helper composition so TypeScript can typecheck.
- If this pattern is needed, stop and get explicit approval before shipping; default behavior is to split/refactor into an explicit class hierarchy and keep members strongly typed.
- In tests, prefer per-instance stubs over prototype mutation (`SomeClass.prototype.method = ...`) unless a test explicitly documents why prototype-level patching is required.
- Add brief code comments for tricky or non-obvious logic.
- Keep files concise; extract helpers instead of “V2” copies. Use existing patterns for CLI options and dependency injection via `createDefaultDeps`.
- Aim to keep files under ~700 LOC; guideline only (not a hard guardrail). Split/refactor when it improves clarity or testability.
- Naming: use **OpenClaw** for product/app/docs headings; use `openclaw` for CLI command, package/binary, paths, and config keys.
- Written English: use American spelling and grammar in code, comments, docs, and UI strings (e.g. "color" not "colour", "behavior" not "behaviour", "analyze" not "analyse").

## Release / Advisory Workflows

- Use `$openclaw-release-maintainer` at `.agents/skills/openclaw-release-maintainer/SKILL.md` for release naming, version coordination, release auth, and changelog-backed release-note workflows.
- Use `$openclaw-ghsa-maintainer` at `.agents/skills/openclaw-ghsa-maintainer/SKILL.md` for GHSA advisory inspection, patch/publish flow, private-fork checks, and GHSA API validation.
- Release and publish remain explicit-approval actions even when using the skill.

## Testing Guidelines

- Framework: Vitest with V8 coverage thresholds (70% lines/branches/functions/statements).
- Naming: match source names with `*.test.ts`; e2e in `*.e2e.test.ts`.
- Run `pnpm test` (or `pnpm test:coverage`) before pushing when you touch logic.
- Agents MUST NOT modify baseline, inventory, ignore, snapshot, or expected-failure files to silence failing checks without explicit approval in this chat.
- For targeted/local debugging, keep using the wrapper: `pnpm test -- <path-or-filter> [vitest args...]` (for example `pnpm test -- src/commands/onboard-search.test.ts -t "shows registered plugin providers"`); do not default to raw `pnpm vitest run ...` because it bypasses wrapper config/profile/pool routing.
- Do not set test workers above 16; tried already.
- If local Vitest runs cause memory pressure (common on non-Mac-Studio hosts), use `OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test` for land/gate runs.
- Live tests (real keys): `CLAWDBOT_LIVE_TEST=1 pnpm test:live` (OpenClaw-only) or `LIVE=1 pnpm test:live` (includes provider live tests). Docker: `pnpm test:docker:live-models`, `pnpm test:docker:live-gateway`. Onboarding Docker E2E: `pnpm test:docker:onboard`.
- Full kit + what’s covered: `docs/help/testing.md`.
- Changelog: user-facing changes only; no internal/meta notes (version alignment, appcast reminders, release process).
- Changelog placement: in the active version block, append new entries to the end of the target section (`### Changes` or `### Fixes`); do not insert new entries at the top of a section.
- Changelog attribution: use at most one contributor mention per line; prefer `Thanks @author` and do not also add `by @author` on the same entry.
- Pure test additions/fixes generally do **not** need a changelog entry unless they alter user-facing behavior or the user asks for one.
- Mobile: before using a simulator, check for connected real devices (iOS + Android) and prefer them when available.

## Commit & Pull Request Guidelines

- Use `$openclaw-pr-maintainer` at `.agents/skills/openclaw-pr-maintainer/SKILL.md` for maintainer PR triage, review, close, search, and landing workflows.
- This includes auto-close labels, bug-fix evidence gates, GitHub comment/search footguns, and maintainer PR decision flow.
- For the repo's end-to-end maintainer PR workflow, use `$openclaw-pr-maintainer` at `.agents/skills/openclaw-pr-maintainer/SKILL.md`.

- `/landpr` lives in the global Codex prompts (`~/.codex/prompts/landpr.md`); when landing or merging any PR, always follow that `/landpr` process.
- Create commits with `scripts/committer "<msg>" <file...>`; avoid manual `git add`/`git commit` so staging stays scoped.
- Follow concise, action-oriented commit messages (e.g., `CLI: add verbose flag to send`).
- Group related changes; avoid bundling unrelated refactors.
- PR submission template (canonical): `.github/pull_request_template.md`
- Issue submission templates (canonical): `.github/ISSUE_TEMPLATE/`

## Git Notes

- If `git branch -d/-D <branch>` is policy-blocked, delete the local ref directly: `git update-ref -d refs/heads/<branch>`.
- Agents MUST NOT create or push merge commits on `main`. If `main` has advanced, rebase local commits onto the latest `origin/main` before pushing.
- Bulk PR close/reopen safety: if a close action would affect more than 5 PRs, first ask for explicit user confirmation with the exact PR count and target scope/query.

## Security & Configuration Tips

- Web provider stores creds at `~/.openclaw/credentials/`; rerun `openclaw login` if logged out.
- Pi sessions live under `~/.openclaw/sessions/` by default; the base directory is not configurable.
- Environment variables: see `~/.profile`.
- Never commit or publish real phone numbers, videos, or live configuration values. Use obviously fake placeholders in docs, tests, and examples.
- Release flow: use the private [maintainer release docs](https://github.com/openclaw/maintainers/blob/main/release/README.md) for the actual runbook, `docs/reference/RELEASING.md` for the public release policy, and `$openclaw-release-maintainer` for the maintainership workflow.

## Local Runtime / Platform Notes

- Vocabulary: "makeup" = "mac app".
- Rebrand/migration issues or legacy config/service warnings: run `openclaw doctor` (see `docs/gateway/doctor.md`).
- Use `$openclaw-parallels-smoke` at `.agents/skills/openclaw-parallels-smoke/SKILL.md` for Parallels smoke, rerun, upgrade, debug, and result-interpretation workflows across macOS, Windows, and Linux guests.
- For the macOS Discord roundtrip deep dive, use the narrower `.agents/skills/parallels-discord-roundtrip/SKILL.md` companion skill.
- Never edit `node_modules` (global/Homebrew/npm/git installs too). Updates overwrite. Skill notes go in `tools.md` or `AGENTS.md`.
- If you need local-only `.agents` ignores, use `.git/info/exclude` instead of repo `.gitignore`.
- When adding a new `AGENTS.md` anywhere in the repo, also add a `CLAUDE.md` symlink pointing to it (example: `ln -s AGENTS.md CLAUDE.md`).
- Signal: "update fly" => `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/openclaw && git pull --rebase origin main'"` then `fly machines restart e825232f34d058 -a flawd-bot`.
- CLI progress: use `src/cli/progress.ts` (`osc-progress` + `@clack/prompts` spinner); don’t hand-roll spinners/bars.
- Status output: keep tables + ANSI-safe wrapping (`src/terminal/table.ts`); `status --all` = read-only/pasteable, `status --deep` = probes.
- Gateway currently runs only as the menubar app; there is no separate LaunchAgent/helper label installed. Restart via the OpenClaw Mac app or `scripts/restart-mac.sh`; to verify/kill use `launchctl print gui/$UID | grep openclaw` rather than assuming a fixed label. **When debugging on macOS, start/stop the gateway via the app, not ad-hoc tmux sessions; kill any temporary tunnels before handoff.**
- macOS logs: use `./scripts/clawlog.sh` to query unified logs for the OpenClaw subsystem; it supports follow/tail/category filters and expects passwordless sudo for `/usr/bin/log`.
- If shared guardrails are available locally, review them; otherwise follow this repo's guidance.
- SwiftUI state management (iOS/macOS): prefer the `Observation` framework (`@Observable`, `@Bindable`) over `ObservableObject`/`@StateObject`; don’t introduce new `ObservableObject` unless required for compatibility, and migrate existing usages when touching related code.
- Connection providers: when adding a new connection, update every UI surface and docs (macOS app, web UI, mobile if applicable, onboarding/overview docs) and add matching status + configuration forms so provider lists and settings stay in sync.
- Version locations: `package.json` (CLI), `apps/android/app/build.gradle.kts` (versionName/versionCode), `apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `apps/macos/Sources/OpenClaw/Resources/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `docs/install/updating.md` (pinned npm version), and Peekaboo Xcode projects/Info.plists (MARKETING_VERSION/CURRENT_PROJECT_VERSION).
- "Bump version everywhere" means all version locations above **except** `appcast.xml` (only touch appcast when cutting a new macOS Sparkle release).
- **Restart apps:** “restart iOS/Android apps” means rebuild (recompile/install) and relaunch, not just kill/launch.
- **Device checks:** before testing, verify connected real devices (iOS/Android) before reaching for simulators/emulators.
- iOS Team ID lookup: `security find-identity -p codesigning -v` → use Apple Development (…) TEAMID. Fallback: `defaults read com.apple.dt.Xcode IDEProvisioningTeamIdentifiers`.
- A2UI bundle hash: `src/canvas-host/a2ui/.bundle.hash` is auto-generated; ignore unexpected changes, and only regenerate via `pnpm canvas:a2ui:bundle` (or `scripts/bundle-a2ui.sh`) when needed. Commit the hash as a separate commit.
- Release signing/notary credentials are managed outside the repo; maintainers keep that setup in the private [maintainer release docs](https://github.com/openclaw/maintainers/tree/main/release).
- Lobster palette: use the shared CLI palette in `src/terminal/palette.ts` (no hardcoded colors); apply palette to onboarding/config prompts and other TTY UI output as needed.
- When asked to open a “session” file, open the Pi session logs under `~/.openclaw/agents/<agentId>/sessions/*.jsonl` (use the `agent=<id>` value in the Runtime line of the system prompt; newest unless a specific ID is given), not the default `sessions.json`. If logs are needed from another machine, SSH via Tailscale and read the same path there.
- Do not rebuild the macOS app over SSH; rebuilds must be run directly on the Mac.
- Voice wake forwarding tips:
  - Command template should stay `openclaw-mac agent --message "${text}" --thinking low`; `VoiceWakeForwarder` already shell-escapes `${text}`. Don’t add extra quotes.
  - launchd PATH is minimal; ensure the app’s launch agent PATH includes standard system paths plus your pnpm bin (typically `$HOME/Library/pnpm`) so `pnpm`/`openclaw` binaries resolve when invoked via `openclaw-mac`.

## Collaboration / Safety Notes

- When working on a GitHub Issue or PR, print the full URL at the end of the task.
- When answering questions, respond with high-confidence answers only: verify in code; do not guess.
- Never update the Carbon dependency.
- Any dependency with `pnpm.patchedDependencies` must use an exact version (no `^`/`~`).
- Patching dependencies (pnpm patches, overrides, or vendored changes) requires explicit approval; do not do this by default.
- **Multi-agent safety:** do **not** create/apply/drop `git stash` entries unless explicitly requested (this includes `git pull --rebase --autostash`). Assume other agents may be working; keep unrelated WIP untouched and avoid cross-cutting state changes.
- **Multi-agent safety:** when the user says "push", you may `git pull --rebase` to integrate latest changes (never discard other agents' work). When the user says "commit", scope to your changes only. When the user says "commit all", commit everything in grouped chunks.
- **Multi-agent safety:** do **not** create/remove/modify `git worktree` checkouts (or edit `.worktrees/*`) unless explicitly requested.
- **Multi-agent safety:** do **not** switch branches / check out a different branch unless explicitly requested.
- **Multi-agent safety:** running multiple agents is OK as long as each agent has its own session.
- **Multi-agent safety:** when you see unrecognized files, keep going; focus on your changes and commit only those.
- Lint/format churn:
  - If staged+unstaged diffs are formatting-only, auto-resolve without asking.
  - If commit/push already requested, auto-stage and include formatting-only follow-ups in the same commit (or a tiny follow-up commit if needed), no extra confirmation.
  - Only ask when changes are semantic (logic/data/behavior).
- **Multi-agent safety:** focus reports on your edits; avoid guard-rail disclaimers unless truly blocked; when multiple agents touch the same file, continue if safe; end with a brief “other files present” note only if relevant.
- Bug investigations: read source code of relevant npm dependencies and all related local code before concluding; aim for high-confidence root cause.
- Code style: add brief comments for tricky logic; keep files under ~500 LOC when feasible (split/refactor as needed).
- Tool schema guardrails (google-antigravity): avoid `Type.Union` in tool input schemas; no `anyOf`/`oneOf`/`allOf`. Use `stringEnum`/`optionalStringEnum` (Type.Unsafe enum) for string lists, and `Type.Optional(...)` instead of `... | null`. Keep top-level tool schema as `type: "object"` with `properties`.
- Tool schema guardrails: avoid raw `format` property names in tool schemas; some validators treat `format` as a reserved keyword and reject the schema.
- Never send streaming/partial replies to external messaging surfaces (WhatsApp, Telegram); only final replies should be delivered there. Streaming/tool events may still go to internal UIs/control channel.
- For manual `openclaw message send` messages that include `!`, use the heredoc pattern noted below to avoid the Bash tool’s escaping.
- Release guardrails: do not change version numbers without operator’s explicit consent; always ask permission before running any npm publish/release step.
- Beta release guardrail: when using a beta Git tag (for example `vYYYY.M.D-beta.N`), publish npm with a matching beta version suffix (for example `YYYY.M.D-beta.N`) rather than a plain version on `--tag beta`; otherwise the plain version name gets consumed/blocked.

## Production Server (Nawkout / dBlitz)

**Droplet:** `167.71.93.186` SSH port `2222`, path `/root/projects/zeus` (dBlitz/zeus fork).

### Access

```bash
ssh -p 2222 root@167.71.93.186
```

### OpenClaw Gateway

- **Systemd service:** `openclaw-gateway` on port `18789` (loopback)
- **Version:** v2026.3.14
- **Tailscale node:** `crm` (IP `100.124.75.80`, MagicDNS `crm.taile47c3.ts.net`)
- **Control UI:** `https://crm.taile47c3.ts.net/dashboard/` (Funnel, token auth, device auth disabled)
- **Gateway auth token:** stored in `gateway.auth.token` in `/root/.openclaw/openclaw.json`

```bash
systemctl --user status openclaw-gateway
systemctl --user restart openclaw-gateway
journalctl --user -u openclaw-gateway -f
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

### Agents

| Agent                 | Model                       | Role                                                                           |
| --------------------- | --------------------------- | ------------------------------------------------------------------------------ |
| **creator** (default) | anthropic/claude-sonnet-4-6 | Creator partnerships (Maya persona, maya@getnawkout.com + partner@nawkout.com) |
| **customer**          | openai/gpt-5-mini           | Customer lifecycle (Maya persona, care@nawkout.com)                            |
| **diagnostic**        | openai/gpt-5-mini           | Sleep diagnostic leads (Dr. Elena, elena@nawkout.com)                          |

Agent workspaces: `~/.openclaw/agents/<agentId>/workspace/` (SOUL.md, IDENTITY.md, HEARTBEAT.md, MEMORY.md, TOOLS.md)

### Channels

- **Gmail:** Push via Google Pub/Sub + Tailscale Funnel + gog CLI. Two mailboxes: `maya@getnawkout.com` (port 8788) and `partner@nawkout.com` (port 8789).
- **WhatsApp:** Enabled via Baileys (WhatsApp Web protocol). Linked device with dedicated number. DM policy: `pairing`. Allowlisted numbers: `+18324832725`, `+13464013096`. Credentials at `~/.openclaw/credentials/whatsapp/`.

### Tailscale Routes

| Path                    | Target                                        | Mode                             |
| ----------------------- | --------------------------------------------- | -------------------------------- |
| `/`                     | `http://127.0.0.1:8788` (Gmail watcher)       | Funnel (public)                  |
| `/dashboard`            | `http://127.0.0.1:18789` (Gateway/Control UI) | Funnel (public, token-protected) |
| `/gmail-pubsub`         | `http://127.0.0.1:8788`                       | Funnel (public)                  |
| `/gmail-pubsub-partner` | `http://127.0.0.1:8789`                       | Funnel (public)                  |

### Agent CLI Tools (via exec)

| Service | CLI             | Auth env var                                        |
| ------- | --------------- | --------------------------------------------------- |
| Gmail   | `gog`           | `GOG_KEYRING_PASSWORD`                              |
| Shopify | `shopify-admin` | `SHOPIFY_KEYRING_PASSWORD`                          |
| HubSpot | `hubspot`       | `HUBSPOT_KEYRING_PASSWORD` + `HUBSPOT_ACCESS_TOKEN` |

Auth env vars stored in `~/.openclaw/.env`.

### Backup

```bash
cd /root && openclaw backup create
```

Creates timestamped `.tar.gz` of `~/.openclaw/` (config, credentials, sessions, agent workspaces). Contains secrets in cleartext; encrypt before offsite storage.

### Memory and Context Retention

OpenClaw's golden rule: **if it's not written to a file, it doesn't exist.** The model only "remembers" what gets written to disk. Instructions typed in conversation don't survive compaction.

**Current config (applied 2026-03-21):**

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 8000,
          systemPrompt: "Extract only what is worth remembering. No fluff.",
          prompt: "Session nearing compaction. If the daily memory file already exists, READ it first and APPEND new entries. Save to memory/daily/YYYY-MM-DD.md: who was discussed, what actions were taken, decisions made, and pending tasks. Reply NO_FLUSH if nothing worth storing.",
        },
      },
    },
  },
  channels: {
    whatsapp: {
      dmHistoryLimit: 20, // injects last 20 DM messages as context on each turn
    },
  },
}
```

**How compaction flush works:**

- Triggers when session tokens cross `contextWindow - reserveTokensFloor - softThresholdTokens`
- Fires a silent agentic turn telling the model to save important context to disk
- Default `softThresholdTokens` (4000) was too small; 8000 gives enough headroom for multi-file writes
- The custom prompt includes an **append instruction** to prevent overwriting previous daily memory entries (known bug with default prompt)

**WhatsApp DM history injection:**

- `dmHistoryLimit: 20` injects recent unprocessed messages under `[Chat messages since your last reply - for context]`
- Default for groups is 50; DMs were unset (no injection)
- Helps Maya retain conversation context across turns even without persistent memory

### Memory Plugins (Persistent / Vector-Based)

**Built-in options (ship with OpenClaw):**

| Plugin           | Status       | What it does                                                                                                                                          |
| ---------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `memory-core`    | **loaded**   | File-based memory with `memory_search` and `memory_get` tools. Simple, transparent, works.                                                            |
| `memory-lancedb` | **disabled** | LanceDB vector DB with auto-recall (injects relevant memories before each response) and auto-capture (saves after each turn). Uses OpenAI embeddings. |

**Third-party options:**

| Plugin                           | Type                 | Notes                                                                                                                       |
| -------------------------------- | -------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `mem0` (@mem0/openclaw-mem0)     | Cloud or self-hosted | Auto-recall/capture, supports OpenAI-compatible providers. Requires Mem0 account or self-hosted instance.                   |
| `supermemory`                    | Cloud                | No local infra needed. Auto-recall/capture with user profiles. Requires Supermemory account.                                |
| `memory-lancedb-pro` (community) | Local                | Enhanced version with hybrid retrieval (vector + BM25), cross-encoder reranking, per-agent isolation, Weibull memory decay. |
| `openclaw-mem`                   | Local                | Trust-aware memory with provenance tracking. Scored highest in benchmarks (hit@k=0.92).                                     |

**memory-lancedb known issues (as of 2026-03):**

- Native dependency `@lancedb/lancedb` fails to install out of the box; requires manual `npm install` in the extension directory, and updates can wipe it
- All agents share one database by default (memory bleed between agents); per-agent isolation requires `memory-lancedb-pro` or `{agentId}` templating
- Opaque storage; hard to browse or correct stored memories
- Benchmarks show `openclaw-mem` scores higher on retrieval accuracy

**Recommendation:** Start with `memory-core` + compaction flush (current setup). If context loss persists, try `memory-lancedb` but be prepared to manually install native deps on the server. For production multi-agent setups, `memory-lancedb-pro` with per-agent isolation is the better choice.

**Enable memory-lancedb (when ready):**

```bash
openclaw config set plugins.entries.memory-lancedb.enabled true
systemctl --user restart openclaw-gateway
# If module loading fails:
cd /usr/lib/node_modules/openclaw/extensions/memory-lancedb && npm install
```

### Control UI

- **URL:** `https://crm.taile47c3.ts.net/dashboard/`
- **Auth:** Gateway token (from `gateway.auth.token` in config)
- **Mode:** Funnel (public HTTPS, token-protected), device auth disabled
- **Features:** Real-time reasoning view, tool calls, sessions, chat, agents, cron jobs, usage

### Voice, Talk Mode, and Node Setup Status (as of 2026-03-21)

**Talk Mode (ElevenLabs TTS) — server config DONE, needs macOS app to use:**

```json5
{
  talk: {
    provider: "elevenlabs",
    voiceId: "21m00Tcm4TlvDq8ikWAM", // Rachel (warm female)
    modelId: "eleven_v2",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

- ElevenLabs API key stored in `~/.openclaw/.env` (`ELEVENLABS_API_KEY`)
- Free tier: 20 min/month. Instant voice cloning available from Starter ($5/month).
- Talk Mode requires a **node device** (macOS app, iOS app, or Android app) connected to the gateway — the node handles the mic loop, gateway handles model + TTS.

**Voice Wake ("Hey Maya") — BLOCKED on Picovoice approval:**

- Picovoice account created, awaiting trial approval email
- Once approved: create custom wake word "Hey Maya" at console.picovoice.ai, download `.ppn` file for macOS, set `PORCUPINE_MODEL_PATH` in env
- Uses Porcupine engine for offline wake word detection (no internet needed for detection)

**macOS Companion App — BLOCKED, needs Xcode build:**

- Source at `apps/macos/` (Swift/SwiftUI, requires full Xcode IDE, not just Command Line Tools)
- Not distributed as a pre-built DMG in GitHub releases; no Homebrew cask exists
- Required for both Talk Mode and Voice Wake on Mac
- Build steps: Xcode install (~12GB) → `apps/macos/` → build and sign
- The CLAUDE.md note about `scripts/package-mac-app.sh` is for distribution packaging, not initial build

**iPhone as Node — BLOCKED, no public App Store release:**

- Official OpenClaw iOS app is super-alpha, internal-use only
- TestFlight beta: `https://testflight.apple.com/join/cutCZt5s` (called "Aight")
- Third-party alternatives on App Store: GoClaw, ClawHub, OpenClaw Messenger (MyClaw)
- Once installed: Settings → Gateway → enter `crm.taile47c3.ts.net` port `443` → approve via `openclaw nodes approve <id>` on server
- Capabilities: camera snap/clip, screen recording, location, notifications

**What's fully working now:**

| Feature                  | Status                                                                     |
| ------------------------ | -------------------------------------------------------------------------- |
| WhatsApp channel (Maya)  | Working — linked, dmPolicy=pairing, allowFrom configured                   |
| Control UI dashboard     | Working — `https://crm.taile47c3.ts.net/dashboard/`                        |
| Memory/context retention | Configured — flush threshold 8000, custom append prompt, dmHistoryLimit 20 |
| Talk Mode server config  | Done — ElevenLabs voice "Rachel" configured                                |
| Voice Wake               | Blocked — awaiting Picovoice approval                                      |
| macOS Companion App      | Blocked — needs full Xcode to build from source                            |
| iPhone Node              | Blocked — TestFlight alpha or third-party apps only                        |

### Agent Architecture Plan (Nawkout / dBlitz)

**Design principle:** Organize agents by FUNCTION, not by platform. One agent managing "all Meta stuff" is wrong. One agent managing "all paid media" across Meta + Google + TikTok is right.

**Tool count rule:** Research shows 5-7 tools is the optimal max per agent. Beyond that, LLM tool selection accuracy degrades exponentially — the model spreads attention thin, picks wrong tools, hallucinates parameters. Keep each agent under 7 tools.

**Agent team (5 agents — Chief of Staff + 4 specialists):**

| Agent | Function | Tools | Model | Channel routing |
|-------|----------|-------|-------|----------------|
| **Chief** (default) | Personal Chief of Staff — routing, morning briefing, revenue pulse, gap detection, cross-agent coordination | gog (Gmail), hubspot, shopify-admin, sessions_spawn, sessions_send | openai/gpt-5-mini | WhatsApp DMs (default), Control UI |
| **Maya** (creator) | Creator partnerships — outreach, relationship mgmt, CRM, resupply | gog (Gmail), hubspot, shopify-admin, meta-cli (creator ad perf only) | anthropic/claude-sonnet-4-6 | Gmail inbound, delegated from Chief |
| **Care** (customer) | Customer lifecycle, support, reviews, retention, diagnostic leads | gog (Gmail), shopify-admin, hubspot | openai/gpt-5-mini | Email channels, delegated from Chief |
| **Ads** (paid media) | Paid media across ALL platforms — budgets, campaigns, creatives, performance | google-cli ads, meta-cli ads, tiktok-cli business, meta-cli capi | openai/gpt-5-mini | Delegated from Chief |
| **Growth** (organic) | Analytics, SEO, organic social, content, shop operations | google-cli ga4, google-cli gsc, meta-cli instagram, meta-cli pages, tiktok-cli shop | openai/gpt-5-mini | Delegated from Chief |

**Why a Chief of Staff agent:**
- Without it, nobody watches the big picture (revenue, cross-agent gaps, fallen-through-the-cracks)
- Morning briefing: overnight activity summary across all agents, revenue pulse, priority recommendations
- Smart routing: "check on Rachel" → delegates to Maya; "pause Meta ads" → delegates to Ads
- Gap detection: "3 creators haven't heard from us in 7+ days" (cross-references HubSpot with Maya's activity)
- Direct answers for simple queries: "what's our MRR?" → Shopify query, no delegation needed
- Cost: ~$3-5/month on GPT-5-mini (cheap — it's routing/synthesis, not creative writing)
- Research confirms: single agents fail 35% of complex tasks, multi-agent teams hit 92% success through specialization

**Chief of Staff workspace files:**

| File | Content |
|------|---------|
| `SOUL.md` | Personality: warm but sharp, direct, no fluff. Core values: Anticipation ("best help is help you didn't ask for"), Ownership ("solutions not excuses"), Revenue-focus ("your #1 job is making sure Devin is successful and the business makes money"). Anti-patterns: no excessive affirmations, no theatre, no padding. |
| `IDENTITY.md` | Name: Chief. Role: Chief of Staff for Devin at Nawkout/dBlitz. |
| `AGENTS.md` | Routing rules: which specialist handles what. When to delegate vs answer directly. Cross-agent coordination protocols. |
| `USER.md` | Devin's preferences, communication style, business context, timezone (CT). |
| `HEARTBEAT.md` | Periodic checks: fallen-through-the-cracks creators, stale HubSpot tasks, unread Gmail. Keep tiny (<2000 chars). |
| `TOOLS.md` | CLI reference for gog, hubspot, shopify-admin (direct access for simple queries). |

**Morning briefing (cron, not heartbeat — exact timing):**
```bash
openclaw cron add \
  --name "Morning briefing" \
  --cron "0 13 * * *" \
  --tz "UTC" \
  --agent chief \
  --session isolated \
  --message "Run morning briefing: 1) Check overnight Gmail activity across all mailboxes 2) Shopify revenue last 24h 3) HubSpot pipeline: any creators with 7+ days no activity 4) Any pending tasks across agents 5) Send concise summary to WhatsApp."
```

**Cross-agent communication (requires config):**
- `sessions_spawn` — async delegation (non-blocking, result announced back). Requires `allowAgents` config.
- `sessions_send` — sync message to another agent (waits for response). Requires `agentToAgent` config (disabled by default).
- Chief needs both enabled; specialists only need to accept incoming.

**Heartbeat best practices (official):**
- Keep HEARTBEAT.md tiny (short checklist) to avoid token burn
- Use `isolatedSession: true` and `lightContext: true` for cost optimization
- Agent replies `HEARTBEAT_OK` if nothing needs attention (dropped silently)
- Empty HEARTBEAT.md = skip run entirely
- Known bug (#14986): per-agent heartbeat intervals ignored in multi-agent; use cron for per-agent timing instead

**Why 4 specialist agents (not 3):**
- Putting all new CLIs (google-cli + meta-cli + tiktok-cli) on one "Ops" agent = ~25 tool actions. Way over the 5-7 optimal threshold.
- Splitting into Ads (paid) + Growth (organic) keeps each at 5-6 tools.

**Why not per-platform agents (Meta agent, Google agent, TikTok agent):**
- One ad campaign optimization decision often spans multiple platforms ("shift budget from Google to Meta").
- The Ads agent needs cross-platform visibility to make these calls.
- Per-platform agents can't see the full picture.

**Tool isolation (security):**
- Maya: CRM read/write, email send, Shopify orders — NO ad budget controls
- Care: customer data, support — NO creator outreach, NO ad controls
- Ads: campaign CRUD, spend controls — NO email send, NO CRM write
- Growth: analytics read-only, content publish — NO ad spend, NO email send

**Model selection rationale (match model to task type, not one model for all):**

Pricing context (March 2026):
- Claude Sonnet 4.6: $3/$15 per 1M tokens (input/output)
- GPT-5-mini: $0.25/$2 per 1M tokens — **60x cheaper on input**
- Gemini 3 Flash: ~$0.001/msg — cheapest, fastest

| Agent | Model | Why this model |
|-------|-------|---------------|
| **Maya** | claude-sonnet-4-6 | Only agent doing creative, relationship-driven work. Email drafting requires nuance, warmth, personality matching. Claude leads at "voice" — sounds human, avoids AI-sounding language. GPT-5-mini would sound generic for creator outreach. Worth the premium. |
| **Care** | gpt-5-mini | Customer support is structured — lookup order, check status, draft templated response. Speed matters more than nuance (quick replies). Doesn't need Claude's creative writing ability. 60x cheaper for high-volume support. |
| **Ads** | gpt-5-mini | Ads management is analytical — read metrics, compare numbers, decide budget allocation. No creative writing needed. Tool calling accuracy is similar across models at 5-6 tools. Cheap + fast wins for periodic checks. |
| **Growth** | gpt-5-mini | Analytics and SEO are data-heavy report generation. Structured output (JSON metrics). Could also use Gemini 3 Flash for Google ecosystem integration (GA4, GSC). GPT-5-mini is the safe default. |

**Key insight:** Only Maya needs an expensive model. The other 3 do structured, analytical tasks where a cheap fast model performs equally well. Mixed-model approach saves 50-70% vs running everything on Claude Sonnet.

**Estimated monthly cost:**
- Maya (Claude Sonnet): ~$15-30/mo
- Care (GPT-5-mini): ~$2-5/mo
- Ads (GPT-5-mini): ~$1-3/mo
- Growth (GPT-5-mini): ~$1-3/mo
- **Total: ~$20-40/mo** (vs $60-120/mo all-Claude)

### CLI Tools for Agents

**Existing (production):**

| CLI | Repo | Binary | Status |
|-----|------|--------|--------|
| `gog` (Gmail/Google Workspace) | external (gogcli) | `gog` | Installed on server |
| `shopify-admin` | `dblitz/shopify-admin-cli` | `shopify-admin` + `shopify-admin-linux-amd64` | Installed on server |
| `hubspot` | `dblitz/hubspot-cli` | `hubspot` + `hubspot-linux-amd64` | Installed on server |

**New (built 2026-03-22, same Go + keyring + JSON pattern):**

| CLI | Repo | APIs bundled | API versions (2026) | macOS | Linux |
|-----|------|-------------|-------------------|-------|-------|
| `google-cli` | `dblitz/google-cli` | Google Ads + GA4 + GSC (one OAuth2 token, multi-scope) | Ads API v23.1, Analytics Data API v1beta, Search Console API v3/v1 | 9.2M | 9.9M |
| `meta-cli` | `dblitz/meta-cli` | Meta Ads + Instagram + Facebook Pages + CAPI (one Graph API token) | Graph API v25.0 | 8.6M | 9.3M |
| `tiktok-cli` | `dblitz/tiktok-cli` | TikTok Business + Shop (separate auth + HMAC-SHA256 signing) | Business API v1.3, Shop Partner API v2 | 8.6M | 9.4M |

**Build status:** All 3 DONE (macOS arm64 + Linux amd64 binaries). Not yet deployed to production server.

**Deploy to production (when ready):**
```bash
scp -P 2222 /Users/devin/dblitz/google-cli/google-cli-linux-amd64 root@167.71.93.186:/usr/local/bin/google-cli
scp -P 2222 /Users/devin/dblitz/meta-cli/meta-cli-linux-amd64 root@167.71.93.186:/usr/local/bin/meta-cli
scp -P 2222 /Users/devin/dblitz/tiktok-cli/tiktok-cli-linux-amd64 root@167.71.93.186:/usr/local/bin/tiktok-cli
```

**CLI pattern (all follow hubspot-cli/shopify-admin-cli):**
- Go binary, single compiled binary per platform (macOS arm64 + Linux amd64)
- Keyring auth via `github.com/99designs/keyring` (macOS Keychain, Linux Secret Service)
- All output is JSON to stdout, errors to stderr
- Called by OpenClaw agents via `exec` tool
- Env var overrides for headless servers (e.g. `GOOGLE_CLI_KEYRING_PASSWORD`)

**Auth setup per CLI (on production server):**
```bash
# Google (OAuth2 — requires browser for consent flow)
google-cli auth add --client-id $GOOGLE_CLIENT_ID --client-secret $GOOGLE_CLIENT_SECRET --developer-token $GOOGLE_ADS_DEVELOPER_TOKEN --customer-id 9934956972 --remote

# Meta (long-lived token — generated in Meta developer portal)
meta-cli auth add --token $META_ACCESS_TOKEN --app-id $META_APP_ID --app-secret $META_APP_SECRET

# TikTok Business (OAuth2 token)
tiktok-cli auth add-business --token $TIKTOK_ACCESS_TOKEN

# TikTok Shop (app key + secret + token)
tiktok-cli auth add-shop --app-key $TTS_APP_KEY --app-secret $TTS_APP_SECRET --token $TTS_ACCESS_TOKEN
```

### Voice Call (Phone)

OpenClaw has an official `voice-call` plugin at `extensions/voice-call/` that gives your agent a real phone number via Twilio, Telnyx, or Plivo. You call the number and talk to the agent with voice.

**Note:** You CANNOT call the WhatsApp number to talk to the agent — WhatsApp Web/Baileys only handles text, not voice calls. Voice calls require a separate phone number via the voice-call plugin.

**Features:**
- Real phone number you can call
- Agent can use tools mid-call (Shopify, Gmail, HubSpot, etc.)
- TTS for agent speech (uses ElevenLabs config)
- STT for your voice input (Whisper)

**Current setup (as of 2026-03-22):**
- **Provider:** Telnyx (freemium account, info@nawkout.com)
- **Phone number:** +1-832-787-1364 (Houston, active)
- **Voice app:** "OpenClaw Maya Voice" (Connection ID: `2921090214740887401`)
- **Plugin:** voice-call loaded, webhook at `https://crm.taile47c3.ts.net/voice/webhook`
- **Config:** `skipSignatureVerification: true` (need to add `telnyx.publicKey` for production)
- **Inbound policy:** allowlist (`+18324832725`)
- **Status:** BLOCKED — Telnyx account balance is -$0.01. Need to add funds ($5 minimum) to enable call processing. Freemium portal has no billing page; need to log in as Business account (check email for password setup link after GitHub identity verification).
- **API key:** stored in `~/.openclaw/.env` (`TELNYX_API_KEY`, `TELNYX_CONNECTION_ID`)

**To unblock voice calls:**
1. Check email from Telnyx for password setup link
2. Log in at portal.telnyx.com as Business (not Freemium)
3. Add payment method and fund account with $5+
4. Calls should work immediately after balance is positive

**Third-party alternative:** ClawdTalk (by Telnyx) — gives agent a phone number with voice + WhatsApp + SMS on one number.

### Chief of Staff — Monitoring & Security Architecture

**Principle:** Chief MONITORS and ALERTS. Infrastructure-level controls run independently.

**Heartbeat (every 30m, lightContext: true):**

```markdown
# HEARTBEAT — 10 checks, trigger-based

## Business
1. Gmail: unread from VIPs/creators older than 2h -> alert
2. HubSpot: active pipeline contact with 7+ days no activity -> alert
3. Shopify: orders with issues/refunds/chargebacks in last 4h -> alert
4. Cron: failed or overdue openclaw cron jobs -> alert

## Agent Oversight
5. Maya: Gmail drafts pending review for 24h+ -> alert
6. Maya: creator conversation unanswered 48h+ -> alert
7. Care: customer support email unanswered 4h+ -> alert

## Infrastructure
8. WhatsApp: openclaw channels status --probe -> alert if NOT connected
9. Disk: df -h -> alert if partition below 20% free
10. Backup: newest /root/*backup*.tar.gz older than 26h -> alert
```

**Scheduled cron jobs (all targeting Chief):**

| Job | Cron | Time (CT) | Purpose |
|-----|------|-----------|---------|
| Daily backup | `0 4 * * *` | 10pm CT | `openclaw backup create` |
| Morning briefing | `0 13 * * *` | 7am CT | Revenue, overnight activity, priorities |
| Weekly security audit | `0 14 * * 1` | Mon 8am CT | `openclaw security audit --deep` |
| Weekly agent review | `0 22 * * 5` | Fri 4pm CT | Grade agents A/B/C, flag gaps |

**What Chief does NOT own (runs independently):**
- Gateway auto-restart: systemd `Restart=always` (already configured)
- DigitalOcean automated snapshots (see Backup below)
- OpenClaw version updates: manual `sudo npm i -g openclaw@latest`
- Credential rotation: manual
- Firewall/network changes: never through agent

### Backup & Disaster Recovery

**3-2-1 rule — fully covered:**

| Layer | What | Frequency | Retention | Status |
|-------|------|-----------|-----------|--------|
| **OpenClaw backup** (agent-level) | `openclaw backup create` via Chief daily cron | Daily 10pm CT | Rolling (manual cleanup) | Active |
| **DigitalOcean snapshot** (full server) | Automated full droplet image | Daily 12-4am UTC | 7 days | Active |
| **Backup verification** | `openclaw backup verify <archive>` | On demand | — | Available |

**DigitalOcean snapshots (confirmed active):**
- Droplet ID: `540729165`, region: `nyc3`
- Daily automated backups, 7-day retention
- Full server image (~48 GiB) — includes OS, OpenClaw, credentials, agents, everything
- Restore = one click in DO dashboard → new droplet from snapshot

**If the server dies (full disaster recovery):**
1. DigitalOcean dashboard → Backups → Restore from latest snapshot (one click)
2. `systemctl --user start openclaw-gateway`
3. Verify: `openclaw doctor && openclaw channels status --probe`

**If only OpenClaw state gets corrupted:**
1. `tar -xzf /root/LATEST-backup.tar.gz -C /`
2. `systemctl --user restart openclaw-gateway`
3. `openclaw doctor`

**If WhatsApp session dies after restore:**
1. `openclaw channels login --channel whatsapp` (re-scan QR from the dedicated phone)

**Security:** Backup archives contain secrets in cleartext (API keys, OAuth tokens, WhatsApp session keys). DO snapshots are encrypted at rest by DigitalOcean.

### Related

- Full marketing-sales operational docs: `../marketing-sales/CLAUDE.md`
- WhatsApp channel docs: `docs/channels/whatsapp.md`
- Memory docs: `docs/concepts/memory.md`
- Session docs: `docs/concepts/session.md`
- Voice Wake docs: `docs/nodes/voicewake.md` (see also `docs/platforms/mac/voicewake.md`)
- Talk Mode docs: `docs/nodes/talk.md`
- iOS app docs: `docs/platforms/ios.md` (see also `apps/ios/README.md`)
- macOS app docs: `docs/platforms/macos.md` (see also `apps/macos/README.md`)
- Tool selection research: 5-7 tools optimal per agent (see ICLR 2026 AutoTool paper)
- OpenClaw multi-agent docs: `docs/concepts/multi-agent.md`
