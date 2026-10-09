# Walkthrough

Chronological log of significant changes. Oldest → newest. Format per entry:
`## YYYY-MM-DD (short title)` followed by `Findings`, `Conclusions`, `Actions`.

## 2026-10-07 (Fix CLI forget/purge command tree)

- Findings: `cmd/cli/main.go` declared the top-level `forget` command twice verbatim (a merge artifact), and the `purge` command carried a stray `forget` child — a soft-delete nested under a hard-delete command, which made `stash purge forget` nonsensical. Separately, `purgeFactCmd` in `cmd/cli/commands.go` was defined but never registered, so `stash purge fact` did not exist even though its handler was written. This surfaced while onboarding a new contributor and inspecting the command tree.
- Conclusions: The fix is purely structural and the intended shape is unambiguous: a single top-level `forget`, and a `purge` command with two children (`episode` and `fact`). No design decision was required, so it was done as a self-contained first-PR-sized change rather than folded into anything larger. The migration numbering gap (no `00003`) was deliberately left alone — goose does not care and renumbering migrations is riskier than it looks.
- Actions: Edited `cmd/cli/main.go` — removed the duplicate `forget` block, removed the stray `forget` nested under `purge`, and added a `fact` child (`Action: purgeFactCmd`) beside `episode`. Committed as `52abad5` on `main` (fork `Bojanni050/stash`). Validated with `git diff --check` (clean) and manual indentation review; `go build` / `go vet` / `gofmt` were NOT run because no Go toolchain is installed in the environment.

## 2026-10-07 (Add CI workflow)

- Findings: `.github/workflows/` contained only `release.yml`, which runs on published releases. Pull requests were therefore never compiled or tested, so a structurally broken CLI command tree (the duplicate `forget` / unreachable `purge fact` issue above) could merge unnoticed.
- Conclusions: Added a minimal CI workflow running `go vet`, `go build` and `go test` on pushes to `main` and on pull requests. A `gofmt`/formatting gate was deliberately left out, because the tree is not guaranteed to be gofmt-clean (inconsistent indentation existed in `cmd/cli/main.go`), which would make CI fail on introduction; a dedicated formatting pass is better done as its own change.
- Actions: Added `.github/workflows/ci.yml`. Not validated locally (no Go toolchain; Docker daemon not running). Caveat: GitHub does not run workflows on forked repositories until Actions is explicitly enabled in the fork's settings.

## 2026-10-07 (Bump GitHub Actions to current majors)

- Findings: The first CI run emitted annotations that `actions/checkout@v4` and `actions/setup-go@v5` target the deprecated Node.js 20 (forced onto Node 24). `release.yml` also pinned older majors (`checkout@v4`, `setup-go@v5`, `softprops/action-gh-release@v1`, and `docker/*` at v3–v5).
- Conclusions: Bumped every action in both workflows to its current major, after verifying via the GitHub API that each major tag exists. Moving major tags (e.g. `@v7`) were chosen over pinned patch versions to match the existing convention and keep future updates simple.
- Actions: `.github/workflows/ci.yml` — checkout v4→v7, setup-go v5→v7. `.github/workflows/release.yml` — checkout v4→v7, setup-go v5→v7, action-gh-release v1→v3, setup-qemu v3→v4, setup-buildx v3→v4, login v3→v4, metadata v5→v6, build-push v5→v7. The `ci.yml` bumps are validated by the CI run triggered on this push; `release.yml` only runs on a published release, so its bumps are not exercised until then.

## 2026-10-07 (Apply gofmt and add a formatting gate)

- Findings: `gofmt -l .` flagged 12 files as not gofmt-clean, including `cmd/cli/main.go`, several `internal/brain/*` files and the new `internal/webui/webui.go`. This confirmed the tree was not consistently formatted — the reason the formatter gate was deliberately left out of the initial CI workflow.
- Conclusions: Ran `gofmt -w .` across the repo and added a blocking `gofmt -l` check to `ci.yml`, so formatting is now enforced going forward. Kept as a separate branch/PR because the diff is purely mechanical and is easier to review in isolation from functional changes.
- Actions: Formatted 12 files (indentation/alignment only, no logic changes — verified with `go build ./...`, `go vet ./...` and `go test ./...`, all passing on Go 1.27.1). Added a "Check formatting (gofmt)" step to `.github/workflows/ci.yml`.

## 2026-10-09 (Fix docker compose so the dashboard is reachable)

- Findings: README and `docs/GETTING_STARTED.md` promise the web dashboard at `http://localhost:9090/ui` after `docker compose up`, but `docker-compose.yml` ran `mcp serve --with-consolidation` — which starts only the MCP SSE server, not the HTTP/dashboard server — and published only port 8080. So the documented dashboard URL did not work out of the box.
- Conclusions: Fixed the compose rather than the docs, because the dashboard is a real feature and the docs describe the intended first-run experience. Switched the service command to `serve` (starts HTTP + MCP + consolidation together) and published port 9090. Also removed the obsolete top-level `version` key that Docker Compose v2 warns about. On the live host this is fronted by `tailscale serve` (8444 → MCP, 8445 → dashboard), which is environment-specific and stays out of the repo.
- Actions: `docker-compose.yml` — command `mcp serve …` → `serve --http-host 0.0.0.0 --http-port 9090 --mcp-host 0.0.0.0 --mcp-port 8080`; added the `"9090:9090"` port mapping; removed `version: '3.8'`. Validated with `docker compose config` (parses clean, no version warning). No Go changes, so no build was required.
