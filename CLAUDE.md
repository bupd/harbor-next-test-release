# Harbor Next - Claude Code Instructions

## Project Overview

Harbor Next is an enhanced fork of [goharbor/harbor](https://github.com/goharbor/harbor) - a cloud-native container registry. It adds patches, improvements, and CI/CD automation on top of the upstream Harbor project.

- **Go backend** in `src/` (multi-module, Go 1.25+)
- **Angular frontend** in `src/portal/` (Angular 16, built with Bun)
- **Build automation** via Taskfile (see `Taskfile.yml` and `taskfile/`)

## Key Commands

```bash
task build            # Build all Go binaries (linux/amd64 locally, multi-arch in CI)
task test:quick       # API spec lint + unit tests (fast)
task test:ci          # Full CI pipeline with reports
task test:unit        # Go unit tests with race detection
task test:lint        # golangci-lint via Docker
task test:lint:api    # Swagger spec lint via Spectral Docker
task images           # Build and push all Docker images
task dev:up           # Start full dev environment with hot reload
task info             # Print version and build info
```

## Contribution Workflow

All changes go through PRs - never push directly to `main`.

```
git checkout -b feat/my-feature
# ... make changes with conventional commits (git commit -s) ...
git push origin feat/my-feature
gh pr create
```

PR title must follow Conventional Commits with lowercase type prefix and capitalized subject: `feat: Add New Feature`, `fix: Resolve Issue`, `docs: Update README`, etc.
All commits require DCO sign-off: `git commit -s`.

**Merging PRs:** Always use **Squash and merge**. Never "Create a merge commit" or "Rebase and merge". Non-squash merges create `Merge pull request #N` commits that break release-please's commit parser.

## Release Process

Releases are automated via [release-please](https://github.com/googleapis/release-please). Configuration lives in `release-please-config.json` (settings, changelog sections, exclude-paths) and `.release-please-manifest.json` (current version).

### Flow

1. A `feat:` or `fix:` PR is squash-merged to `main`
2. Release-please scans commits since the last release
3. It opens a `chore: release X.Y.Z` PR that updates `VERSION` and `CHANGELOG.md`
4. Maintainer reviews and merges the release PR (squash merge)
5. GitHub Release is created with tag `vX.Y.Z` (tags include `v` prefix)
6. The release pipeline runs: build -> merge manifests -> sign -> update release notes

### Release Pipeline Jobs (`release-please.yml`)

When a release is created, four jobs run in sequence:

1. **build** - Builds each component per platform in a matrix (8 components x 2 platforms). Go components are cross-compiled natively (not inside Docker) for speed, then a Docker image is built and pushed by digest.
2. **merge** - Downloads per-platform digests for each component and creates a multi-arch manifest list, tagged as `<registry>/harbor-<component>:<version>`.
3. **sign** - Signs all 8 images with [cosign](https://github.com/sigstore/cosign) using keyless OIDC (GitHub Actions identity).
4. **release notes** - Enriches the GitHub Release body with:
   - `## Highlights` extracted from merged PRs that have a `## Release Notes` section in their description
   - `## Enterprise Features` if enterprise patches were applied
   - `## Container Images` table with all image references
   - Cosign verification command

### Components

The 8 Docker images built per release: `core`, `jobservice`, `registryctl`, `exporter`, `portal`, `registry`, `trivy-adapter`, `nginx`.

Go components (core, jobservice, registryctl, exporter, registry, trivy-adapter) are cross-compiled before the Docker build. Portal and nginx are built entirely inside Docker.

### Enterprise Patches (Optional)

If `PATCHES_REPO` is set as a GitHub variable, the build job clones the patches repo and applies patches from a `series` file before building. Patch descriptions are collected and added to release notes. Requires `PATCHES_TOKEN` secret.

### Version Bump Rules

- `fix:` -> patch (2.19.0 -> 2.19.1)
- `feat:` -> minor (2.19.0 -> 2.20.0)
- `feat!:` or `BREAKING CHANGE:` footer -> major

### Changelog Sections

Visible: `feat:`, `fix:`, `perf:`, `revert:`, `refactor:`, `docs:`.
Hidden (not in changelog): `ci:`, `build:`, `chore:`, `test:`.

### Exclude Paths

Commits that only touch `.github/`, `docs/`, or `tests/` do NOT trigger a version bump even with `feat:` or `fix:`. Use `ci:` for CI-only changes.

## File Structure

```
Taskfile.yml                    # Root task runner (includes taskfile/*.yml)
taskfile/
  build.yml                     # Go binary compilation, swagger codegen
  image.yml                     # Docker multi-arch image builds
  test.yml                      # Linting, unit tests, vulnerability scanning
  dev.yml                       # Local dev environment (docker-compose)
versions.env                    # Pinned versions for all tools and base images
VERSION                         # Current release version (managed by release-please)
CHANGELOG.md                    # Auto-generated changelog (managed by release-please)
CONTRIBUTING.md                 # Contribution guide (PR workflow, merge rules, release notes)
release-please-config.json      # Release-please settings, changelog sections, exclude-paths
.release-please-manifest.json   # Current version: { ".": "X.Y.Z" }
lefthook.yml                    # Local git hooks (conventional commits, DCO, spellcheck)
src/                            # Go backend source
  go.mod
  core/                         # Core registry service
  jobservice/                   # Background job service
  registryctl/                  # Registry controller
  cmd/exporter/                 # Prometheus exporter
  portal/                       # Angular frontend
    package.json
    bun.lock
api/v2.0/swagger.yaml           # Harbor REST API spec
dockerfile/                     # Dockerfiles for each component
devenv/                         # Docker Compose for local development
.github/
  workflows/                    # CI/CD pipelines
  labeler.yml                   # Label rules for PR auto-labeling
```

## GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `test.yml` | PRs + push to main | `task test:quick` - unit tests (PostgreSQL + Redis) and API lint |
| `build.yml` | PRs + push to main | `task build` - compile all Go binaries (linux/amd64) |
| `release-please.yml` | Push to main | Release PR automation; on release: multi-arch image build, sign, publish |
| `pr-title.yml` | PR opened/edited | Enforce conventional commit format on PR titles |
| `lint-actions.yml` | PRs + push to main (workflow files only) | Verify all GitHub Actions are pinned to full commit SHAs |
| `labeler.yml` | PR opened | Auto-label PRs by component (core, portal, jobservice, api, ci, docs, deps, tests) |
| `dependency-review.yml` | PRs to main | Block high-severity CVEs in dependency changes |
| `spellcheck.yml` | PRs + push to main (docs/configs only) | Typo detection via typos-cli |
| `scorecard.yml` | Weekly + push to main | OpenSSF Scorecard security analysis |
| `welcome.yml` | First issue/PR | Welcome message for new contributors |

Test and build run as separate parallel workflows for faster CI feedback. Both map 1:1 to local taskfile commands.

## Local Git Hooks (lefthook)

Install: `lefthook install` (requires [lefthook](https://github.com/evilmartians/lefthook))

Hooks enforce:
- Spell check on staged `.md`/`.yml` files (`pre-commit`)
- Conventional commit message format (`commit-msg`)
- DCO sign-off on every commit (`commit-msg`)

## Image Registry

Images are pushed to `8gears.container-registry.com/8gcr/` by default.
Override with `REGISTRY_ADDRESS` and `REGISTRY_PROJECT` GitHub repository variables.

### Required GitHub Configuration

**Variables (Settings > Secrets and variables > Actions > Variables):**
- `REGISTRY_ADDRESS` - Registry hostname (default: `8gears.container-registry.com`)
- `REGISTRY_PROJECT` - Registry namespace (default: `8gcr`)
- `REGISTRY_USERNAME` - Registry login username
- `PATCHES_REPO` - (optional) GitHub repo path for enterprise patches

**Secrets (Settings > Secrets and variables > Actions > Secrets):**
- `REGISTRY_PASSWORD` - Registry login password
- `PATCHES_TOKEN` - (optional) GitHub token to clone the patches repo
