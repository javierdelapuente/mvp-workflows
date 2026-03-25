# operator-workflows — Current State Analysis & Redesign Notes

> Source: https://github.com/canonical/operator-workflows  
> Focus: integration tests and publishing charms/rocks  
> Context: ISD277 — operator-workflows redesign (Braindump, Mar 24 2026)

---

## Repository Overview

`operator-workflows` is a reusable GitHub Actions library for Canonical charms (Kubernetes and machine operators).  
It is built with **TypeScript GitHub Actions** (compiled to `dist/`) orchestrated by **YAML workflow files**.

```
.github/workflows/   ← reusable YAML workflows (callers use `uses: canonical/operator-workflows/...@main`)
src/                 ← TypeScript source for GitHub Actions
internal/            ← compiled GitHub Action bundles (plan, build, publish, etc.)
dist/                ← compiled dist artefacts
tests/               ← unit/integration tests for the TypeScript actions
```

---

## Data Model (`src/model.ts`)

```
Plan
  working_directory: string
  build: BuildPlan[]

BuildPlan
  type: 'charm' | 'rock' | 'docker-image' | 'file'
  name: string
  source_file: string
  source_directory: string
  build_target: string | undefined    ← only for file resources
  output_type: 'file' | 'registry'   ← how the image/rock is stored
  output: string                      ← artifact name (sanitised, unique per run)

CharmResource
  type: 'file' | 'oci-image'
  description?: string
  filename?: string
  upstream-source?: string
```

The `Plan` JSON is created at the start of every integration test run, stored as a GitHub Actions artifact, and later re-used verbatim by the publish workflow.

---

## TypeScript Actions (`src/`)

| File | GitHub Action | Purpose |
|---|---|---|
| `plan.ts` | `internal/plan` | Discovers charmcraft.yaml, rockcraft.yaml, *.Dockerfile, file resources → emits a `Plan` JSON |
| `build.ts` | `internal/build` | Builds one item from the plan (charm / rock / docker-image / file). Uploads artefact + manifest.json |
| `plan-integration.ts` | `internal/plan-integration` | Waits for all build jobs, downloads artefacts, assembles pytest CLI args (`--charm-file`, `--*-image`, etc.) |
| `plan-scan.ts` | `internal/plan-scan` | Produces scan matrix entries for Trivy image scanning |
| `get-plan.ts` | `internal/get-plan` | Used by publish: finds the matching integration test run by git tree ID, downloads its Plan |
| `publish.ts` | `internal/publish` | Downloads charm/rock/file artefacts from the integration run → uploads to Charmhub via `charmcraft` |
| `manifest.ts` | (shared) | Parses the manifest.json sidecar that each build step uploads alongside its artefact |
| `utils.ts` | (shared) | `mkdtemp`, `normalizePath` helpers |

### Plan generation detail (`plan.ts`)

Scans `working-directory` for:
- `**/charmcraft.yaml` → charm build entries (excludes `tests/` subtree)
- `**/*rockcraft.yaml` → rock build entries
- `**/*.Dockerfile` → docker-image build entries
- file resources declared in `charmcraft.yaml` / `metadata.yaml` resources of type `file` (unless description starts with `(local)`)

Image output type (`output_type`):
- `'artifact'` input → always `file`
- `'registry'` input → always `registry`
- default → `file` for fork PRs, `registry` otherwise

### Build detail (`build.ts`)

| Asset type | Toolchain | Caching |
|---|---|---|
| charm | `charmcraft pack` (or `ccc` if charmcraftcache enabled) | none |
| rock | `rockcraft pack` + skopeo copy to ghcr.io or tar | weekly manifest cache (registry rocks only, skipped for paas-charm rocks) |
| docker-image | `docker build` + save/push | none |
| file | runs `build-<resource>.sh` script | none |

Every build step emits a `manifest.json`:
```json
{ "name": "<asset-name>", "files": ["<file.charm>"] }
// or
{ "name": "<asset-name>", "images": ["ghcr.io/org/name:sha"] }
```

### Publish detail (`publish.ts` + `get-plan.ts`)

1. `get-plan` finds the most recent integration test workflow run that matches the current **git tree SHA** (last 14 days).  
   Optionally overridden by `workflow-run-id` input.  
   Multiple plan artifacts (e.g. from different arch jobs) are merged — deduplicating by `output`.
2. `publish` downloads charm artefacts, resolves OCI resource names via `resource-mapping` input or `<rock-name>-image` convention, uploads images via `charmcraft upload-resource --image=<dockerID>` or skopeo for `.rock` files.
3. `charming-actions/upload-charm` uploads the `.charm` file(s) and releases to the destination channel.

---

## Workflow Architecture

### `integration_test.yaml` (main entry point, 18 KB, ~50 inputs)

```
plan        [internal/plan]        → discovers what to build, detects code-only changes
  ↓
build       [internal/build]       → parallel matrix over plan.build items
  ↓
plan-scan   [internal/plan-scan]   → generates Trivy scan matrix
  ↓
scan        [Trivy]                → parallel matrix over images
integration-test → integration_test_run.yaml
  ↓
required_status_checks             → gating job
```

Key inputs (selected):

| Input | Default | Notes |
|---|---|---|
| `provider` | `microk8s` | Juju substrate |
| `channel` | `latest/stable` | actions-operator channel |
| `juju-channel` | `2.9/stable` | |
| `modules` | `[""]` | pytest `-k` selectors |
| `series` | `[""]` | `--series` args |
| `extra-test-matrix` | `{}` | additional matrix axes |
| `upload-image` | `""` | `artifact` / `registry` / auto |
| `self-hosted-runner` | false | |
| `tmate-debug` | false | |
| `zap-enabled` | false | OWASP ZAP |
| `trivy-fs-enabled` | false | Trivy filesystem scan |
| `load-test-enabled` | false | |
| `pre-build-script` | `""` | bash script before build |
| `pre-run-script` | `""` | bash script before tests |

### `integration_test_run.yaml` (actual test execution, 25 KB)

Handles the full test matrix (`series × modules × extra-test-matrix`). Jobs:

```
select-runner           → decides GitHub-hosted vs self-hosted runner
plan-integration        → [internal/plan-integration] waits for builds, assembles --charm-file / --*-image args
integration-test        → matrix of (series, module) → runs tox -e integration
  (optional) setup-devstack-swift
  (optional) canonical-k8s provisioning
  (optional) microk8s provisioning
  (optional) lxd/lxc provisioning
  (optional) tmate on failure
trivy-fs                → Trivy FS scan
zap                     → OWASP ZAP scan
load-test               → k6 or locust
required_status_checks
```

### `publish_charm.yaml` (8 KB)

```
select-channel          → charming-actions/channel (branch/tag → Charmhub channel)
plan                    → [internal/get-plan] finds integration test run by tree SHA
  ↓
publish-charm           → [internal/publish] downloads artefacts → charmcraft upload-resource + upload-charm
draft-publish-docs      → discourse-gatekeeper (dry_run: true)
release-charm-libs      → charming-actions/release-libraries
```

Key publish inputs:

| Input | Default | Notes |
|---|---|---|
| `channel` | auto | destination channel |
| `integration-test-workflow-file` | `integration_test.yaml` | workflow to search for matching run |
| `identifier` | `""` | job identifier for multi-job repos |
| `resource-mapping` | `{}` | rock-name → resource-name mapping |
| `workflow-run-id` | `""` | explicit run override |
| `force-publish` | false | skip doc-only change check |
| `publish-docs` | false | Discourse draft publish |
| `publish-libs` | true | release charm libs |

### `promote_charm.yaml`

Promotes from origin channel → target channel via `charming-actions/promote-charm`.

---

## Known Pain Points (from ISD277, Mar 24 2026)

| # | Problem | Location |
|---|---|---|
| 1 | No monorepo support — can't associate charms with their rocks across subdirs | `plan.ts`, `publish_charm.yaml` |
| 2 | ~50 inputs in `integration_test.yaml` with mixed concerns (ZAP, Trivy, load, provisioning, devstack) | `integration_test.yaml`, `integration_test_run.yaml` |
| 3 | Organic YAML/TypeScript/Bash mixture — hard to reason about | entire repo |
| 4 | No pinning — large blast radius on any change | all `@main` references |
| 5 | Difficult local testing | plan/build/publish TypeScript actions |
| 6 | Plan artifact lookup limited to 14-day window | `get-plan.ts:54` |
| 7 | Publish only looks at most recent matching integration test run per tree SHA | `get-plan.ts` |
| 8 | `required_status_checks` gating is brittle (manually enumerates job names) | both integration_test.yaml and _run.yaml |

---

## Current Artefact / Data Flow

```
┌─────────────────────────────────────────────────────────┐
│  integration_test.yaml                                   │
│                                                          │
│  plan ──► build[0] ──► plan--charm.charm  (artifact)    │
│       ──► build[1] ──► plan--rock.rock    (artifact)     │
│       ──► build[n] ──► ...                               │
│                                                          │
│  plan-integration ─ waits for build jobs ─► pytest args  │
│                                                          │
│  tox -e integration \                                    │
│    --charm-file=./my-charm.charm \                       │
│    --my-rock-image=localhost:32000/my-rock:sha \         │
│    --series jammy -k module_a                            │
└─────────────────────────────────────────────────────────┘
                         │  plan artifact (plan.json)
                         ▼
┌─────────────────────────────────────────────────────────┐
│  publish_charm.yaml                                      │
│                                                          │
│  get-plan ──► finds matching run by git tree SHA         │
│           ──► downloads plan.json artifact               │
│                                                          │
│  publish  ──► downloads charm artifact ──► charmcraft    │
│           ──► downloads rock artifact  ──► docker pull   │
│           ──► charmcraft upload-resource --image         │
│           ──► charming-actions/upload-charm              │
└─────────────────────────────────────────────────────────┘
```

---

## MVP Test Targets (ISD277)

- **pollen-operator** — simple machine charm, 1 charm, no rock
- **github-runner-operators** — 2× 12-factor charms in a monorepo
- **paas-charm** — 12-factor, rock embedded in charm dir (triggers special paas-charm rock caching bypass)

---

---

## Redesign: Build Plan Data Model

### Decisions

| Decision | Choice | Notes |
|---|---|---|
| File name | `build-plan.yaml` | Lives at repo root |
| Format | YAML | Human-friendly; auto-generated or hand-written |
| Auto-generation | Tool discovers `charmcraft.yaml`, `rockcraft.yaml`, `snapcraft.yaml` | Simple repos fully auto; complex/monorepos override manually |
| Charm→resource relationships | **Explicit in the plan** | Not inferred at runtime from charmcraft metadata |
| Upstream-source OCI resources | **Invisible** — not in the plan | charmcraft handles them; no build step generated |
| Monorepo | One `build-plan.yaml` covers the whole repo | Multiple charms, rocks, snaps in a single file |
| Snaps | Included as first-class citizens | Alongside rocks and charms |
| Schema version | `version: 1` field | Future-proof |
| Runtime consumption | TBD — local-first, GitHub Actions integration deferred | Test locally before wiring into CI |

### Proposed `build-plan.yaml` Schema

```yaml
version: 1

rocks:
  - name: my-rock
    source: ./rocks/my-rock      # directory containing rockcraft.yaml

snaps:
  - name: my-snap
    source: ./snap               # directory containing snapcraft.yaml

charms:
  - name: my-charm
    source: ./                   # directory containing charmcraft.yaml
    resources:
      my-rock-image:             # resource name as in charmcraft.yaml metadata
        type: oci-image
        rock: my-rock            # references a rock defined above by name
      my-config-file:
        type: file
        script: build-my-config.sh   # script to build the file, relative to charm source dir
        filename: config.bin         # expected output filename
```

### Examples

**Simple repo (1 charm, no rock):**
```yaml
version: 1
charms:
  - name: pollen
    source: ./
```

**Simple repo (1 charm + 1 rock):**
```yaml
version: 1
rocks:
  - name: pollen-rock
    source: ./rock
charms:
  - name: pollen
    source: ./
    resources:
      pollen-image:
        type: oci-image
        rock: pollen-rock
```

**Monorepo (2 charms sharing a rock):**
```yaml
version: 1
rocks:
  - name: app-rock
    source: ./rocks/app
charms:
  - name: charm-a
    source: ./charms/charm-a
    resources:
      app-image:
        type: oci-image
        rock: app-rock
  - name: charm-b
    source: ./charms/charm-b
    resources:
      app-image:
        type: oci-image
        rock: app-rock
```

### What the auto-generator tool does

Given a repo root, the tool:
1. Finds all `charmcraft.yaml` files (excluding `tests/`) → creates a charm entry
2. Finds all `rockcraft.yaml` files → creates a rock entry
3. Finds all `snapcraft.yaml` files → creates a snap entry
4. For each charm, reads `charmcraft.yaml` resources; for each `oci-image` resource **without** `upstream-source`, tries to match it to a discovered rock by name convention
5. For each `file` resource (without `(local)` description), creates a file resource entry

The tool emits `build-plan.yaml`. Developer can then edit it for non-standard layouts.

### Asset Types

The plan supports exactly three asset types: **rocks**, **snaps**, and **charms**.  
`docker-image` (raw `*.Dockerfile`) is **not** a first-class type — rocks replace that use case.

### Open Questions

- [ ] Should snaps declare resources too (like charms do)?
- [ ] How does the plan reference a rock that is embedded inside the charm directory (e.g. paas-charm pattern)?
- [ ] Validation: should the tool validate that all referenced rock names exist in the plan?

---

## Redesign: Build Output Model

After the build runs, the build tool writes a `build-output.yaml` — the same structure as `build-plan.yaml` with an `output` field added to each built item. Source/relationship info is preserved so the file is self-contained.

### Shape

```yaml
version: 1

rocks:
  - name: my-rock
    source: ./rocks/my-rock               # from plan
    output: localhost:32000/my-rock:abc123  # image ref (registry) or file path (local)

snaps:
  - name: my-snap
    source: ./snap
    output: ./my-snap_1.0_amd64.snap

charms:
  - name: my-charm
    source: ./
    output:
      - ./my-charm_ubuntu-22.04-amd64.charm  # one entry per base
    resources:
      my-rock-image:
        type: oci-image
        rock: my-rock
        output: localhost:32000/my-rock:abc123  # resolved from the rock above
      my-config-file:
        type: file
        script: build-my-config.sh
        filename: config.bin
        output: ./config.bin
```

### Properties

- **Self-contained** — test runners and publish tools only need this one file
- **Relationships preserved** — the `rock:` reference stays, traceable back to the source
- **Same schema** — plan and output share the same top-level structure; `output` is the only addition
- **Rock output appears twice** — at top level (standalone consumers) and inline under charm resources (convenience); both point to the same value
- **Local-friendly** — `output` is a local file path when building locally, an image ref when pushed to a registry

---

## MVP Tools (Local-only, Bash)

Two bash scripts in `mvp-workflows/`. Dependency: `yq` for YAML read/write.  
Local-first — no GitHub Actions integration yet.

---

### Tool 1: `generate-plan`

Scans a repo directory and writes `build-plan.yaml`.

**Usage:**
```bash
./generate-plan [REPO_DIR] [OUTPUT_FILE]
# defaults: REPO_DIR=. OUTPUT_FILE=build-plan.yaml
```

**Algorithm:**
1. Find all `rockcraft.yaml` files (excluding `tests/`) → rock entries with `source: <dir>`
2. Find all `snapcraft.yaml` files (excluding `tests/`) → snap entries with `source: <dir>`
3. Find all `charmcraft.yaml` files (excluding `tests/`) → charm entries with `source: <dir>`
4. For each charm, read its `charmcraft.yaml` resources:
   - `oci-image` without `upstream-source` → try to match by name to a discovered rock (`<resource-name>` or `<resource-name>-image` → rock named `<resource-name>`)
   - `file` without `(local)` description → file resource entry
5. Write `build-plan.yaml`

**Output:** `build-plan.yaml` at `OUTPUT_FILE` path.  
Developer edits manually if the heuristic match is wrong or for monorepo layouts.

---

### Tool 2: `build`

Reads `build-plan.yaml`, builds all assets, writes `build-output.yaml`.

**Usage:**
```bash
./build [PLAN_FILE] [OUTPUT_FILE]
# defaults: PLAN_FILE=build-plan.yaml OUTPUT_FILE=build-output.yaml
```

**Algorithm:**
1. Read `build-plan.yaml`
2. For each **rock**: run `rockcraft pack` in `source` dir → find produced `.rock` file → record `output.file`
3. For each **snap**: run `snapcraft pack` in `source` dir → find produced `.snap` file → record `output.file`
4. For each **charm**: run `charmcraft pack` in `source` dir → find produced `.charm` file(s) → record `output.files`
5. Resolve each charm resource output from its rock's output
6. Write `build-output.yaml` (plan structure + `output` fields filled in)

**Output:** `build-output.yaml` at `OUTPUT_FILE` path.

---

### Annotated Format Reference

**`build-plan.yaml`:**
```yaml
version: 1

# Rocks to build. Each entry references a directory containing rockcraft.yaml.
rocks:
  - name: my-rock          # must match the `name` field in rockcraft.yaml
    source: ./rocks/my-rock

# Snaps to build. Each entry references a directory containing snapcraft.yaml.
snaps:
  - name: my-snap          # must match the `name` field in snapcraft.yaml
    source: ./snap

# Charms to build. Each entry references a directory containing charmcraft.yaml.
charms:
  - name: my-charm         # must match the `name` field in charmcraft.yaml
    source: ./
    resources:
      # OCI image resource — references a rock defined above by name
      my-rock-image:
        type: oci-image
        rock: my-rock
      # File resource — built by a script in the charm source directory
      my-config-file:
        type: file
        script: build-my-config.sh   # relative to charm source dir
        filename: config.bin         # expected output filename
```

**`build-output.yaml`** (same structure + `output` fields):
```yaml
version: 1

rocks:
  - name: my-rock
    source: ./rocks/my-rock
    output:
      file: ./rocks/my-rock/my-rock_1.0_amd64.rock   # path to produced .rock file

snaps:
  - name: my-snap
    source: ./snap
    output:
      file: ./snap/my-snap_1.0_amd64.snap

charms:
  - name: my-charm
    source: ./
    output:
      files:
        - ./my-charm_ubuntu-22.04-amd64.charm         # one per base
    resources:
      my-rock-image:
        type: oci-image
        rock: my-rock
        output:
          file: ./rocks/my-rock/my-rock_1.0_amd64.rock  # resolved from rock above
      my-config-file:
        type: file
        script: build-my-config.sh
        filename: config.bin
        output:
          file: ./config.bin
```

| File | Lines | Relevance |
|---|---|---|
| `.github/workflows/integration_test.yaml` | ~450 | entry point, 50 inputs |
| `.github/workflows/integration_test_run.yaml` | ~800 | execution, all test concerns |
| `.github/workflows/publish_charm.yaml` | ~200 | publish flow |
| `src/plan.ts` | ~200 | auto-discovery of build targets |
| `src/build.ts` | ~300 | per-asset build logic |
| `src/get-plan.ts` | ~130 | publish plan retrieval |
| `src/publish.ts` | ~290 | charmhub upload |
| `src/plan-integration.ts` | ~130 | test argument assembly |
| `src/model.ts` | ~25 | central data types |

---

## MVP Status

### Completed ✓

**pollen-operator** (simple machine charm, 1 charm, no rock):

| Step | Command | Result |
|---|---|---|
| Generate plan | `./generate-plan pollen-operator` | `build-plan.yaml` ✓ |
| Build | `./build pollen-operator` | `pollen_ubuntu-22.04-amd64.charm` + `build-output.yaml` ✓ |
| Provision | `concierge prepare` (via `./test`) | Juju 3.6/stable + LXD controller ✓ |
| Test | `./test pollen-operator` | **3 passed** ✓ |

**Known issues fixed during testing:**
- `yq` trailing comma in array construction (`["foo",]` → `["foo"]`)
- Absolute charm path caused double-slash in pytest (`./` + `/abs/path`); fixed by keeping paths relative to repo dir

### Next

- [ ] Test against `github-runner-operators` (monorepo, 2× 12-factor charms)
