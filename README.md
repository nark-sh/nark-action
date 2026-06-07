# nark-action

GitHub Action that scans TypeScript projects for missing dependency error handling against 169+ npm package profiles.

Wraps [`nark`](https://github.com/nark-sh/nark) so you can add a single line to your CI workflow — with the verify-artifact step, OOM-safe `NODE_OPTIONS`, telemetry-off default, automatic artifact upload, and a parsed step summary all wired in by default.

## Quick start

```yaml
- uses: nark-sh/nark-action@v2
```

That's it. nark auto-detects your `tsconfig.json`, scans your project, writes `nark-audit.json`, posts a step summary with violation counts, and uploads the audit as a workflow artifact.

## Why v2?

v2 ships the dogfood-proven defaults we learned the hard way running Nark on our own SaaS — specifically the protections against [the silent-success CI failure mode](https://nark.sh/articles/github-actions-security-scanner-silent-success):

- **`NODE_OPTIONS=--max-old-space-size=8192`** by default — GitHub-hosted runners default Node's old-gen heap to ~2 GB, which OOMs mid-sized TS programs at exit 134 (`FATAL ERROR: Reached heap limit`).
- **Verify-artifact step built in** — when the scanner crashes, `nark-audit.json` isn't written, `actions/upload-artifact@v4` emits a warning, and `continue-on-error: true` reports the whole job green. v2 fails loud instead.
- **Telemetry off by default** in CI — the SaaS receipts that motivate telemetry come from interactive scans, not CI runs.
- **Automatic artifact upload + step summary** — no need to wire `actions/upload-artifact` or write your own summary script.

See [Recommended rollout](https://nark.sh/recommended-rollout) for the 3-phase shadow-mode rollout pattern v2 is designed for.

## Migrating from v1

v1 was a thin wrapper: it called `npx nark` with your inputs and exited. That's it. v2 adds output handling, artifact upload, and a step summary by default. Most callers can flip the tag:

```diff
- uses: nark-sh/nark-action@v1
+ uses: nark-sh/nark-action@v2
```

You'll see a new `nark-audit` workflow artifact on every run + a step summary panel. If you were already running your own `actions/upload-artifact@v4` step after the v1 action, either remove yours or set `upload-artifact: 'false'` on v2 to avoid duplicates.

## Full example

```yaml
name: CI
on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci

      # Generate any client code Nark's TS analyzer needs to resolve
      # types. Prisma is the common case; drop this if you don't use it.
      - run: npx prisma generate

      - name: Check error handling
        uses: nark-sh/nark-action@v2
        with:
          tsconfig: ./apps/web/tsconfig.json
```

## Shadow-mode rollout (recommended for first run)

The first run on any non-Nark-aware codebase typically surfaces 50–250 findings. Block-from-day-one is how scanners get ripped out. Use `continue-on-error: true` for the first week or two, then drop it once the baseline is triaged:

```yaml
- uses: nark-sh/nark-action@v2
  id: nark
  continue-on-error: true   # ← Phase 1: shadow mode

- name: Bot comment on findings
  if: steps.nark.outputs.total-violations != '0'
  run: |
    echo "Nark found ${{ steps.nark.outputs.total-violations }} violations:"
    echo "  errors: ${{ steps.nark.outputs.error-count }}"
    echo "  warnings: ${{ steps.nark.outputs.warning-count }}"
```

Full walkthrough at [nark.sh/recommended-rollout](https://nark.sh/recommended-rollout).

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `tsconfig` | Path to tsconfig.json (auto-detected if omitted) | `''` |
| `fail-threshold` | Exit 1 if violations at or above this severity (`error` \| `warning` \| `info`) | `error` |
| `report-only` | Always exit 0 regardless of violations | `false` |
| `diff-base` | If set, only report violations on lines added/modified by `git diff $diff-base..HEAD`. Pair with `github.event.pull_request.base.sha` for PR scans. | `''` |
| `version` | nark version to use | `latest` |
| `output` | Path to write the audit JSON | `nark-audit.json` |
| `node-options` | `NODE_OPTIONS` for the nark step. Default raises heap to 8 GB to prevent OOM. Set to `''` to use Node defaults. | `--max-old-space-size=8192` |
| `verify-artifact` | If `true` (default), fail the step when nark exits without producing the audit JSON. Catches the silent-success failure mode. | `true` |
| `upload-artifact` | If `true` (default), upload the audit JSON as a workflow artifact. | `true` |
| `artifact-name` | Name for the uploaded artifact. | `nark-audit` |
| `artifact-retention-days` | Workflow artifact retention. GitHub default is 90. | `30` |
| `telemetry` | If `true`, leave `NARK_TELEMETRY` untouched (CLI default sends anonymous usage data). | `false` |

## Outputs

| Output | Description |
|--------|-------------|
| `audit-path` | Path of the produced audit JSON. Empty if no audit was written. |
| `total-violations` | Total violation count. `0` if no audit was produced. |
| `error-count` | Error-severity violation count. |
| `warning-count` | Warning-severity violation count. |
| `info-count` | Info-severity violation count. |

## Examples

### Custom tsconfig path

```yaml
- uses: nark-sh/nark-action@v2
  with:
    tsconfig: ./packages/api/tsconfig.json
```

### Report-only (don't fail the build)

```yaml
- uses: nark-sh/nark-action@v2
  with:
    report-only: 'true'
```

Equivalent to `continue-on-error: true` on the step, but distinguishes "scan succeeded with violations" from "scanner crashed" in the workflow conclusion.

### Fail on warnings too

```yaml
- uses: nark-sh/nark-action@v2
  with:
    fail-threshold: warning
```

### PR diff mode (only flag what the PR changed)

Only report violations on lines the PR actually added or modified. Pre-existing violations on untouched lines of modified files are excluded.

```yaml
name: Nark
on:
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          # Required so the action can compute the diff against the PR base
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - uses: nark-sh/nark-action@v2
        with:
          diff-base: ${{ github.event.pull_request.base.sha }}
```

**Note:** `github.event.pull_request.base.sha` only exists on `pull_request` events. Don't set `diff-base` on `push`-triggered runs unless you supply a different base (e.g. `${{ github.event.before }}`). When `diff-base` is empty, the action falls back to scanning the entire project.

### Pin to a specific nark version

```yaml
- uses: nark-sh/nark-action@v2
  with:
    version: '2.4.0'
```

### Skip the built-in artifact upload (you'll do it yourself)

```yaml
- uses: nark-sh/nark-action@v2
  id: nark
  with:
    upload-artifact: 'false'
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: my-custom-name
    path: ${{ steps.nark.outputs.audit-path }}
```

### Small projects: disable the heap bump

If your project is small enough that 2 GB is plenty, you can save a few hundred ms of Node startup by passing the Node default:

```yaml
- uses: nark-sh/nark-action@v2
  with:
    node-options: ''
```

## What nark checks

nark scans your TypeScript code against profiles for 169+ npm packages (axios, prisma, stripe, pg, ioredis, etc.) and reports places where your code calls library functions without proper error handling.

Learn more: [nark on GitHub](https://github.com/nark-sh/nark) | [nark.sh](https://nark.sh)

## License

AGPL-3.0
