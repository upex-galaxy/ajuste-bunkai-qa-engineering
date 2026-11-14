# ADR-0018 — The `.env` loader runs the installed varlock; it never installs one beside it

- **Status:** Accepted (by the owner, 2026-11-13)
- **Date:** 2026-11-13
- **Deciders:** framework owner (boilerplate maintainer): session decision of 2026-11-13, "apply the durable loader fix now" (`.session/framework-development/mcp-loader-no-install/plan.md`, approved in session). The technical call inside it (`--no-install` over `bun x`) was made by the implementing session under that approval and is recorded below with the option it beat
- **Tags:** secrets, mcp, varlock, harness, windows, startup
- **Supersedes:** ADR-0011, §Decision bullet 1 only (the launch shape of the loader). The one-loader-every-host rule, the `--filter` contract, the retirement of the plaintext copies and the `--filter`/`--inject`/`--no-redact-stdout` flag set all stand
- **Superseded by:** —

---

## Context

ADR-0011 shipped the loader as `bunx -p varlock@1.20.0 varlock run --no-redact-stdout --inject vars --filter A,B -- <server>`. On Windows that launch installs varlock on every spawn, and `bunx` installs a given package into ONE shared destination per user: `%TEMP%/bunx-<uid>-varlock@1.20.0`. A host boots all of its MCP servers at once, so the three loader-launched servers (`openapi`, `slack-aurora`, `dbhub`) all start writing that one directory in the same second. All but one lose the copy and exit 1 before the MCP handshake:

```
mcp connect failed  server=dbhub
MCP server process exited with code 1:
EBUSY: failed copying files from cache to destination for package varlock
EBUSY: failed copying files from cache to destination for package @varlock/native-helper-win32-x64
```

Measured in this checkout (OpenCode log, 2026-11-13T22:31:03Z): `openapi`, `slack-aurora` and `dbhub` failed together, and the temp directory's mtime (22:32:27Z) sits one minute AFTER the failures — the winner created it while the losers died. The same class hit `openapi` alone on 2026-11-11T20:52Z and produced a `Request timed out` connect on 2026-11-12T21:44Z.

Three properties made it hard to read as a loading problem: the failure names whichever server lost the race, so it looks server-specific; it disappears once the temp directory is warm, so restarting "sometimes" fixes it; and `dbhub.toml` substitutes a literal `${DBHUB_HOST}` when a variable is missing, so the first instinct is a credential. All six `DBHUB_*` were set, and the exact `opencode.jsonc` command answered the `initialize` handshake when run alone.

## Decision

1. **The loader resolves, it does not install.** Every loader launch becomes `bunx --no-install varlock run --no-redact-stdout --inject vars --filter A,B -- <server>` (`MCP_ENV_LOADER_HEAD` in `cli/lib/agent-compatibility-contracts.ts`). `MCP_ENV_LOADER_COMMAND` stays `bunx`, so the loader's shape — and everything that reads it (`unwrapEnvLoader`, `foldBunX`, the parity comparison) — is unchanged.
2. **One version pin, not two.** `varlock` is an exact `devDependency` in `package.json`, and that is now the only place its version is stated. `--no-install` fails fast and loudly when the dependency is missing (`Could not find an existing 'varlock' binary to run. Stopping because --no-install was passed.`) instead of fetching one and racing.
3. **A legacy shape is a migration signal, not a silent break.** `LEGACY_CODEX_ENV_LOADER_ARGS` keeps the old `-p varlock@<pin>` form because that is what recognizes an unfiltered Codex config from before ADR-0011. A config still on the OLD filtered shape (`-p varlock@<pin> … --filter`) no longer parses as "through the loader", so `agents:compat:check` reports it as out of date in the boilerplate and as a WARNING downstream, naming the exact launch. The three host configs are protected or bootstrap-only, so only a hand edit can deliver the change; naming it is the most a check may do.

## Consequences

- **Positive:** the startup race is gone at its source, on every boot, warm cache or cold. Each spawn is a `node_modules` lookup instead of a package extraction, so MCP startup is faster regardless of platform. The version the loader runs is the version `bun install` installed — a config/`package.json` drift can no longer exist. And the failure mode when something IS wrong is a one-line message naming the missing binary, not a per-server `EBUSY` that mimics a credential or database problem.
- **Negative / trade-offs:** a checkout that never ran `bun install` now fails every loader launch. This is already a prerequisite (`prepare` runs on install, `bun setup` installs, `bun run worktree:provision` provisions a worktree), and the failure is explicit. A downstream project whose config is still on the `-p varlock@<pin>` shape sees a new WARNING until it edits that launch; the loader still starts, so nothing breaks before it does.
- **Neutral / follow-ups:** the `--filter`, `--inject vars` and `--no-redact-stdout` decisions of ADR-0011 are untouched, and its retirement of the plaintext copies is unaffected. The `EBUSY` behaviour is worth a row in the ADR-0006 measurements ledger with the loader row it belongs to.

## Alternatives considered

- **`bun x varlock …`** — also resolves the local binary and also avoids the temp install, but it silently falls back to installing when resolution fails, which is the same race with a smaller window. `--no-install` makes the absence loud instead.
- **Keep `bunx -p varlock@<pin>` and pre-warm the temp directory** — treats the symptom. The race returns whenever the OS cleans `%TEMP%`, which is exactly the condition that produced the 2026-11-13 failure, and nothing in the repo can observe or prevent that.
- **A separate `TMPDIR`/`TEMP` per server** — removes the collision by giving each server a private cache, so the same package is extracted three times per boot and the config carries a wart no other host needs.
- **Bump the pin to a newer varlock (1.23.0)** — the copy step is bun's, not varlock's; the version does not address it.
- **A per-server startup delay** — a sleep long enough to be reliable is long enough to blow Codex's 30-second startup budget, and it couples three configs' timing.

## References

- `.session/framework-development/mcp-loader-no-install/plan.md` — measurements, candidates, verification checklist.
- `cli/lib/agent-compatibility-contracts.ts` — `MCP_ENV_LOADER_*`, `mcpEnvLoaderArgs()`, `unwrapEnvLoader()`.
- ADR-0011 — the loader, the filter contract and the retirement of the plaintext copies.
- ADR-0006 — measurement provenance; this decision was measured, not assumed.
