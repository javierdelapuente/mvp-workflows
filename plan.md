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

---

## Tool Design: Four Scripts

```
generate-plan   reads repo → writes build-plan.yaml
build           reads build-plan.yaml → builds assets → writes build-output.yaml
provision       reads concierge.yaml → runs concierge prepare
wait-build      (CI only) polls GitHub API → waits for build jobs → downloads build-output.yaml
test            reads build-output.yaml → runs tox -e integration [--module X]
```

### Local workflow
```bash
./generate-plan .
./build .
./provision .
./test . [--module test_charm.py]
```

### CI workflow (parallel build + provision)
```
Job A (build):   ./build  →  upload build-output.yaml artifact
Job B (test):    ./provision
                 ./wait-build        ← blocks until Job A done, downloads build-output.yaml
                 ./test --module X
```

Both jobs start simultaneously. Provisioning and building overlap.
`wait-build` is the synchronisation point.

### provision script
- If `concierge.yaml` exists in repo: `sudo concierge prepare`
- Otherwise: warn and continue (assume environment already ready)

### wait-build script (CI only)
- Uses `gh` CLI (available on all GitHub-hosted runners)
- Polls `GET /repos/:repo/actions/runs/:run_id/jobs` every 10s
- Finds jobs whose names contain `/ Build` with matching prefix
- Waits for all to reach `conclusion: success`
- Downloads `build-output` artifact to current directory
- Fails if any build job fails

### test script
- Reads `build-output.yaml`
- Assembles pytest posargs: `--charm-file`, `--<resource>=<image>`
- Accepts optional `--module <file>` flag → passes `-k <file>` to pytest
- Runs `tox -e integration -- <posargs>`

### Open questions
- [ ] Where does the modules list live for CI matrix fan-out? (`test-plan.yaml`? workflow call-site? → probably spread.yaml itself)
- [x] Spread: decided — see section below

---

## Spread Integration Design

### Core idea: spread.yaml as single source of truth

`spread.yaml` lives in the repo and defines the test modules as **variants**. It drives:
- **Local parallel testing** via `spread lxdvm:` (N VMs in parallel)
- **CI matrix generation** — GitHub Actions parses `spread.yaml` to build the matrix dynamically
- **CI test execution** — each matrix job runs `spread ci: integration/test:$MODULE`

Adding a new test module = add a variant to `spread.yaml`. Both local and CI pick it up automatically. No separate module list to maintain.

### Mandatory or optional?

**Optional** — repos with **one module** use `provision → test` directly and don't need `spread.yaml`. Repos with **multiple modules** commit `spread.yaml`.

### Two backends

```
lxdvm    allocates a fresh LXD VM per worker (lxc launch --vm)
         VMs are full machines → Juju/LXD runs natively, no nesting
         workers: N → N VMs in parallel
         used for: local parallel dev

ci       makes the current machine available to spread via SSH (root + password)
         no new VM allocated — spread SSHes to localhost
         workers: 1 (the runner IS the machine, already provisioned)
         used for: each CI matrix job
```

Learned from charmcraft's spread setup: they use the same two-backend pattern with a `.extension` shell script that implements `allocate/discard/backend-prepare/backend-restore` hooks.

### spread.yaml for indico-operator

```yaml
# spread.yaml (at repo root)
project: indico-operator

backends:
  lxdvm:
    type: adhoc
    allocate: HOST:.extension allocate lxd-vm
    discard:  HOST:.extension discard lxd-vm
    prepare:  HOST:.extension backend-prepare
    restore:  HOST:.extension backend-restore
    systems:
      - ubuntu-22.04:
          workers: 5           # N VMs = N modules, for local parallel run
  ci:
    type: adhoc
    allocate: HOST:.extension allocate ci
    discard:  HOST:.extension discard ci
    prepare:  HOST:.extension backend-prepare
    restore:  HOST:.extension backend-restore
    systems:
      - ubuntu-22.04:
          workers: 1

path: /root/project

suites:
  integration/:
    summary: Integration tests
    environment:
      MODULE/test_actions: test_actions
      MODULE/test_charm:   test_charm
      MODULE/test_loki:    test_loki
      MODULE/test_s3:      test_s3
      MODULE/test_saml:    test_saml
    prepare: |
      ./provision .            # runs ONCE per worker (not per variant)
```

```yaml
# integration/task.yaml
summary: Run $MODULE integration test
kill-timeout: 90m
execute: |
  ./test . --module $MODULE
```

### CI workflow driven by spread.yaml

```
Job A (build):
  ./build → upload build-output artifact

Job generate-matrix:
  parse spread.yaml → extract variant names → output JSON array
  → drives strategy.matrix for Job B

Job B × N (test):      ← one per variant in spread.yaml
  ./provision
  ./wait-build         ← downloads build-output.yaml
  spread ci: integration/test:$MODULE
```

`spread ci:` SSHes to localhost (the runner itself), copies the project (including `build-output.yaml`
downloaded by `wait-build`), and runs `./test . --module $MODULE` there.

**`wait-build` is still needed** — build and test jobs start simultaneously. `wait-build`
synchronises them (downloads `build-output.yaml` artifact once build completes).

### Reading modules from spread.yaml in GitHub Actions

```yaml
- name: Extract test modules
  id: modules
  run: |
    modules=$(yq '.suites["integration/"].environment | keys | .[]
                  | select(startswith("MODULE/")) | ltrimstr("MODULE/")' spread.yaml               | jq -R . | jq -sc .)
    echo "matrix=$modules" >> $GITHUB_OUTPUT
```

This produces e.g. `["test_actions","test_charm","test_loki","test_s3","test_saml"]`
which feeds `strategy.matrix.module`.

### No external script: generate-spread writes self-contained spread.yaml

We do **not** put a `.extension` file in each repo. Instead, a `generate-spread` tool writes a
**fully self-contained `spread.yaml`** with all allocate/discard/prepare/restore scripts embedded
inline (spread's `adhoc` backend supports this). Repos commit only `spread.yaml` + `integration/task.yaml`.

When the backend logic changes, teams re-run `./generate-spread .` to refresh their `spread.yaml`.
The generator is the single source of truth. The generated file is committed and stable.

```yaml
# spread.yaml — generated by ./generate-spread, then committed
project: indico-operator

backends:
  lxdvm:
    type: adhoc
    allocate: |
      # inline: create LXD VM and print its IP to fd 3
      VM_NAME="spread-lxdvm-${RANDOM}"
      lxc launch --vm ubuntu:22.04 "$VM_NAME" \
          -c limits.cpu=4 -c limits.memory=8GiB -d root,size=20GiB
      while ! lxc exec "$VM_NAME" -- true &>/dev/null; do sleep 0.5; done
      lxc exec "$VM_NAME" -- sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
      echo "root:$SPREAD_PASSWORD" | lxc exec "$VM_NAME" -- chpasswd
      ADDR=""
      while [ -z "$ADDR" ]; do
          ADDR=$(lxc ls -f csv | grep "^$VM_NAME" | cut -d, -f3 | cut -d' ' -f1)
      done
      echo "$ADDR" >&3
    discard: |
      instance=$(lxc ls -f csv | grep ",$SPREAD_SYSTEM_ADDRESS " | cut -f1 -d,)
      lxc delete -f "$instance"
    prepare: |
      snap wait system seed.loaded
      snap refresh --hold
    restore: |
      rm -rf "$SPREAD_PATH"
      mkdir -p "$SPREAD_PATH"
    systems:
      - ubuntu-22.04:
          workers: 5
  ci:
    type: adhoc
    allocate: |
      # inline: make this CI runner accessible via SSH
      sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
      sudo systemctl restart ssh
      echo "root:$SPREAD_PASSWORD" | sudo chpasswd
      echo localhost >&3
    discard: |
      true
    prepare: |
      snap wait system seed.loaded
    restore: |
      true
    systems:
      - ubuntu-22.04:
          workers: 1

path: /root/project

suites:
  integration/:
    summary: Integration tests
    environment:
      MODULE/test_actions: test_actions
      MODULE/test_charm:   test_charm
      MODULE/test_loki:    test_loki
      MODULE/test_s3:      test_s3
      MODULE/test_saml:    test_saml
    prepare: |
      ./provision .
```

### generate-spread tool

```
./generate-spread [REPO_DIR]
```

**Scaffold-once** behaviour:
- If `spread.yaml` already exists → **do nothing** (exit 0, print a notice)
- If `spread.yaml` does not exist → generate it from the test module list
- Same for `integration/task.yaml`

This means `generate-spread` is a one-time scaffold. After that, the team owns `spread.yaml`
and can add/remove variants or tweak backend options freely. It is hand-editable once committed.

Module discovery (used only on first run):
- Globs `tests/integration/test_*.py`, sorts, strips the `.py` extension

### Files per repo

```
spread.yaml             # scaffolded by generate-spread, then hand-maintained
integration/
  task.yaml             # scaffolded by generate-spread, then hand-editable
```

No `.extension`, no symlinks, no external dependencies at runtime.

### Local workflow (multi-module)

```bash
./generate-plan .
./build .              # produces build-output.yaml + .charm in repo root
spread lxdvm:          # allocates N VMs in parallel, provisions each, runs all modules
```

Single module locally (no spread needed):
```bash
./provision .
./test . --module test_charm
```

### Revised workflow picture

```
Local (single module, no spread):
  generate-plan → build → provision → test --module X

Local (all modules, parallel):
  generate-plan → build → spread lxdvm:
  (spread: N VMs × N variants → provision + test --module X each)

CI (GH Actions matrix, driven by spread.yaml):
  Job A (build):           build → upload artifact
  Job generate-matrix:     parse spread.yaml variants → output module list
  Job B × N (test):        provision → wait-build → spread ci: integration/test:$MODULE
```

---

## Session 3 — Spread Integration & LXD VM End-to-End Testing

### What was built

Five bash tools in `mvp-workflows/`:

| Tool | Purpose |
|------|---------|
| `generate-plan` | Reads `charmcraft.yaml`/`rockcraft.yaml`, writes `build-plan.yaml` |
| `build` | Builds rocks/charms per plan, writes `build-output.yaml`, pushes rocks to `localhost:32000` |
| `provision` | Runs `concierge prepare` for local use |
| `test` | Reads `build-output.yaml`, assembles pytest args, runs `tox -e integration` |
| `generate-spread` | Scaffolds `spread.yaml` + `integration/run/task.yaml` (once only) |

### Spread design

- `lxdvm` backend: real LXD VMs (`lxc launch --vm`), 3 workers, 8GiB RAM, 20GiB disk
- `ci` backend: runs on the GitHub Actions runner itself (no allocation)
- Suite prepare: self-contained — installs concierge, bootstraps Juju/microk8s, installs tox
- Task execute: self-contained — reads `build-output.yaml` with Python, pushes rocks to registry if needed, runs tox
- Variants: one per test module (e.g. `MODULE/test_charm: test_charm`)

### Key fixes discovered during testing

1. **SSH**: Ubuntu 22.04 VM overrides sshd config via `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` — must write `PasswordAuthentication yes` there explicitly
2. **ADDRESS API**: spread adhoc allocate uses `ADDRESS "$IP"` bash function (not `echo ADDRESS=$IP`)
3. **Self-contained tasks**: suite prepare/task execute can't call `../mvp-workflows/provision` since spread only packs the charm repo dir; logic is inlined instead
4. **concierge**: must pass `-c "$SPREAD_PATH/concierge.yaml"` explicitly; without it falls back to `dev` preset
5. **tox on Ubuntu 22.04**: `pip3 install tox tox-uv` (no `--break-system-packages` on older pip)
6. **pytest args**: charm via `--charm-file=<path>`, resources via `--<resource-name>=<ref>` (not `--resource`)
7. **MetalLB**: indico needs `microk8s enable metallb:"${NET_PREFIX}.200-${NET_PREFIX}.210"` (dynamic subnet from VM IP)

### Results

| Repo | `spread lxdvm:` | Notes |
|------|----------------|-------|
| pollen-operator | ✅ PASSED | Machine charm, LXD-in-LXD-VM |
| indico-operator | ⚠️ tooling OK, test OOM | k8s charm + full stack needs >8GiB RAM |

### Known limitations / next steps

- `wait-build` script (CI: polls GitHub API until charm/rock builds finish) — not yet implemented
- indico `test_charm` needs a larger VM (>8GiB) due to full k8s stack resource usage
- `generate-spread` does not regenerate if `spread.yaml` already exists (intentional scaffold-once)
