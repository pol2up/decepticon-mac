# decepticon-mac fork — Design Spec

**Status:** approved (2026-05-26)
**Owner:** pol2up
**Upstream:** [PurpleAILAB/Decepticon](https://github.com/PurpleAILAB/Decepticon)
**Fork:** [pol2up/decepticon-mac](https://github.com/pol2up/decepticon-mac)
**Quality bar:** ship locally **and** PR upstream (each fix dual-tracked)

---

## 1. Context & Motivation

Over a single session using Decepticon v1.1.2 on macOS (Apple Silicon, Darwin 25.2.0) we encountered eight distinct, reproducible bugs — several of them blocking. The OAuth-with-Claude-Max integration in particular fails repeatedly because of an architectural mismatch between Decepticon's "credentials live in a file" assumption (true on Linux) and macOS's Keychain-first credential model. Other bugs include an agent-stuck-in-tool-call-loop (no recovery middleware catches it), a `langgraph dev`-mode blocking-error that aborts subagent dispatch, and irreversible engagement deletion (`rm -rf` with no Trash fallback on macOS).

This spec defines a **parallel-install fork** named `decepticon-mac` that:
1. Coexists with stock `decepticon` on the same machine (separate binary, paths, ports, Docker stack).
2. Carries fixes for all eight bugs.
3. Keeps each fix as a **clean upstream-PR diff** so we both ship locally and contribute back.

The upstream codebase is already parameterized for side-by-side stacks (`DECEPTICON_STACK_NAME`, port env vars), so the renaming layer is small (~3 source files) and the rest of the diff is just the fixes themselves.

## 2. Bug Catalog

| # | Bug | macOS-specific? | Existing upstream issue/PR |
|---|---|---|---|
| 1 | `langgraph dev` BlockingError in subagent dispatch | no | **#295** open |
| 2 | OAuth race: `~/.claude/.credentials.json` ↔ macOS Keychain | yes | none |
| 3 | LiteLLM `auth/` handler doesn't honor `CLAUDE_CODE_OAUTH_TOKEN` | no | none |
| 4 | `write_file` stuck looping when `content` param omitted; `MentorMiddleware` doesn't catch | no | none (MentorMiddleware exists per #248) |
| 5 | Engagement delete = `rm -rf workspace/<slug>` (unrecoverable) | indirect (macOS users expect Trash) | none |
| 6 | CLI front-end doesn't auto-reconnect after LiteLLM/LangGraph restart | no | none |
| 7 | Web dashboard + terminal WS unauthenticated, no opt-in for auth | no | none |
| 8 | Sandbox `<defunct>` tmux/bash zombie processes | no | none |

## 3. Architecture

```
GitHub                                  Local
─────────────────────                   ───────────────────────────────────────
PurpleAILAB/Decepticon ──(fork)──▶  pol2up/decepticon-mac
   (upstream main)                   ├── main         ←(tracks upstream, never diverges)
                                     ├── chore/configurable-image-namespace
                                     ├── feat/configurable-product-name
                                     ├── fix/oauth-token-env-var
                                     ├── fix/oauth-mac-keychain
                                     ├── fix/blocking-error-default
                                     ├── fix/write-file-content-loop
                                     ├── fix/engagement-soft-delete
                                     ├── fix/cli-auto-reconnect
                                     ├── fix/web-optional-auth
                                     ├── fix/sandbox-tmux-zombies
                                     └── mac/daily-driver
                                          │
                                          ▼ make install
                                /usr/local/bin/decepticon-mac
                                ~/.decepticon-mac/
                                Docker stack DECEPTICON_STACK_NAME=mac
                                Ports 3010/3013/2034/4010/7484/7697/5442
                                Images ghcr.io/pol2up/decepticon-mac-*  (local-only)
```

**Branch invariants:**
- `main` is byte-equal to upstream `main`. Never committed to directly. Synced via `git fetch upstream && git reset --hard upstream/main`.
- Every fix lives on its own `fix/<name>` (or `chore/`, `feat/`) branch off `main`. Each branch is what we push to its upstream PR — a clean focused diff.
- `mac/daily-driver` is the integration branch: `main` + every accepted local fix (squash-merged). What we build & install from.

**Key principle:** the *source code* is upstream-clean. All the "separate app" mechanics (binary name, paths, ports, stack name) live in the installer/Makefile/build config, not in Python or TypeScript code. That keeps each PR diff small.

## 4. Install Pipeline

### 4.1 Paths & names

| Aspect | Upstream | `decepticon-mac` |
|---|---|---|
| Launcher CLI | `/usr/local/bin/decepticon` | `/usr/local/bin/decepticon-mac` (symlink) |
| Binary location | `~/.local/bin/decepticon` | `~/.local/bin/decepticon-mac` |
| Data / config | `~/.decepticon/` | `~/.decepticon-mac/` |
| Workspace | `~/.decepticon/workspace/` | `~/.decepticon-mac/workspace/` |
| Credentials mount | `~/.claude/.credentials.json` (RO) | same (RO, shared, no contention after fix #1) |

### 4.2 Docker stack — uses upstream's existing env parameterization (no compose-file edits required)

```env
DECEPTICON_STACK_NAME=mac         # → container_name: decepticon-mac-langgraph, etc.
DECEPTICON_HOME=~/.decepticon-mac
WEB_PORT=3010                     # upstream default 3000
TERMINAL_PORT=3013                # upstream default 3003
LANGGRAPH_PORT=2034               # upstream default 2024
LITELLM_PORT=4010                 # upstream default 4000
NEO4J_HTTP_PORT=7484              # upstream default 7474
NEO4J_BOLT_PORT=7697              # upstream default 7687
POSTGRES_PORT=5442                # upstream default 5432
```

### 4.3 Images
- Upstream pulls `ghcr.io/purpleailab/decepticon-*:1.1.2` from public GHCR.
- Fork builds locally: `ghcr.io/pol2up/decepticon-mac-*:dev`, never pushed.
- **Enabler `chore/configurable-image-namespace`** replaces the hard-coded prefix in `docker-compose.yml` with `${DECEPTICON_IMAGE_NAMESPACE:-ghcr.io/purpleailab}` so the fork can override.

### 4.4 Installer
A `Makefile` in the fork's root with targets:

| Target | What it does |
|---|---|
| `make launcher` | Builds Go launcher with ldflags `-X main.productName=decepticon-mac -X main.defaultHome=~/.decepticon-mac` → `~/.local/bin/decepticon-mac` |
| `make images` | `docker buildx bake` against `mac/daily-driver` → local images tagged `ghcr.io/pol2up/decepticon-mac-*:dev` |
| `make install` | runs `launcher` + `images` + creates `/usr/local/bin/decepticon-mac` symlink + seeds `~/.decepticon-mac/.env` with the port-shifted defaults if absent |
| `make uninstall` | removes binary, symlink, and (with `-i` confirmation) `~/.decepticon-mac/` |
| `make dev-shell` | drops user into a shell with `PATH` rewritten so `decepticon-mac` resolves from fork checkout |
| `make sync-upstream` | fetches upstream/main, rebases local main, rebases all open fix/* branches |

### 4.5 Source-code touch list — the renaming layer
Exactly three upstream-PR-able commits compose the install renaming layer; everything else is in the per-bug branches:

1. `chore/configurable-image-namespace` — 1 line in `docker-compose.yml` (env-override the GHCR prefix).
2. `feat/configurable-product-name` — `clients/launcher/cmd/root.go` + a few callers — read product name + data-dir default from Go ldflags. Same Go binary code becomes either `decepticon` or `decepticon-mac` at build time.
3. `.env.example` — document `DECEPTICON_IMAGE_NAMESPACE` and the port overrides (the port vars already exist; just better documented).

Everything else stays bit-for-bit identical to upstream; our `fix/*` branches diff cleanly against upstream's `main`.

## 5. Per-Bug Fix Plan

Order = recommended landing order on `mac/daily-driver`. Enablers first, then highest-leverage / lowest-risk fixes.

### 5.0 Enablers (must ship first)

#### 0a — `chore/configurable-image-namespace`
- **What:** Replace `image: ghcr.io/purpleailab/decepticon-...` in `docker-compose.yml` with `image: ${DECEPTICON_IMAGE_NAMESPACE:-ghcr.io/purpleailab}/decepticon-...`. Document in `.env.example`.
- **Risk:** trivial.
- **Test:** unit test that parses the rendered compose with the var set and asserts the image refs use the override; existing CI must still pass.
- **Upstream PR:** yes, useful to anyone forking.

#### 0b — `feat/configurable-product-name`
- **What:** Go ldflags `-X main.productName=...` `-X main.defaultHomeDirName=...` so the launcher reports itself as `decepticon-mac` and defaults to `~/.decepticon-mac/`. Touches `clients/launcher/cmd/root.go` and any callers that hard-code "decepticon".
- **Risk:** low.
- **Test:** build with each set of ldflags; assert `--version` output differs; assert default-home resolution differs.
- **Upstream PR:** yes, useful for side-by-side dev/prod stacks.

### 5.1 — `fix/oauth-token-env-var` ⭐ permanent OAuth fix

- **Bug:** Decepticon's LiteLLM `auth/` handler only reads `~/.claude/.credentials.json`. There's no path to use the officially-supported `CLAUDE_CODE_OAUTH_TOKEN` long-lived token, so the refresh-token race with host Claude Code is unavoidable.
- **Fix:** In the LiteLLM custom claude_code handler module:
  1. Check `os.environ.get("CLAUDE_CODE_OAUTH_TOKEN")` first.
  2. If set: synthesize an in-memory credentials dict with `accessToken=<env>`, `refreshToken=""`, `expiresAt=<now + 365 days * 1000>`, `subscriptionType="max"`, `scopes=["user:inference","user:profile"]`. Never attempt to refresh.
  3. Falls back to reading the file when env unset (existing behavior preserved).
- **Risk:** medium — handler internals + auth-path correctness.
- **Tests:**
  - Env-set: handler skips file read, no refresh ever attempted, valid token used.
  - Env-unset: file-read path unchanged (regression guard).
  - Expired-env-token simulation: handler raises a clear error (no silent failure).
- **Upstream PR:** yes, broadly useful for anyone wanting the long-lived-token escape hatch.

### 5.2 — `fix/oauth-mac-keychain`

- **Bug:** On macOS, Claude Code stores credentials in Keychain, not in `~/.claude/.credentials.json`. Users who pick the OAuth method during `onboard` get cryptic 401s with no diagnosis.
- **Fix:** In `clients/launcher/cmd/onboard.go`, when the user selects the Anthropic OAuth method on macOS:
  1. Probe `security find-generic-password -s "Claude Code-credentials"` (metadata only, no secret read → no Keychain dialog).
  2. If item exists AND file is absent: print a clear instruction block recommending **either** `claude setup-token` (which feeds #1's permanent fix path) **or** a one-time export (`security ... -w > ~/.claude/.credentials.json`). Bundle a `decepticon-refresh-oauth` helper script as part of the fork's `bin/` for the export workflow.
  3. If item exists AND file exists: warn about the refresh-race and recommend the env-var path.
- **Risk:** low.
- **Tests:** `@pytest.mark.skipif(sys.platform != "darwin")` mock the `security` CLI; assert each branch's printed guidance.
- **Upstream PR:** yes — macOS guidance benefits all macOS users; the logic is platform-conditional so non-Mac users see no change.

### 5.3 — `fix/blocking-error-default`

- **Bug:** `langgraph dev` raises on sync I/O in async code; Decepticon's `deepagents/middleware/subagents.py` makes a sync `socket.send` → fatal `BlockingError` → run failed.
- **Fix:** Append `--allow-blocking` to the langgraph command in `docker-compose.yml`'s `langgraph` service. Gate via env: `LANGGRAPH_STRICT_ASYNC=0` default (allow), `=1` opts into strict (for developers debugging async issues).
- **Coordination with upstream:** **#295** is open. We comment there with our patch, offer to coordinate. If #295 takes a different direction (e.g., fixes the actual sync I/O call inside deepagents middleware), we drop ours.
- **Risk:** low.
- **Tests:** smoke — launch a langgraph instance with `--allow-blocking`, exercise a subagent dispatch that previously failed, assert it now succeeds.
- **Upstream PR:** coordinate with #295; submit as fall-back PR if #295 stalls.

### 5.4 — `fix/write-file-content-loop`

- **Bug:** LLM omits `content` parameter when calling `write_file` (likely max_tokens truncation on large writes); pydantic rejects with `content: Field required`; LLM retries identical broken call forever. `MentorMiddleware` exists (#248) but doesn't catch this exact pattern.
- **Fix (three-pronged):**
  1. **Tool description tightened** in `decepticon-core/tools/write_file.py` (or equivalent): "ALWAYS include content. For files > 8 KB use the bash tool with a heredoc (`cat > path << 'EOF' ... EOF`) — this avoids tool-call token-budget truncation."
  2. **Error message inline hint**: when pydantic raises `content: Field required`, wrap the error with an additional message: "Hint: this often means the content field was truncated. Either provide a shorter `content` value, or use bash heredoc to write large files."
  3. **`MentorMiddleware` extension**: detect ≥3 consecutive identical-failure tool calls. **"Identical"** = same tool name AND same SHA-256 hash of canonicalized JSON kwargs AND same error class (e.g. `pydantic.ValidationError` with same missing-field set). Inject a forced divergence message into the next model turn: *"You've called <tool> identically 3 times and gotten the same error. Try a different tool or different arguments. For write_file content errors, use the bash heredoc pattern."*
- **Risk:** medium — middleware behavior change requires careful test coverage to avoid false-positive divergence injection.
- **Tests:**
  - Unit: `write_file({"file_path": "/x"})` → error message contains both pydantic detail AND the heredoc hint.
  - Unit: simulate the loop pattern; assert middleware injects divergence message exactly once at the 3rd attempt; assert no injection when calls succeed.
  - Regression: model call with `write_file({"file_path": "/x", "content": "..."})` works unchanged.
- **Upstream PR:** yes — universal LLM-agnostic guard.

### 5.5 — `fix/engagement-soft-delete`

- **Bug:** Deleting an engagement (via web UI delete button) calls `rm -rf workspace/<slug>`. Files (including user-uploaded research) are unrecoverable.
- **Fix:** Introduce a `safe_delete(path: Path)` utility in the web server's filesystem layer:
  - **macOS path:** invoke `osascript -e 'tell application "Finder" to delete POSIX file "<path>"'` — works headlessly, file goes to user's Trash, restorable by macOS conventions.
  - **Linux path:** move to `~/.decepticon-mac/.trash/<slug>-<ISO-timestamp>/` (a 30-day retention zone). A small cron/timer job (or a startup check) prunes entries older than 30 days.
- **Web UI:** "Delete engagement" → confirmation modal updated to say "Move to Trash (recoverable for 30 days)". A new "Trash" view (or a CLI command `decepticon-mac trash list/restore`) lists deleted engagements.
- **Risk:** low.
- **Tests:**
  - macOS path (`@pytest.mark.skipif(sys.platform != "darwin")`): mock `osascript`, assert command shape includes correct POSIX path quoting.
  - Linux path: create tmpdir, call `safe_delete`, assert it moved to `.trash/<slug>-<ts>/`.
  - Retention: create files with mtime > 30 days old in trash; run prune; assert removed.
- **Upstream PR:** yes — everyone benefits from removing the destructive default.

### 5.6 — `fix/sandbox-tmux-zombies`

- **Bug:** `decepticon-sandbox` accumulates `<defunct>` tmux + bash processes (observed 4 zombies during one session).
- **Fix:** In `containers/sandbox-*` / `sandbox_server.py`:
  1. Install a `SIGCHLD` handler at startup that calls `os.waitpid(-1, os.WNOHANG)` in a loop until no more children.
  2. The tmux helper invokes `tmux kill-server` (or `tmux kill-session -t <name>`) on engagement teardown rather than letting tmux servers linger.
- **Risk:** low — sandbox-internal process management.
- **Tests:** integration — spawn 10 short-lived tmux sessions via `/execute`; wait 5s; `ps -eo state` shows zero `Z` (zombie) processes.
- **Upstream PR:** yes — hygiene fix.

### 5.7 — `fix/cli-auto-reconnect`

- **Bug:** CLI's WebSocket stream to LangGraph drops when LangGraph or LiteLLM restarts; user must `Ctrl+C` and relaunch.
- **Fix:** In `clients/cli/src/hooks/useAgent.ts`:
  1. Wrap the streaming connection with an exponential-backoff reconnector: delays `1s, 2s, 4s, 8s, 16s` then cap at 16s; reset on successful message.
  2. On reconnect, query thread state (`GET /threads/<tid>/state`) and resume from latest checkpoint instead of starting a fresh run.
  3. UI: ephemeral inline notice ("Reconnecting…" / "Reconnected") so user sees what's happening.
- **Risk:** medium — touches the streaming machinery; risk of double-firing runs if not careful.
- **Tests:**
  - Unit (Ink hook with React Testing Library + a mock WS): drop the WS mid-stream; assert reconnect with backoff schedule; assert thread state fetched on resume; assert no duplicate run dispatched.
- **Upstream PR:** yes — universal UX win.

### 5.8 — `fix/web-optional-auth` (do last — security-sensitive)

- **Bug:** Web dashboard (`http://localhost:3000`) and terminal WebSocket are fully unauthenticated by design; safe on localhost, critical exposure if ever port-forwarded.
- **Fix:** Default behavior **unchanged**. Add an opt-in env `DECEPTICON_WEB_AUTH`:
  - `none` (default — today's behavior).
  - `password:<bcrypt-hash>` — HTTP basic auth on web routes; WS handshake requires a token derived from the same secret.
  - `disable-bind-public` — server refuses to bind to non-loopback addresses, hard fails at startup if it would.
- **Files touched:** `clients/web/src/lib/auth-bridge.ts`, `clients/web/server/terminal-server.ts`, `SECURITY.md`, `.env.example`.
- **Risk:** **high** (security-sensitive).
- **Gate:** PR description must include a short threat model, list of attack surfaces considered, and explicit "manual security reviewer sign-off" line. Reviewer = fork owner (pol2up) until a co-maintainer exists; sign-off recorded by PR comment `/security-review approved`. Don't squash-merge into `mac/daily-driver` without that.
- **Tests:**
  - Default mode: unauthenticated GET returns 200 (today's behavior).
  - `password:` mode: 401 without creds; 200 with valid creds; WS rejected without token; WS accepted with token.
  - `disable-bind-public`: server bound to `127.0.0.1` succeeds; binding to `0.0.0.0` aborts startup with explicit error.
- **Upstream PR:** yes, but expect careful review — security feature.

## 6. Branch & PR Strategy

### Lifecycle of one fix

```
1. git switch main && git pull upstream main && git push origin main  # keep main aligned
2. git switch -c fix/<name>                                          # off main
3. implement + write tests
4. git push origin fix/<name>
5. gh pr create --base PurpleAILAB:main --head pol2up:fix/<name>     # upstream PR
6. (in parallel — ship locally)
     git switch mac/daily-driver
     git merge --squash fix/<name>
     git commit -m "merge fix/<name> (PR #<num>) into mac/daily-driver"
     make install                                                    # rebuild + reinstall
7. on upstream merge: drop local fix/<name>; rebase mac/daily-driver onto new main (squashed commit drops naturally)
8. on upstream rejection or divergence: keep fix/<name> alive; squashed commit stays in daily-driver
```

### Sync cadence
- `make sync-upstream` weekly (rebases `main` against `upstream/main`, rebases open `fix/*` branches).
- After any new upstream release: rebase `mac/daily-driver` cleanly.

### Conflict handling
- If upstream touches same files: rebase `fix/<name>` onto new `main`, resolve, force-push to the PR. Daily-driver picks up changes on next reintegration.
- If upstream merges a *substitute* fix: close our PR; drop our `fix/<name>`; rebase daily-driver — upstream's version wins.

### CI on our fork
- Enable GitHub Actions on `pol2up/decepticon-mac`; mirror upstream's `ci.yml` + `codeql.yml`.
- Required green CI on every push to `fix/*` before squash-merging into `mac/daily-driver`.
- New workflow `mac-install-smoke.yml` on macOS GitHub-hosted runners runs `make install --dry-run` to catch installer regressions early.

## 7. Testing Approach

**Baseline (non-negotiable):** every `fix/*` branch must pass upstream's existing matrix:
`pytest` · `ruff` · `basedpyright` · `gitleaks` · `hadolint` · `pre-commit` hooks · CodeQL.

**Per-fix new tests:** see each section above for explicit test list.

**Manual end-to-end verification before promoting `mac/daily-driver`:**
1. `make install` from current daily-driver HEAD.
2. `decepticon-mac onboard` → fresh engagement.
3. Both stocks running side-by-side; `docker ps` shows two stacks; no port/container/volume collisions.
4. Send a known-good prompt; assert run completes without errors in `decepticon-mac logs`.
5. Verify the fix that was just landed: each branch carries a manual reproduction recipe in its PR description.

**Security review gate** (only for #8 `fix/web-optional-auth`): PR description must include threat model + manual reviewer sign-off.

## 8. Cutover / Initial Install

### One-time bootstrap (DONE during this session)
```bash
gh repo fork PurpleAILAB/Decepticon --fork-name decepticon-mac --clone=true
cd ~/Projects/decepticon-mac
git remote -v   # origin=pol2up/decepticon-mac, upstream=PurpleAILAB/Decepticon  ✓
git switch -c mac/daily-driver  # ✓
mkdir -p docs/superpowers/specs/
# (this spec was committed here)
```

### Implementation order
1. Land **0a** `chore/configurable-image-namespace` on `fix/*` branch → PR upstream → squash into `mac/daily-driver`.
2. Land **0b** `feat/configurable-product-name`.
3. Add `Makefile` with `launcher`, `images`, `install`, `uninstall`, `dev-shell`, `sync-upstream` targets.
4. First `make install` — installs `decepticon-mac` side-by-side with stock `decepticon`.
5. Verify side-by-side run: both stacks healthy, both web UIs load on different ports.
6. Iterate fixes 5.1 → 5.8 in order, each on its own branch, dual-tracked (upstream PR + daily-driver squash-merge).
7. After each fix lands locally: smoke-test it via Section 7 manual verification.

### First-time configuration
```bash
decepticon-mac onboard      # interactive; seeds ~/.decepticon-mac/.env
decepticon-mac              # launches CLI + web on port 3010
```

### Uninstall (clean rollback)
```bash
make uninstall              # removes binary, symlink, optionally ~/.decepticon-mac
```

## 9. Out of Scope (for this round)

- Publishing fork's images to a public registry (kept local-only).
- A `brew tap` / Homebrew formula for `decepticon-mac` (could come later).
- Windows / Linux distribution parity for the fork (Mac is the explicit target).
- Forking any of upstream's submodules (`benchmark/MHBench`, `benchmark/xbow-validation-benchmarks`) — they're benchmark-only, irrelevant to the bug fixes.
- Backwards-compatibility for installs done before this fork existed (clean install from scratch).

## 10. Risks & Open Questions

| Risk | Likelihood | Mitigation |
|---|---|---|
| Upstream rejects / radically changes one of our PRs | medium | each fix has both an upstream-PR diff AND a local daily-driver squash — local install is unaffected if a PR stalls |
| `CLAUDE_CONFIG_DIR` / Claude Code behavior changes | medium | our permanent OAuth fix uses `CLAUDE_CODE_OAUTH_TOKEN` (officially supported), not `CLAUDE_CONFIG_DIR` (undocumented) |
| LiteLLM's claude_code handler API drifts | low | pin LiteLLM version in fork's `pyproject.toml` until we verify newer versions work |
| Forking workflow drifts as upstream evolves rapidly | medium | `make sync-upstream` runs weekly to catch drift early |
| Disk usage from 2× Docker stacks (~20 GB) | low | acceptable on user's Mac (661 GB free); document teardown path |

---

**Spec end.**
