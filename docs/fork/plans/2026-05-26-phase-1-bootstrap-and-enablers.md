# decepticon-mac Phase 1: Bootstrap & Enabler PRs — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the `decepticon-mac` fork to the point where it installs and runs side-by-side with stock `decepticon` on the user's Mac — *before* any bug fixes ship.

**Architecture:** Two surgical upstream-PR-able changes (configurable image namespace + configurable product name) plus a `Makefile`-based installer that uses those parameterizations to install the fork at a separate prefix, separate stack name, and shifted ports. Each change is on its own `chore/*` or `feat/*` branch; each is squash-merged into `mac/daily-driver` once landed.

**Tech Stack:** Go (launcher build with ldflags), Docker Compose v2, GNU Make, Python 3 (pytest, ruff), Bash (Makefile recipes).

**Companion spec:** `docs/fork/2026-05-26-decepticon-mac-fork-design.md` — sections 1–4, 5.0 (enablers), 6 (PR strategy), 8 (cutover).

---

## File Structure (what this phase creates or modifies)

| Path | Status | Responsibility |
|---|---|---|
| `docker-compose.yml` | modify | Env-overridable image namespace (`DECEPTICON_IMAGE_NAMESPACE`) on every `ghcr.io/purpleailab/decepticon-*` line. |
| `.env.example` | modify | Document `DECEPTICON_IMAGE_NAMESPACE` and the existing port-override env vars. |
| `clients/launcher/cmd/root.go` | modify | Replace hard-coded `"decepticon"` product name with ldflag-injected `productName` + `defaultHomeDirName`. |
| `clients/launcher/cmd/root_test.go` | create | Unit tests for ldflag-driven product name & home defaults. |
| `clients/launcher/cmd/onboard.go` | modify (minimal) | Use the new `defaultHomeDirName` constant where it currently hard-codes `~/.decepticon`. |
| `clients/launcher/cmd/start.go` | modify (minimal) | Same: replace any `~/.decepticon` literals with the resolved default. |
| `tests/test_compose_namespace.py` | create | Python test that renders compose with overrides and asserts image refs change. |
| `Makefile` | create | Targets: `launcher`, `images`, `install`, `uninstall`, `dev-shell`, `sync-upstream`, `clean`. |
| `scripts/seed-env.sh` | create | Idempotent script to write port-shifted defaults into `~/.decepticon-mac/.env` if absent. |
| `scripts/uninstall.sh` | create | Removes binary, symlink, optionally `~/.decepticon-mac/`. |
| `docs/fork/RUNNING.md` | create | Operator doc: how to install, run side-by-side, troubleshoot, uninstall. |

**Branch flow for this phase:**
1. `chore/configurable-image-namespace` → upstream PR → squash into `mac/daily-driver`.
2. `feat/configurable-product-name` → upstream PR → squash into `mac/daily-driver`.
3. `mac/daily-driver` directly (Makefile + scripts + docs — these are fork-only, not upstream-PR'd).
4. `make install` from `mac/daily-driver` HEAD → verify side-by-side.

---

## Task 1: `chore/configurable-image-namespace`

**Branch:** `chore/configurable-image-namespace`, off `main`.

**Files:**
- Modify: `docker-compose.yml` (each `ghcr.io/purpleailab/decepticon-*` image line)
- Modify: `.env.example` (document new env var)
- Create: `tests/test_compose_namespace.py`

- [ ] **Step 1: Create branch off `main`**

```bash
cd ~/Projects/decepticon-mac
git fetch upstream main
git switch main
git reset --hard upstream/main
git push origin main
git switch -c chore/configurable-image-namespace
```

- [ ] **Step 2: Identify all image references in `docker-compose.yml`**

Run:
```bash
grep -nE 'image:\s*ghcr\.io/purpleailab/' docker-compose.yml
```
Expected: 5 lines (litellm, sandbox, langgraph, web, cli — verify the exact count matches what the file shows).

- [ ] **Step 3: Write the failing test**

Create `tests/test_compose_namespace.py`:
```python
"""Verify docker-compose.yml respects DECEPTICON_IMAGE_NAMESPACE override.

Renders the compose file via `docker compose config` with the env var set,
then asserts that every Decepticon image reference uses the override prefix.
"""
from __future__ import annotations

import os
import shutil
import subprocess
from pathlib import Path

import pytest
import yaml

REPO_ROOT = Path(__file__).resolve().parents[1]
COMPOSE = REPO_ROOT / "docker-compose.yml"


def _render(env_overrides: dict[str, str]) -> dict:
    """Run `docker compose config` and parse the result."""
    if shutil.which("docker") is None:
        pytest.skip("docker CLI not available")
    env = {**os.environ, **env_overrides}
    out = subprocess.check_output(
        ["docker", "compose", "-f", str(COMPOSE), "config"],
        env=env,
        cwd=REPO_ROOT,
        text=True,
    )
    return yaml.safe_load(out)


def _decepticon_image_refs(rendered: dict) -> list[str]:
    """Return every `image:` value for a Decepticon-prefixed service."""
    refs: list[str] = []
    for name, svc in (rendered.get("services") or {}).items():
        img = svc.get("image", "")
        if "decepticon-" in img:
            refs.append(img)
    return refs


def test_default_namespace_is_purpleailab():
    rendered = _render({})
    refs = _decepticon_image_refs(rendered)
    assert refs, "expected at least one decepticon image reference"
    for ref in refs:
        assert ref.startswith("ghcr.io/purpleailab/decepticon-"), (
            f"default namespace should be ghcr.io/purpleailab/, got: {ref}"
        )


def test_namespace_override_via_env():
    rendered = _render({"DECEPTICON_IMAGE_NAMESPACE": "ghcr.io/pol2up/decepticon-mac"})
    refs = _decepticon_image_refs(rendered)
    assert refs, "expected at least one decepticon image reference"
    for ref in refs:
        assert ref.startswith("ghcr.io/pol2up/decepticon-mac/decepticon-"), (
            f"override should rewrite namespace, got: {ref}"
        )
```

- [ ] **Step 4: Run the test to verify it fails**

Run:
```bash
cd ~/Projects/decepticon-mac
python -m pytest tests/test_compose_namespace.py -v 2>&1 | tail -20
```
Expected: `test_namespace_override_via_env` FAILS — current compose ignores the env var (image refs still start with `ghcr.io/purpleailab/`).

- [ ] **Step 5: Apply the compose change**

Edit `docker-compose.yml`: every line matching `image: ghcr.io/purpleailab/decepticon-` becomes `image: ${DECEPTICON_IMAGE_NAMESPACE:-ghcr.io/purpleailab}/decepticon-`.

Concretely, find each occurrence and rewrite. For example:
```yaml
# before
    image: ghcr.io/purpleailab/decepticon-langgraph:${DECEPTICON_VERSION:-latest}
# after
    image: ${DECEPTICON_IMAGE_NAMESPACE:-ghcr.io/purpleailab}/decepticon-langgraph:${DECEPTICON_VERSION:-latest}
```

Verify all references changed:
```bash
grep -cE 'image:\s*\$\{DECEPTICON_IMAGE_NAMESPACE' docker-compose.yml
# should match the count from Step 2
grep -cE 'image:\s*ghcr\.io/purpleailab/' docker-compose.yml
# should be 0
```

- [ ] **Step 6: Document the env var in `.env.example`**

Append (or insert under a "Build / image overrides" section if there's a natural spot):
```bash
# --- Image namespace override ---
# Override the GHCR prefix for decepticon-* images. Defaults to the
# upstream-published namespace. Forks should set this to their own GHCR
# (or local registry) prefix so `docker compose pull` resolves to their
# images instead of upstream's.
#   default: ghcr.io/purpleailab
#   fork example: ghcr.io/pol2up/decepticon-mac
# DECEPTICON_IMAGE_NAMESPACE=ghcr.io/purpleailab
```

- [ ] **Step 7: Run the test to verify it passes**

Run:
```bash
python -m pytest tests/test_compose_namespace.py -v 2>&1 | tail -10
```
Expected: both tests PASS.

- [ ] **Step 8: Run the full test suite to confirm no regression**

Run:
```bash
python -m pytest 2>&1 | tail -10
```
Expected: all tests pass (or at least no *new* failures; pre-existing failures noted but not introduced).

- [ ] **Step 9: Run pre-commit hooks locally**

Run:
```bash
pre-commit run --files docker-compose.yml .env.example tests/test_compose_namespace.py 2>&1 | tail -20
```
Expected: all hooks pass (ruff, basedpyright, hadolint, gitleaks, yamllint, trailing whitespace, etc.). If hadolint flags the compose change, address it before continuing.

- [ ] **Step 10: Commit**

```bash
git add docker-compose.yml .env.example tests/test_compose_namespace.py
git -c commit.gpgsign=false commit -m "chore(compose): make image namespace env-overridable

Replaces hard-coded \`ghcr.io/purpleailab/\` prefix on every decepticon-*
image with \`\${DECEPTICON_IMAGE_NAMESPACE:-ghcr.io/purpleailab}/\`, so
forks can point compose at their own registry / local builds without
patching the file.

Default behavior is unchanged (env unset -> ghcr.io/purpleailab).

Test: tests/test_compose_namespace.py verifies both default and
override paths via \`docker compose config\` rendering."
```

- [ ] **Step 11: Push branch and open upstream PR**

```bash
git push -u origin chore/configurable-image-namespace
gh pr create \
  --repo PurpleAILAB/Decepticon \
  --base main \
  --head pol2up:chore/configurable-image-namespace \
  --title "chore(compose): make image namespace env-overridable" \
  --body "## Motivation

Hard-coded \`ghcr.io/purpleailab/\` on every \`decepticon-*\` image line forces forks to patch \`docker-compose.yml\` to point at their own registry or locally-built images. That patch then has to be maintained against every upstream change.

## Change

Replace each \`image: ghcr.io/purpleailab/decepticon-*\` line with \`image: \${DECEPTICON_IMAGE_NAMESPACE:-ghcr.io/purpleailab}/decepticon-*\`. Default behavior unchanged.

## Test

\`tests/test_compose_namespace.py\` renders the compose file via \`docker compose config\` with the env var set / unset and asserts the image refs reflect it."
```

- [ ] **Step 12: Squash-merge into `mac/daily-driver`**

```bash
git switch mac/daily-driver
git merge --squash chore/configurable-image-namespace
git -c commit.gpgsign=false commit -m "merge chore/configurable-image-namespace into mac/daily-driver

Upstream PR: <fill in URL from Step 11 output>"
```

---

## Task 2: `feat/configurable-product-name`

**Branch:** `feat/configurable-product-name`, off `main`.

**Files:**
- Modify: `clients/launcher/cmd/root.go`
- Modify: `clients/launcher/cmd/onboard.go` (literal removal only)
- Modify: `clients/launcher/cmd/start.go` (literal removal only)
- Create: `clients/launcher/cmd/root_test.go`

### Pre-task investigation

- [ ] **Step 1: Identify hard-coded product name & home-dir literals**

Run:
```bash
cd ~/Projects/decepticon-mac/clients/launcher
grep -rnE '"decepticon"|"\.decepticon"' cmd internal 2>&1 | head -30
```
Note the lines for use in Steps 6–7. Expected: at least one in `root.go` (the `cobra.Command{Use: "decepticon", ...}` declaration) and several others using `~/.decepticon`.

- [ ] **Step 2: Switch to a fresh branch**

```bash
cd ~/Projects/decepticon-mac
git switch main
git switch -c feat/configurable-product-name
```

### Test first

- [ ] **Step 3: Write the failing test**

Create `clients/launcher/cmd/root_test.go`:
```go
package cmd

import (
	"strings"
	"testing"
)

func TestProductNameDefaultsToDecepticon(t *testing.T) {
	if ProductName != "decepticon" {
		t.Fatalf("expected ProductName default %q, got %q", "decepticon", ProductName)
	}
}

func TestDefaultHomeDirNameDefaultsToDotDecepticon(t *testing.T) {
	if DefaultHomeDirName != ".decepticon" {
		t.Fatalf("expected DefaultHomeDirName default %q, got %q", ".decepticon", DefaultHomeDirName)
	}
}

func TestRootCommandUseFollowsProductName(t *testing.T) {
	if !strings.HasPrefix(rootCmd.Use, ProductName) {
		t.Fatalf("rootCmd.Use=%q should start with ProductName=%q", rootCmd.Use, ProductName)
	}
}
```

- [ ] **Step 4: Run the test to verify it fails to compile**

Run:
```bash
cd ~/Projects/decepticon-mac/clients/launcher
go test ./cmd/... -run 'TestProductName|TestDefaultHomeDir|TestRootCommandUse' -v 2>&1 | tail -10
```
Expected: BUILD FAILS — `ProductName` and `DefaultHomeDirName` are undefined.

### Implementation

- [ ] **Step 5: Add ldflag-injectable variables in `root.go`**

Edit `clients/launcher/cmd/root.go`. At the top of the file, right after the `package cmd` line and existing imports, add (or extend the existing `var ( ... )` block):

```go
// Build-time-injected metadata. Override with Go ldflags, e.g.:
//   go build -ldflags="-X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.ProductName=decepticon-mac' \
//                      -X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.DefaultHomeDirName=.decepticon-mac'"
//
// Defaults preserve the historical "decepticon" name + "~/.decepticon"
// data dir so unbuilt / unstamped binaries behave identically to upstream.
var (
	ProductName        = "decepticon"
	DefaultHomeDirName = ".decepticon"
)
```

(If `var ( ... )` block already exists with `Version` or similar, add these two inside it instead.)

- [ ] **Step 6: Replace hard-coded `"decepticon"` in `cobra.Command{Use: ...}` declaration**

Find in `root.go`:
```go
var rootCmd = &cobra.Command{
    Use:   "decepticon",
    Short: "...",
    ...
}
```

Change `Use: "decepticon"` → `Use: ProductName`. If `Short` or other strings also contain the literal product name, leave them as-is for now *unless* they would break user-facing help text — in which case use `fmt.Sprintf` with `ProductName`.

- [ ] **Step 7: Replace hard-coded `~/.decepticon` literals via the constant**

In `root.go`, find any path resolution like `filepath.Join(home, ".decepticon")` and rewrite as `filepath.Join(home, DefaultHomeDirName)`.

Repeat for `clients/launcher/cmd/onboard.go` and `clients/launcher/cmd/start.go` — wherever `".decepticon"` appears in a `filepath.Join` call. Do **not** rewrite documentation strings, comments, or env-var names (`DECEPTICON_HOME`) — only path literals.

After edits:
```bash
grep -rnE '"\.decepticon"' clients/launcher/cmd 2>&1
```
Expected: zero hits (only references should be via `DefaultHomeDirName`).

- [ ] **Step 8: Run the tests to verify they pass**

```bash
cd ~/Projects/decepticon-mac/clients/launcher
go test ./cmd/... -run 'TestProductName|TestDefaultHomeDir|TestRootCommandUse' -v 2>&1 | tail -10
```
Expected: all 3 tests PASS.

- [ ] **Step 9: Add a build-time ldflag-injection regression test**

Append to `clients/launcher/cmd/root_test.go`:
```go
import (
	"os/exec"
	"path/filepath"
	"runtime"
	// (keep existing imports: strings, testing)
)

// TestLdflagInjection_End2End builds the launcher with custom ldflags
// and asserts the resulting binary reports the injected ProductName.
func TestLdflagInjection_End2End(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping go-build test in -short mode")
	}
	// Find launcher module root (cmd is one level down from launcher/).
	_, thisFile, _, _ := runtime.Caller(0)
	launcherRoot := filepath.Dir(filepath.Dir(thisFile))
	out := filepath.Join(t.TempDir(), "decepticon-mac-test")
	cmd := exec.Command(
		"go", "build",
		"-ldflags",
		"-X github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.ProductName=decepticon-mac "+
			"-X github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.DefaultHomeDirName=.decepticon-mac",
		"-o", out, ".",
	)
	cmd.Dir = launcherRoot
	if buildOut, err := cmd.CombinedOutput(); err != nil {
		t.Fatalf("ldflag build failed: %v\n%s", err, buildOut)
	}
	versionCmd := exec.Command(out, "--version")
	versionOut, err := versionCmd.CombinedOutput()
	if err != nil {
		t.Fatalf("--version failed: %v\n%s", err, versionOut)
	}
	if !strings.Contains(string(versionOut), "decepticon-mac") {
		t.Fatalf("expected --version to contain %q, got: %s",
			"decepticon-mac", versionOut)
	}
}
```

- [ ] **Step 10: Run the new end-to-end test**

```bash
cd ~/Projects/decepticon-mac/clients/launcher
go test ./cmd/... -run TestLdflagInjection -v 2>&1 | tail -10
```
Expected: PASS — binary built with ldflags reports `decepticon-mac` in `--version`.

- [ ] **Step 11: Run the full launcher test suite**

```bash
cd ~/Projects/decepticon-mac/clients/launcher
go test ./... 2>&1 | tail -10
```
Expected: all existing tests still pass.

- [ ] **Step 12: Run pre-commit on changed Go files**

```bash
cd ~/Projects/decepticon-mac
pre-commit run --files \
  clients/launcher/cmd/root.go \
  clients/launcher/cmd/root_test.go \
  clients/launcher/cmd/onboard.go \
  clients/launcher/cmd/start.go 2>&1 | tail -15
```
Expected: pass. If gofmt / goimports flag formatting, accept those edits.

- [ ] **Step 13: Commit**

```bash
git add clients/launcher/cmd/root.go \
        clients/launcher/cmd/root_test.go \
        clients/launcher/cmd/onboard.go \
        clients/launcher/cmd/start.go
git -c commit.gpgsign=false commit -m "feat(launcher): ldflag-configurable product name and home dir

Introduces two ldflag-injectable variables on \`cmd\` package:
  - ProductName        (default: \"decepticon\")
  - DefaultHomeDirName (default: \".decepticon\")

The cobra root command's Use field and every \"~/.decepticon\" path
literal now route through these constants, so a fork can build the same
binary code under a different name (e.g. \"decepticon-mac\" /
\"~/.decepticon-mac\") just by passing custom ldflags. Default build
behavior is byte-identical to before.

Tests:
  - unit: defaults preserved.
  - end-to-end: build with -X ldflags and assert --version reports the
    injected product name.

Use case: side-by-side dev/prod stacks on the same machine, or forks
that want to coexist with stock installs."
```

- [ ] **Step 14: Push and open upstream PR**

```bash
git push -u origin feat/configurable-product-name
gh pr create \
  --repo PurpleAILAB/Decepticon \
  --base main \
  --head pol2up:feat/configurable-product-name \
  --title "feat(launcher): ldflag-configurable product name and home dir" \
  --body "## Motivation

Running multiple Decepticon stacks on one machine (e.g. a dev fork next to a stock install, or two engagement personas) requires renaming the binary and the data dir. Today both are hard-coded to \`decepticon\` / \`~/.decepticon\` throughout the launcher.

## Change

Two new ldflag-injectable variables in \`clients/launcher/cmd\`:
- \`ProductName\` (default \`decepticon\`)
- \`DefaultHomeDirName\` (default \`.decepticon\`)

The cobra root command's \`Use\` field and every \`~/.decepticon\` path literal now route through these constants. Default build behavior unchanged.

## Test

- Unit test: defaults preserved.
- End-to-end test: builds with custom ldflags and asserts \`--version\` output reflects the injected product name.

## Example use

\`\`\`bash
go build -ldflags=\"\\
  -X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.ProductName=decepticon-mac' \\
  -X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.DefaultHomeDirName=.decepticon-mac'\" \\
  -o decepticon-mac ./clients/launcher
\`\`\`"
```

- [ ] **Step 15: Squash-merge into `mac/daily-driver`**

```bash
git switch mac/daily-driver
git merge --squash feat/configurable-product-name
git -c commit.gpgsign=false commit -m "merge feat/configurable-product-name into mac/daily-driver

Upstream PR: <fill in URL from Step 14 output>"
```

---

## Task 3: `Makefile` + install scripts (fork-only, not upstream-PR'd)

**Branch:** `mac/daily-driver` (commit directly — these files are fork-specific).

**Files:**
- Create: `Makefile`
- Create: `scripts/seed-env.sh`
- Create: `scripts/uninstall.sh`

### Define paths up front

The Makefile uses these constants throughout — keep them consistent:

```
FORK_NAME            = decepticon-mac
HOME_DIR_NAME        = .decepticon-mac
BIN_DIR              = $(HOME)/.local/bin
SYMLINK_DIR          = /usr/local/bin
DATA_DIR             = $(HOME)/$(HOME_DIR_NAME)
IMAGE_NAMESPACE      = ghcr.io/pol2up/decepticon-mac
IMAGE_VERSION        = dev
STACK_NAME           = mac
LAUNCHER_PKG         = ./clients/launcher
LDFLAGS              = -X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.ProductName=$(FORK_NAME)' \
                       -X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.DefaultHomeDirName=$(HOME_DIR_NAME)'
```

### Steps

- [ ] **Step 1: Confirm we're on `mac/daily-driver`**

```bash
cd ~/Projects/decepticon-mac
git switch mac/daily-driver
git branch --show-current
```
Expected: `mac/daily-driver`.

- [ ] **Step 2: Create `Makefile`**

Create `~/Projects/decepticon-mac/Makefile`:
```makefile
# decepticon-mac fork — Makefile
#
# Targets:
#   launcher        Build the Go launcher binary into ~/.local/bin/decepticon-mac
#   images          Build all decepticon-mac-* Docker images locally
#   install         launcher + images + symlink + .env seed
#   uninstall       Remove binary, symlink, optionally ~/.decepticon-mac
#   dev-shell       Subshell with PATH rewritten so decepticon-mac resolves from this checkout
#   sync-upstream   Fetch upstream/main + rebase main + report fix/* divergence
#   clean           Remove built binary (does NOT touch installed copy)

FORK_NAME       := decepticon-mac
HOME_DIR_NAME   := .decepticon-mac
BIN_DIR         := $(HOME)/.local/bin
SYMLINK_DIR     := /usr/local/bin
DATA_DIR        := $(HOME)/$(HOME_DIR_NAME)
IMAGE_NAMESPACE := ghcr.io/pol2up/decepticon-mac
IMAGE_VERSION   := dev
STACK_NAME      := mac
LAUNCHER_PKG    := ./clients/launcher
LAUNCHER_BIN    := $(BIN_DIR)/$(FORK_NAME)
SYMLINK_PATH    := $(SYMLINK_DIR)/$(FORK_NAME)
LDFLAGS         := -X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.ProductName=$(FORK_NAME)' \
                   -X 'github.com/PurpleAILAB/Decepticon/clients/launcher/cmd.DefaultHomeDirName=$(HOME_DIR_NAME)'

# Decepticon images we need to build (matches docker-compose.yml services).
IMAGES := decepticon-langgraph decepticon-litellm decepticon-sandbox decepticon-cli decepticon-web

.PHONY: help launcher images install uninstall dev-shell sync-upstream clean

help:
	@awk 'BEGIN {FS=":.*##"} /^[a-zA-Z_-]+:.*##/ {printf "  %-20s %s\n", $$1, $$2}' $(MAKEFILE_LIST)

launcher: ## Build $(FORK_NAME) binary into ~/.local/bin
	@mkdir -p $(BIN_DIR)
	@echo ">> building $(FORK_NAME) -> $(LAUNCHER_BIN)"
	cd $(LAUNCHER_PKG) && go build -ldflags "$(LDFLAGS)" -o $(LAUNCHER_BIN) .
	@echo "   built: $$($(LAUNCHER_BIN) --version 2>&1 | head -1)"

images: ## Build decepticon-mac-* images locally
	@echo ">> building $(words $(IMAGES)) images under $(IMAGE_NAMESPACE):$(IMAGE_VERSION)"
	@for svc in $(IMAGES); do \
	    dockerfile="containers/$${svc#decepticon-}.Dockerfile"; \
	    if [ ! -f "$$dockerfile" ]; then echo "   skip $$svc (no $$dockerfile)"; continue; fi; \
	    echo "   $$svc -> $(IMAGE_NAMESPACE)/$$svc:$(IMAGE_VERSION)"; \
	    docker build -f "$$dockerfile" -t "$(IMAGE_NAMESPACE)/$$svc:$(IMAGE_VERSION)" \
	        --build-arg VERSION=$(IMAGE_VERSION) .; \
	done

install: launcher images ## Full install: binary + images + symlink + .env seed
	@echo ">> creating symlink $(SYMLINK_PATH)"
	@if [ ! -L "$(SYMLINK_PATH)" ] || [ "$$(readlink $(SYMLINK_PATH))" != "$(LAUNCHER_BIN)" ]; then \
	    sudo ln -sf $(LAUNCHER_BIN) $(SYMLINK_PATH); \
	    echo "   symlinked: $(SYMLINK_PATH) -> $(LAUNCHER_BIN)"; \
	else \
	    echo "   symlink already correct"; \
	fi
	@mkdir -p $(DATA_DIR)
	@bash scripts/seed-env.sh "$(DATA_DIR)"
	@echo "   done. Run \`$(FORK_NAME) onboard\` to configure."

uninstall: ## Remove binary, symlink, optionally ~/.$(HOME_DIR_NAME)
	@bash scripts/uninstall.sh "$(LAUNCHER_BIN)" "$(SYMLINK_PATH)" "$(DATA_DIR)"

dev-shell: ## Open subshell where $(FORK_NAME) resolves from this checkout
	@echo ">> entering dev-shell. PATH prepended with $(BIN_DIR)."
	@PATH="$(BIN_DIR):$$PATH" $$SHELL

sync-upstream: ## Fetch upstream/main and rebase mac/daily-driver
	@echo ">> fetching upstream/main"
	git fetch upstream main
	@echo ">> updating local main"
	@CURR=$$(git branch --show-current); \
	    git switch main && \
	    git reset --hard upstream/main && \
	    git push origin main && \
	    git switch $$CURR
	@echo ">> rebasing mac/daily-driver onto main"
	git switch mac/daily-driver && git rebase main || \
	    { echo "REBASE FAILED — resolve, then \`git rebase --continue\`"; exit 1; }
	@echo "   sync complete. Run \`make install\` to rebuild."

clean: ## Remove built binary from ~/.local/bin (does NOT remove $(DATA_DIR))
	@rm -f $(LAUNCHER_BIN)
	@echo ">> removed $(LAUNCHER_BIN)"
```

- [ ] **Step 3: Create `scripts/seed-env.sh`**

Create `~/Projects/decepticon-mac/scripts/seed-env.sh`:
```bash
#!/usr/bin/env bash
# Seed ~/.decepticon-mac/.env with port-shifted defaults if absent.
# Idempotent: if .env already exists, leaves it alone.
set -euo pipefail

DATA_DIR="${1:?usage: seed-env.sh <data-dir>}"
ENV_FILE="$DATA_DIR/.env"

if [[ -f "$ENV_FILE" ]]; then
    echo "   .env already present at $ENV_FILE — leaving untouched"
    exit 0
fi

mkdir -p "$DATA_DIR"
cat > "$ENV_FILE" <<'EOF'
# decepticon-mac fork — seeded by `make install` on first run.
# Edit freely; `make install` will not overwrite an existing .env.

# --- Stack identity (do NOT change unless you know what you're doing) ---
DECEPTICON_STACK_NAME=mac
DECEPTICON_HOME=__DATA_DIR__
DECEPTICON_IMAGE_NAMESPACE=ghcr.io/pol2up/decepticon-mac
DECEPTICON_VERSION=dev

# --- Port-shifted so we coexist with stock decepticon (defaults in parens) ---
WEB_PORT=3010              # stock: 3000
TERMINAL_PORT=3013         # stock: 3003
LANGGRAPH_PORT=2034        # stock: 2024
LITELLM_PORT=4010          # stock: 4000
NEO4J_HTTP_PORT=7484       # stock: 7474
NEO4J_BOLT_PORT=7697       # stock: 7687
POSTGRES_PORT=5442         # stock: 5432

# --- Auth (run `decepticon-mac onboard` to populate properly) ---
# ANTHROPIC_API_KEY=
# DECEPTICON_AUTH_PRIORITY=anthropic_api
# DECEPTICON_AUTH_CLAUDE_CODE=false
EOF
# substitute the actual data dir into the placeholder
sed -i.bak "s|__DATA_DIR__|$DATA_DIR|g" "$ENV_FILE"
rm -f "${ENV_FILE}.bak"
chmod 600 "$ENV_FILE"
echo "   seeded $ENV_FILE (port-shifted defaults)"
```

Make it executable:
```bash
chmod +x scripts/seed-env.sh
```

- [ ] **Step 4: Create `scripts/uninstall.sh`**

Create `~/Projects/decepticon-mac/scripts/uninstall.sh`:
```bash
#!/usr/bin/env bash
# Uninstall decepticon-mac. Removes binary and symlink unconditionally.
# Prompts before removing the data dir (which may hold user configs / workspace).
set -euo pipefail

BIN="${1:?usage: uninstall.sh <binary> <symlink> <data-dir>}"
SYMLINK="${2:?usage: uninstall.sh <binary> <symlink> <data-dir>}"
DATA_DIR="${3:?usage: uninstall.sh <binary> <symlink> <data-dir>}"

if [[ -f "$BIN" ]]; then
    rm -f "$BIN"
    echo "   removed $BIN"
fi
if [[ -L "$SYMLINK" ]]; then
    sudo rm -f "$SYMLINK"
    echo "   removed $SYMLINK"
fi
if [[ -d "$DATA_DIR" ]]; then
    read -p "   remove $DATA_DIR (configs + workspace)? [y/N] " ans
    case "$ans" in
        y|Y|yes|YES)
            rm -rf "$DATA_DIR"
            echo "   removed $DATA_DIR" ;;
        *)
            echo "   kept $DATA_DIR" ;;
    esac
fi
echo "   uninstall complete"
```

```bash
chmod +x scripts/uninstall.sh
```

- [ ] **Step 5: Sanity-check `make help`**

```bash
cd ~/Projects/decepticon-mac
make help
```
Expected: list of targets with descriptions, one per line. If output is empty or garbled, fix the `help:` target.

- [ ] **Step 6: Commit Makefile + scripts**

```bash
git add Makefile scripts/seed-env.sh scripts/uninstall.sh
git -c commit.gpgsign=false commit -m "build(fork): Makefile and install scripts for decepticon-mac

Adds a Makefile with launcher / images / install / uninstall / dev-shell
/ sync-upstream / clean targets, plus two helper scripts:
  - scripts/seed-env.sh: idempotent first-run .env seed with port-shifted
    defaults (3010 web, 3013 terminal, 2034 langgraph, 4010 litellm,
    7484/7697 neo4j, 5442 postgres).
  - scripts/uninstall.sh: removes binary + symlink unconditionally,
    prompts before removing ~/.decepticon-mac.

These are fork-only artefacts (not upstream-PR'd). Relies on:
  - chore/configurable-image-namespace (compose env override)
  - feat/configurable-product-name (Go ldflags)
both of which are squash-merged on mac/daily-driver before this commit."
```

---

## Task 4: First `make install` (the moment-of-truth)

**Files:** none created/modified — this task runs the install and verifies.

- [ ] **Step 1: Ensure Docker is running**

```bash
docker info >/dev/null 2>&1 && echo "docker ✓" || { echo "docker daemon down — start Docker Desktop"; exit 1; }
```
Expected: `docker ✓`. If not, launch Docker Desktop manually and retry.

- [ ] **Step 2: Run `make launcher` alone first (cheapest target)**

```bash
cd ~/Projects/decepticon-mac
make launcher 2>&1 | tail -10
```
Expected: `~/.local/bin/decepticon-mac` exists; the printed `--version` line includes `decepticon-mac`.

- [ ] **Step 3: Sanity-check the built binary**

```bash
~/.local/bin/decepticon-mac --version
~/.local/bin/decepticon-mac --help | head -20
```
Expected: version output and help text both say `decepticon-mac`, never `decepticon`. Available subcommands list matches upstream (`onboard`, `start`, `stop`, `status`, etc.).

- [ ] **Step 4: Run `make images` (slow — multiple GB to build)**

```bash
make images 2>&1 | tail -30
```
Expected: each `decepticon-*` image is built locally and tagged `ghcr.io/pol2up/decepticon-mac/decepticon-*:dev`. Watch for build errors; the most likely failure is missing build deps in the Dockerfiles — copy + fix if so, but most should "just work" since they're upstream-provided.

If a Dockerfile fails: stop the loop, fix, re-run only the failed image with `docker build -f containers/<name>.Dockerfile -t ghcr.io/pol2up/decepticon-mac/decepticon-<name>:dev .`.

- [ ] **Step 5: Verify all images exist**

```bash
docker images | grep "pol2up/decepticon-mac"
```
Expected: 5 lines (langgraph, litellm, sandbox, cli, web) — or however many `containers/*.Dockerfile` files exist.

- [ ] **Step 6: Run `make install` (symlink + env seed)**

```bash
make install 2>&1 | tail -15
```
Expected:
- prompts for `sudo` once (for the symlink).
- `/usr/local/bin/decepticon-mac` symlink created.
- `~/.decepticon-mac/.env` seeded with the port-shifted defaults.

- [ ] **Step 7: Verify install is fully wired**

```bash
ls -l /usr/local/bin/decepticon-mac
which decepticon-mac
decepticon-mac --version
ls -la ~/.decepticon-mac/
cat ~/.decepticon-mac/.env | head -20
```
Expected (in order):
- symlink → `~/.local/bin/decepticon-mac`
- `decepticon-mac` resolves to `/usr/local/bin/decepticon-mac`
- `--version` reports `decepticon-mac`
- data dir exists, contains `.env`
- `.env` content matches the seed template, with `DECEPTICON_HOME=/Users/polreqs/.decepticon-mac` (substituted).

---

## Task 5: Side-by-side verification

**Files:** none created/modified.

- [ ] **Step 1: Confirm stock decepticon is still installed & runnable**

```bash
which decepticon
decepticon --version
ls -d ~/.decepticon
```
Expected: stock install untouched; `decepticon --version` reports the upstream version (1.1.2 currently).

- [ ] **Step 2: Stop stock decepticon (if running) so we can start the fork cleanly first**

```bash
decepticon stop 2>&1 | tail -5
docker ps --format '{{.Names}}' | grep '^decepticon-' || echo "(no stock containers)"
```
Expected: no `decepticon-*` containers running (the fork's containers will be named `decepticon-mac-*` so they wouldn't be matched by `^decepticon-` followed by a non-`mac` segment, but stop both to be safe).

- [ ] **Step 3: Run `decepticon-mac onboard`** (interactive — user runs this themselves)

In the user's terminal:
```bash
decepticon-mac onboard
```
Expected: same wizard as stock, but writes to `~/.decepticon-mac/.env`. User picks auth method, model profile, etc.

- [ ] **Step 4: Launch the fork's stack**

```bash
decepticon-mac start 2>&1 | tail -15
# or just `decepticon-mac` if start is the default
```
Wait for it to report all services healthy.

- [ ] **Step 5: Verify fork's containers exist with the expected names**

```bash
docker ps --format '{{.Names}}\t{{.Status}}' | grep '^decepticon-mac-'
```
Expected: 7 containers (litellm, langgraph, sandbox, web, cli, postgres, neo4j), all `Up`/`healthy`. None should be named bare `decepticon-*`.

- [ ] **Step 6: Verify the fork's web UI loads on port 3010**

```bash
curl -fsS http://localhost:3010/ -o /dev/null -w "fork web: HTTP %{http_code}\n"
```
Expected: `fork web: HTTP 200`.

- [ ] **Step 7: Now ALSO start stock decepticon — true side-by-side test**

In a second terminal:
```bash
decepticon start 2>&1 | tail -15
```
Wait for stock to come up.

- [ ] **Step 8: Both stacks running simultaneously**

```bash
docker ps --format '{{.Names}}\t{{.Status}}'
```
Expected: both `decepticon-*` (stock, 7 containers) AND `decepticon-mac-*` (fork, 7 containers) all `Up`/`healthy`. Total 14 Decepticon containers (plus your own dev containers).

- [ ] **Step 9: Both web UIs respond independently**

```bash
curl -fsS http://localhost:3000/ -o /dev/null -w "stock web: HTTP %{http_code}\n"
curl -fsS http://localhost:3010/ -o /dev/null -w "fork  web: HTTP %{http_code}\n"
```
Expected: both `HTTP 200`.

- [ ] **Step 10: Both networks present, no collisions**

```bash
docker network ls | grep -E "decepticon|decepticon-mac"
```
Expected: separate networks for stock (`decepticon_decepticon-net`, `decepticon_sandbox-net`) and fork (`decepticon-mac_decepticon-net`, `decepticon-mac_sandbox-net`).

- [ ] **Step 11: Stop both stacks cleanly**

```bash
decepticon stop
decepticon-mac stop
docker ps | grep -E "decepticon" || echo "(all stopped)"
```
Expected: `(all stopped)`.

---

## Task 6: Operator docs

**Files:** create `docs/fork/RUNNING.md`.

- [ ] **Step 1: Write the running guide**

Create `~/Projects/decepticon-mac/docs/fork/RUNNING.md`:
```markdown
# Running `decepticon-mac` Side-by-Side with stock `decepticon`

This fork installs as a parallel app; the original Decepticon stays usable.

## One-time install

```bash
git clone https://github.com/pol2up/decepticon-mac.git ~/Projects/decepticon-mac
cd ~/Projects/decepticon-mac
git switch mac/daily-driver
make install      # prompts for sudo once (for the /usr/local/bin/decepticon-mac symlink)
decepticon-mac onboard
```

## Daily use

```bash
decepticon-mac           # launches CLI + web on port 3010
```

## Reinstall after pulling new fixes

```bash
cd ~/Projects/decepticon-mac
git switch mac/daily-driver
git pull --rebase
make install             # rebuilds binary + images; preserves your .env
```

## Sync from upstream

```bash
make sync-upstream       # fetch upstream/main, rebase main, rebase mac/daily-driver
make install             # rebuild against the synced HEAD
```

## Side-by-side endpoints

|        | Stock     | Fork      |
|--------|-----------|-----------|
| Web    | :3000     | :3010     |
| WS     | :3003     | :3013     |
| LangGraph API | :2024 | :2034 |
| LiteLLM | :4000    | :4010     |
| Neo4j HTTP | :7474 | :7484    |
| Neo4j Bolt | :7687 | :7697    |
| Postgres | :5432  | :5442    |

You can run both at the same time. Container names are prefixed
`decepticon-` (stock) vs `decepticon-mac-` (fork). Networks and volumes
are independent.

## Troubleshooting

**`make install` fails on `sudo ln -sf`:** if you can't `sudo`, install
the binary only at `~/.local/bin/decepticon-mac` and add that to your
PATH. Skip the `/usr/local/bin` symlink.

**Both stacks try to use the same Claude OAuth file:** that's expected.
The race-free path is to set `CLAUDE_CODE_OAUTH_TOKEN` (a long-lived
token from `claude setup-token`) in `~/.decepticon-mac/.env` — see fix
#1 in the spec.

**Disk pressure:** two stacks pull ~20 GB of images. `docker system prune`
won't reclaim our `:dev`-tagged images; run `make clean` or
`docker rmi $(docker images -q ghcr.io/pol2up/decepticon-mac/*)` to
remove the fork's images. `make uninstall` does the launcher side.
```

- [ ] **Step 2: Commit**

```bash
cd ~/Projects/decepticon-mac
git add docs/fork/RUNNING.md
git -c commit.gpgsign=false commit -m "docs(fork): operator guide for side-by-side use

Covers one-time install, daily use, reinstall after fixes, upstream
sync, port mapping, and the common troubleshooting cases."
```

---

## Self-Review Checklist (run after writing the plan)

- [x] **Spec coverage:** every requirement from the spec sections 1–4, 5.0 (enablers), 6 (PR strategy), 8 (cutover) is addressed by a task above.
- [x] **No placeholders:** every step has an exact path, complete code, exact command, and expected output.
- [x] **Type consistency:** `ProductName`/`DefaultHomeDirName` referenced consistently across Task 2 and the Makefile; image name format `ghcr.io/pol2up/decepticon-mac/decepticon-<svc>:dev` consistent in the Makefile, scripts, and Task 4 verification commands.

---

## Definition of Done for Phase 1

1. `chore/configurable-image-namespace` PR opened upstream; commit squash-merged on `mac/daily-driver`.
2. `feat/configurable-product-name` PR opened upstream; commit squash-merged on `mac/daily-driver`.
3. `Makefile` + scripts committed on `mac/daily-driver`.
4. `decepticon-mac` binary installed at `/usr/local/bin/decepticon-mac` → `~/.local/bin/decepticon-mac`.
5. All 5 fork images built locally and tagged `ghcr.io/pol2up/decepticon-mac/decepticon-*:dev`.
6. `~/.decepticon-mac/.env` seeded with port-shifted defaults.
7. Both stacks (stock + fork) run simultaneously; both web UIs respond on their respective ports.
8. `docs/fork/RUNNING.md` committed.

Once these are all true, Phase 1 is complete and we move to Plan 2: `fix/oauth-token-env-var` (the permanent OAuth fix).
