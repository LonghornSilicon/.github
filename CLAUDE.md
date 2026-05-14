# LonghornSilicon `.github` — Claude Code Guide

This file is loaded automatically by Claude Code sessions opened in this
repo. It's also useful as a human reference for the CI/CD setup. If
you're starting a new CI gate, adding a new block repo, or debugging
why something failed, read this first.

The actual work happens in **`.github/workflows/block-ci.yml`** —
the reusable workflow every LonghornSilicon block repo calls into.

---

## What this repo is

`LonghornSilicon/.github` is the org-special "infrastructure" repo for
LonghornSilicon. GitHub treats it specially:

- `profile/README.md` is rendered as the **org's landing page** at
  https://github.com/LonghornSilicon
- `.github/workflows/*.yml` files here can be **reused by every repo in
  the org** via `uses: LonghornSilicon/.github/.github/workflows/<name>.yml@main`

So this repo holds *shared infrastructure*, not chip code. Edit it
carefully — a change here propagates to every block repo on the next
push.

---

## What's in here

```
LonghornSilicon/.github/
├── CLAUDE.md                          ← you are reading this
├── profile/
│   └── README.md                      ← org landing page (Lambda branding)
└── .github/
    └── workflows/
        └── block-ci.yml               ← the reusable CI workflow
```

That's it. Lean by design.

---

## How block repos call the workflow

Every LonghornSilicon block repo has a thin `.github/workflows/ci.yml`
that delegates to this one. Example (from
`LonghornSilicon/adaptive-precision-attention`):

```yaml
name: CI

on:
  push:
    branches: [master]
    paths-ignore:
      - 'README.md'
      - 'memory/**'
  pull_request:
    branches: [master]
  workflow_dispatch:

jobs:
  ci:
    uses: LonghornSilicon/.github/.github/workflows/block-ci.yml@main
    with:
      block-name: precision_controller        # SV top + LibreLane DESIGN_NAME
      expected-ff-count: 30                    # closed-form prediction
      has-reference-model: true                # sw/reference_model/ exists
      has-paper: true                          # paper/<block>.tex exists
```

That's the whole CI for the apa repo. ~25 lines. All the actual gate
logic lives in this repo's `block-ci.yml`.

---

## The 7 gates today (1 more disabled)

Every gate's first step is `About this gate (purpose and method)` —
click into a job on the GitHub Actions UI and expand that step to see
purpose / method / what-it-catches / why-it-matters at a glance.

| # | Gate | What | Default | Run time |
|---|---|---|---|---|
| 1 | `rtl-functional-verification` | iverilog testbench passes | always | ~1 min |
| 2 | `coverage-gate` | Verilator ≥ 90% line coverage | enabled (`has-coverage-gate`) | ~1 min |
| 3 | `rtl-synthesis` | Yosys synth + FF count assertion | always | ~30 sec |
| 4 | `formal-equivalence` | Yosys equiv_induct -seq 5 (RTL ≡ post-synth) | enabled (`has-formal-equivalence`) | ~2 min |
| 5 | `reference-model-tests` | sw/reference_model/Makefile test-all | enabled (`has-reference-model`) | ~30 sec |
| 6 | `openlane-sky130` | DRC/LVS/antenna/IR-drop/setup/hold all 0 | always | ~7-10 min |
| 7 | `paper-build` | pdflatex 4-pass with bibtex | opt-in (`has-paper`) | ~1 min |
| 8 | `cadence-16ffc-signoff` | Genus + Innovus + Tempus against TSMC PDK | DISABLED (`if: false`) | ~30-60 min (future) |

All 7 active gates run on **GitHub-hosted ubuntu-latest** — no self-hosted
runner needed. We tried self-hosted earlier; the cloud box was ARM64 and
LibreLane is x86_64 only, so we moved to GitHub-hosted for simplicity.
Free unlimited Linux minutes for public repos.

The 16FFC Cadence job is structurally present so we can enable it later
without re-architecting; flip `if: false` → `if: ${{ inputs.has-cadence-signoff }}`
when:
- TSMC PDK provisioned on a self-hosted runner
- Genus / Innovus / Tempus licenses available
- Runner labeled `self-hosted, linux, x64, tsmc-16ffc`

---

## Workflow inputs (current)

| Input | Type | Default | Purpose |
|---|---|---|---|
| `block-name` | string | required | SV top module + LibreLane DESIGN_NAME |
| `expected-ff-count` | number | required | Closed-form FF prediction the synth gate asserts |
| `has-reference-model` | bool | true | Run sw/reference_model/Makefile test-all |
| `has-paper` | bool | false | Run pdflatex on paper/<block>.tex |
| `has-coverage-gate` | bool | true | Run Verilator coverage assertion |
| `coverage-threshold` | number | 90 | Min line-coverage percentage |
| `has-formal-equivalence` | bool | true | Run Yosys equivalence proof |
| `extra-rtl-sources` | string | "" | Extra `.sv` files (relative to rtl/) for multi-file designs |
| `openlane-config-dir` | string | "" | Override `openlane/<block-name>/` path |

When adding a new input, follow this pattern:

```yaml
on:
  workflow_call:
    inputs:
      new-input-name:
        description: |
          One-paragraph human-readable explanation.
          Mention defaults, common values, when to override.
        type: string | number | boolean
        required: true | false
        default: <value>     # if not required
```

---

## How to add a new gate

The pattern is **always**:

1. **Add an input** (optional `has-<thing>` boolean to make it opt-in/out)
2. **Add the job** with:
   - `if: ${{ inputs.has-<thing> }}` if opt-in/out
   - An "About this gate" first step explaining purpose
   - The actual gate steps (install tools, run check, assert)
   - A step-summary write so the run page shows the result
   - Artifact upload on `if: always()` so even failures preserve evidence

3. **Update the top-of-file comment block** listing all gates
4. **Test on a sacrificial branch** before pushing to `main` (see "Testing
   changes safely" below)
5. **Push**. Every block repo gets the new gate on its next push.

Skeleton for a new gate:

```yaml
  # ====================================================================
  # N. <gate-name>
  #
  # Purpose
  # -------
  # One paragraph.
  #
  # What it does
  # ------------
  # One paragraph of mechanics.
  #
  # Why it matters
  # --------------
  # One paragraph of context.
  # ====================================================================
  my-new-gate:
    name: N. Human-readable name
    if: ${{ inputs.has-my-new-gate }}      # if opt-in
    runs-on: ubuntu-latest
    steps:
      - name: About this gate (purpose and method)
        run: |
          cat <<'EOF'
          ════════════════════════════════════════════════════════════════════
            Gate N — Human-Readable Name

            Purpose: <one sentence>
            Method:  <one paragraph>
            Catches: <which bug class>
            Why it matters: <broader context>
          ════════════════════════════════════════════════════════════════════
          EOF

      - uses: actions/checkout@v4

      - name: Install tools
        run: sudo apt-get install -y <tool>

      - name: Run the check
        working-directory: rtl
        run: |
          # ... do the work ...
          # ... assert pass/fail ...

          {
            echo "## N. <Gate name> — \`${{ inputs.block-name }}\`"
            echo ""
            echo "<markdown summary for the run page>"
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Upload artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: <gate-name>-${{ inputs.block-name }}
          path: <where artifacts live>
          retention-days: 30
```

---

## How to add a new block repo

1. **Create the repo** on GitHub: `LonghornSilicon/<block-name>` (public).
2. **Set up the directory structure** per the blueprint
   (`adaptive-precision-attention/docs/new_block_blueprint.md` is the
   canonical reference).
3. **Add a thin CI workflow** at `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push: { branches: [master] }
  pull_request: { branches: [master] }
  workflow_dispatch:

jobs:
  ci:
    uses: LonghornSilicon/.github/.github/workflows/block-ci.yml@main
    with:
      block-name: <name>                      # matches SV top
      expected-ff-count: <closed-form>        # the block's FF count prediction
      has-reference-model: true | false       # sw/reference_model/ exists?
      has-paper: true | false                 # paper/*.tex exists?
      # Optional:
      # has-coverage-gate: true (default)
      # has-formal-equivalence: true (default)
      # extra-rtl-sources: "sub_module1.sv sub_module2.sv"
```

That's the entire CI setup for a new block. ~25 lines.

---

## Testing workflow changes safely

A bug in `block-ci.yml@main` breaks CI on *every* LonghornSilicon
repo until fixed. Two patterns to test changes without breaking
everyone:

### Pattern A — Test on a branch first

1. Push your workflow change to a branch (e.g., `feat-new-gate`) on this repo
2. In a sacrificial block repo (or apa on a feature branch), change the
   workflow ref:
   ```yaml
   uses: LonghornSilicon/.github/.github/workflows/block-ci.yml@feat-new-gate
   ```
3. Push that block-repo branch. It uses the in-progress workflow.
4. If green, merge `feat-new-gate` to `main` on this repo.
5. Revert the block-repo ref back to `@main`.

### Pattern B — Commit-pin during stabilization

If a new gate is flaky for a while:

```yaml
uses: LonghornSilicon/.github/.github/workflows/block-ci.yml@<sha>
```

Pins the block repo to a specific commit. Unpins to `@main` once stable.

---

## Patterns we use throughout

### Step summaries

Every gate writes a markdown block to `$GITHUB_STEP_SUMMARY` so the run
page summary tab has the highlights without needing to expand step logs.
Pattern:

```bash
{
  echo "## N. Gate name — \`${{ inputs.block-name }}\`"
  echo ""
  echo "<markdown body>"
} >> "$GITHUB_STEP_SUMMARY"
```

### Environments for deployment widgets

`environment:` blocks on jobs make them show up as "Deployments" in the
right-hand sidebar of the repo home / PRs / commits — the same widget
Vercel uses. We do this for select jobs (e.g., `sky130-signoff`) so the
GDS PNG and run logs are visible at a glance:

```yaml
environment:
  name: sky130-signoff
  url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

### `if: always()` on artifact uploads

So failures still upload their evidence (synth logs, equiv logs,
coverage reports). Crucial for debugging in production.

### Concurrency cancellation

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

When a new push lands on the same branch, the older in-progress run
cancels. Saves CI minutes and prevents noise.

---

## Gotchas we hit during development

1. **`paths-ignore` and empty commits.** An empty commit (`--allow-empty`)
   may or may not trigger CI depending on how GitHub interprets "no
   paths changed" against `paths-ignore`. Don't rely on empty commits;
   make a real one-line edit somewhere outside `paths-ignore` to
   reliably trigger.

2. **Self-hosted runner labels.** We initially had `runs-on: [self-hosted, linux, x64]`
   then someone changed it to `ARM64` on a hunch. The actual cloud box
   was AMD EPYC (x86_64), so the runner advertised ARM64 but couldn't
   run x86_64 Docker images. We pulled the self-hosted runner entirely
   and switched to `ubuntu-latest`. **If a future Cadence runner is
   added, double-check the actual `uname -m` on the host vs. the
   `--labels` you registered with.**

3. **Yosys cell-name regex.** Yosys 0.33 emits cell names like
   `$_SDFFE_PN0P_` with **digits in the suffix**. A regex like
   `DFF[A-Z_]*\s+[0-9]+` stops at the first digit and never matches.
   Use `awk '/\$_S?DFF/ {sum+=$NF}'` instead — robust to suffix variants.

4. **LibreLane flag ordering.** `--docker-no-tty` MUST come before
   `--dockerized` or Docker still attaches a TTY and silently fails on
   non-interactive runners. Already enshrined in the workflow but
   don't reorder them.

5. **Coverage parsing.** Verilator's `verilator_coverage` prints
   percentages in different formats across versions. The current
   workflow grabs the last `%`-suffixed number from the output —
   reasonably robust but not bulletproof. If it ever picks up the
   wrong number, switch to `--write-info` for structured output.

6. **Per-commit git identity.** Don't run `git config user.email` /
   `user.name` — it propagates outside the immediate scope. Use
   per-command env vars instead:
   ```bash
   GIT_AUTHOR_NAME=themoddedcube \
   GIT_AUTHOR_EMAIL=themoddedcube@gmail.com \
   GIT_COMMITTER_NAME=themoddedcube \
   GIT_COMMITTER_EMAIL=themoddedcube@gmail.com \
   git commit ...
   ```
   This is the convention throughout LonghornSilicon repos. Match it.

7. **`if: false` jobs still appear in run logs.** The 16FFC Cadence job
   shows as "Skipped" on every run. Useful as a placeholder; don't
   delete it just because it's not running yet.

---

## Decision history (why we made certain choices)

- **GitHub-hosted runners over self-hosted.** Tested self-hosted; the
  ARM64 mismatch killed it. GitHub-hosted is free for public repos,
  has zero maintenance, and a clean 10-min run is acceptable for
  sub-daily CI.

- **Reusable workflow over composite action.** Composite actions are
  for single-step abstractions; reusable workflows are for
  multi-job ones. We have 7+ jobs; reusable workflow is the right tool.

- **Open-flow first, Cadence later.** Sky130 + OpenLane + Verilator +
  Yosys gives us a complete tape-out-style flow today. Cadence layers
  on top when PDK arrives; the existing gates don't go away.

- **One reusable workflow, not one per gate.** We considered splitting
  into per-gate reusable workflows (e.g., `synth.yml`, `coverage.yml`).
  Decided against — a single workflow with conditional jobs is easier
  to call (one `uses:` line) and easier to maintain (one file to read).

- **`@main` pinning, not `@<sha>`.** Block repos call
  `block-ci.yml@main` so improvements propagate automatically. The
  downside is a bad commit on main breaks every repo, so we test
  workflow changes carefully before pushing (see "Testing changes
  safely" above).

- **Documentation in three places.** YAML comments (for source readers),
  "About this gate" step (for CI UI viewers), step summaries (for run
  page viewers). Same content tripled — different audiences land on
  different surfaces.

---

## What's planned (roadmap)

Tier-1 wins worth adding before tape-out:

- [ ] Branch coverage in addition to line coverage (Verilator
      `--coverage-branch`)
- [ ] FSM coverage where applicable (`--coverage-toggle`)
- [ ] Lint gate (`verilator --lint-only`)
- [ ] SymbiYosys property checks (per block, opt-in)
- [ ] More PVT corners in OpenLane (the SDC already supports it; just
      tune the LibreLane config)
- [ ] `CODEOWNERS` enforcement on block repos

Tier-2 (needed for credible 16nm tape-out):

- [ ] DFT (scan + ATPG coverage gate)
- [ ] Power signoff with activity-driven workloads
- [ ] UVM-style testbenches
- [ ] Cadence 16FFC sign-off (the disabled job 8 — flip when ready)

Tier-3 not planned at our scale:

- Custom emulation platforms (ZeBu / Palladium)
- Auto-bisect / flake detection / regression management dashboards
- Specialized DV / PD / DFT teams

---

## When you ask Claude to do CI work

Common requests and how they map to this repo:

| Request | What to do |
|---|---|
| "Add a coverage gate" | Already done. See job 2 in `block-ci.yml`. |
| "Add lint gate" | New job + `has-lint` input. Use Verilator `--lint-only`. See "How to add a new gate" above. |
| "Why is X failing on apa?" | Look at apa's run page; the per-gate "About this gate" step explains what the gate expects. |
| "Add a new block repo's CI" | Copy the thin caller from apa's `.github/workflows/ci.yml`, change inputs. |
| "Make CI fail on warnings" | Most jobs already do; for Verilator change `-Wno-fatal` to default. |
| "Run CI on a private branch without affecting main" | See "Testing changes safely" above. |
| "What does gate N do?" | Read the docblock above the job in `block-ci.yml`, OR click the job on a run page and expand "About this gate". |

---

## Files you'll touch when modifying CI

- **`.github/workflows/block-ci.yml`** (this repo) — the actual workflow
- **`profile/README.md`** (this repo) — org landing page; don't touch
  for CI work
- **Block repo's `.github/workflows/ci.yml`** — the thin caller; only
  edit when changing block-specific inputs

You should almost never need to touch anything else for pure CI work.

---

## Contact

This guide and the workflow are maintained by Chaithu Talasila
(UT Austin) and the LonghornSilicon team. For bigger architectural
changes, propose on a feature branch and review before merging to
`main`.
