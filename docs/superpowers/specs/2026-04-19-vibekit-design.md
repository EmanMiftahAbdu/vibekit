# Vibekit — v1 Design

**Status:** Draft for approval
**Date:** 2026-04-19
**Author:** Eman Miftah Abdu
**Repo:** `github.com/EmanMiftahAbdu/vibekit`
**npm package (planned):** `vibe-kit`

---

## Summary

Vibekit is a single-install layer for Claude Code that makes the explosion of agents, skills, and hooks "just work" for novice users ("vibe coders"). Install once, and Claude Code transparently routes work to the right specialists, adapts to the project you're in, and remembers what you were doing across sessions — without the user having to learn what's installed or which command to type.

## Problem

Claude Code's plugin ecosystem (ECC, Superpowers, claude-mem, awesome-claude-code, etc.) has grown to hundreds of agents, skills, and hooks. The gap is not capability — the gap is **discoverability and orchestration for non-experts**. Today, a novice who installs ECC and Superpowers gets ~250 skills with no idea which apply when, no cross-session memory by default, and no project-aware tool selection. Power users assemble their own meal; vibe coders walk away.

## Differentiation

Three claims nobody else owns today:

1. **Invisible cross-ecosystem routing.** Detect whatever the user already has installed (ECC, Superpowers, claude-mem, custom plugins) and unify it behind one transparent dispatch layer. Every existing orchestrator (`wshobson/agents`, `ruflo`, `superpowers`) makes routing an explicit user surface (slash commands, mode switches, plugin installs). We hide it.
2. **Memory + routing as one product.** `claude-mem` owns continuity. `wshobson/agents` owns routing. Nothing fuses them with zero per-project config.
3. **Novice-first on the Claude Code CLI.** Roo Code proved auto-routing UX works for novices, but only inside an IDE. The CLI surface — where the agent/skill explosion actually lives — is wide open.

## Success criteria (v1 done-bar)

A new user can:
1. Run `npx vibe-kit install` on a fresh machine in under 60 seconds, with no config questions.
2. Open a Next.js project and a Django project in two terminals, and observe that `claude` in each session has different relevant specialists in context — without doing anything different.
3. Close a Claude session, reopen the next day, and have prior context restored automatically (via claude-mem).
4. Run `vibe-kit doctor` and get a clear, novice-readable diagnostic of what's wired up.

## Non-goals (v1)

- Building a new memory engine (we depend on `claude-mem`).
- Replacing Claude Code's existing skill auto-activation mechanism (we feed it better inputs).
- Supporting non-Claude-Code AI tools (Cursor, Cline, etc.).
- A GUI, web dashboard, or paid tier.
- Per-project configuration UI.

## Architecture

Six small components, each replaceable in isolation.

### 1. `vibe-kit` CLI
Node/TypeScript CLI distributed via npm under the package name `vibe-kit`. Commands:
- `vibe-kit install` — global, idempotent setup (~30s, no config questions).
- `vibe-kit uninstall` — clean removal of hooks and state.
- `vibe-kit doctor` — diagnostic: what's installed, what's wired, what's missing.
- `vibe-kit which` — for the current project, show what would activate.
- `vibe-kit update` — refresh starter bundle and pinned skill versions.

### 2. Installer
The `install` command:
- Patches `~/.claude/settings.json` to register a `SessionStart` hook that points to `~/.claude/vibekit/hooks/session-start.js`.
- Creates `~/.claude/vibekit/` for state, logs, and the cached manifest.
- Installs `claude-mem` (if not already present) using its standard installer; supports `--no-memory` flag to opt out.
- Installs the starter bundle (`@vibe-kit/defaults`).
- Idempotent: running twice is a no-op or a clean upgrade.

### 3. Starter bundle (`@vibe-kit/defaults`)
A thin npm package that is purely a versioned dependency manifest. It pins ~10–15 hand-picked skill plugins (initial pick: `planner`, `brainstorming`, `code-reviewer`, `debugging`, `tdd-workflow`, `search-first`, `frontend-design`, `design-review`, `frontend-patterns`, `backend-patterns`, `git-workflow`). The exact list is curated and updated independently of the CLI.

### 4. Project detector
A pure, stateless function. Given a project root path, reads well-known manifest files and returns a list of relevant skill IDs:
- `package.json` → JS/TS, Next.js, React, Vue, Express, etc.
- `pyproject.toml` / `requirements.txt` → Python, Django, FastAPI, Flask.
- `Cargo.toml` → Rust.
- `go.mod` → Go.
- `Gemfile` → Ruby/Rails.
- `pubspec.yaml` → Flutter/Dart.
- `composer.json` → PHP/Laravel.
- (extensible map; unknown stacks return empty list)

### 5. Manifest builder
Runs inside the SessionStart hook. Inputs:
- Globally installed skills/agents discovered by scanning `~/.claude/skills/`, `~/.claude/agents/`, and `~/.claude/plugins/`.
- Project-relevant skill IDs from the project detector.
- Presence/absence of `claude-mem`.

Output: a single Markdown manifest (`available specialists for this session`) listing each skill with its name, one-line description, and when-to-use trigger. Cached to `~/.claude/vibekit/manifest-cache/<project-hash>.md` for fast subsequent loads. Cache invalidated on mtime change of project manifest files.

### 6. Context injector (the SessionStart hook itself)
A short Node script. On every Claude Code session start:
1. Build the manifest (via 4 + 5).
2. Emit it as `additionalContext` along with a short meta-instruction: *"For any task, scan the manifest below first and dispatch to the most relevant specialist before doing the work yourself."*
3. If `claude-mem` is installed, defer to its memory restoration (we do not duplicate that work).

There is no separate "router" process. Routing is performed by Claude itself, on the model the user is already running, using the unified manifest we inject.

## Data flow

**At install (one-time):**
```
user: npx vibe-kit install
  → CLI patches ~/.claude/settings.json (adds SessionStart hook)
  → CLI creates ~/.claude/vibekit/ state dir
  → CLI installs claude-mem (skip if present, or if --no-memory)
  → CLI installs @vibe-kit/defaults bundle
  → CLI verifies and prints summary
```

**Per session:**
```
user: claude (in any project dir)
  → Claude Code fires SessionStart hook
  → vibekit hook scans project + global plugins
  → Manifest built (cached if unchanged) [~100ms]
  → Manifest + meta-instruction emitted as additionalContext
  → claude-mem hook restores prior session memory
  → Claude session begins with unified, project-aware context
```

## Error handling

**Principle: the SessionStart hook must NEVER block the session.**

- Every failure path degrades gracefully: a broken manifest builder results in a session with no extra context, not a session that fails to start.
- All failures log to `~/.claude/vibekit/logs/session-start.log` with timestamp, error, and stack trace.
- A `vibe-kit doctor` command surfaces these failures to the user post-hoc.
- Hook execution is wrapped in a 2-second timeout; if exceeded, it exits with no context emitted.
- Manifest cache is invalidated when project files (`package.json`, etc.) change mtime.

## Testing strategy

| Layer | Approach |
|---|---|
| Project detector | Pure-function unit tests (table-driven: input fixtures of project roots → expected skill ID lists) |
| Manifest builder | Unit tests with mocked filesystem |
| Installer | Integration tests in temp dir: run install → verify settings.json patched, state dir created |
| SessionStart hook | Integration test: invoke hook with mock Claude Code env vars, assert correct stdout |
| End-to-end | Spin up a real Claude Code session against a fixture project, assert manifest appears in transcript |

Coverage target: 80% on detector + manifest builder (the load-bearing logic).

## Open questions (resolve during planning)

1. **claude-mem dependency model.** Auto-install during `vibe-kit install` for zero-effort UX, with `--no-memory` opt-out flag. (Tentative decision; revisit during planning if it complicates the installer.)
2. **Skill ID normalization.** ECC skills use `everything-claude-code:<name>`; Superpowers uses `superpowers:<name>`. Manifest builder needs a normalization layer — design during planning.
3. **Manifest size limits.** If a user has 250+ skills installed, the injected manifest could bloat session context. Need a heuristic to cap or rank — defer to v1.1 if cap-by-relevance proves complex.
4. **Distribution.** npm publish only for v1, or also a Homebrew tap? Defer.

## Future evolution (out of v1 scope)

- **v1.1:** Profile-based bundles (`vibe-kit install --profile=frontend`) for users who want a tailored set.
- **v1.2:** Lazy install — auto-install relevant skills when a new stack is detected mid-session.
- **v2.0:** PR upstream to ECC and Superpowers adding a shared "skill manifest" metadata format that vibekit consumes natively.
