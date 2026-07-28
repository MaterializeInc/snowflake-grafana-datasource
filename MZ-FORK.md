# Why this fork exists

Materialize maintains this fork solely to publish **security-patched rebuilds** of
upstream releases. We do not intend to carry functional changes here.

## The problem

Upstream's most recent release is **v2.1.0 (2025-04-04)**, compiled with Go 1.24.1.
Upstream `main` has moved on considerably, but no release has been cut since. Because
the release artifact is frozen, its vendored Go dependencies are frozen too — and AWS
Inspector flags them on every rebuild of our Grafana image.

Measured against the image running in our ECS cluster, this plugin accounted for **83 of
92 active findings, including all 12 criticals**. None were OS packages; every one was a
Go module statically linked into `gpx_snowflake-datasource`.

## What this branch changes

Branched from the upstream `v2.1.0` tag, so **the plugin's own source is byte-for-byte
upstream**. Only the build inputs move:

- `go.mod` / `go.sum` — `go get -u ./...` + `go mod tidy`, which also pulls
  `grafana-plugin-sdk-go` from v0.274.0 to v0.294.0 and raises the `go` directive to
  1.26.5.
- `.github/workflows/release.yml` — `go-version` from `1.23` to `1.26.x`. This matters:
  several findings are Go **stdlib** CVEs that are only fixed in 1.26.x, so the
  toolchain version is part of the fix, not an incidental detail.
- `.github/workflows/release.yml` — build only `build:linux build:linuxARM64`. Upstream
  also ships darwin/windows/linuxARM; we deploy to Fargate only. ARM64 is retained
  deliberately so the artifact survives an arm64 migration of the Grafana task.
- `package.json` — version `2.1.0-mz.1`, to distinguish our rebuild from a genuine
  upstream `v2.1.0` and to satisfy the release workflow's tag/version equality check.

Verified before publishing: `go vet` clean, `go test ./pkg/...` passes all four
packages, cross-compiles for linux/amd64, and the frontend `yarn build` produces a
valid `dist/`. Every dependency meets or exceeds the fix version Inspector asks for.

## What this branch deliberately does NOT change

Query semantics. Our old fork carried a patch (this repo's PR #1) making
`$__timeFilter` inclusive via `BETWEEN`; upstream is exclusive (`col > … AND col < …`).
That patch was never upstreamed, and i2 stopped tracking this fork on 2025-01-15, so
**exclusive bounds are what has actually been running in production since then.** This
rebuild keeps upstream behaviour so it is a pure dependency change. Restoring inclusive
bounds is a separate, deliberate decision.

## Releasing

Tag `v2.1.0-mz.1`; `release.yml` builds and attaches
`snowflake-grafana-datasource.zip`. The tag must equal `v` + `package.json` version or
the workflow fails its own check. i2's Grafana `Dockerfile` installs that zip by URL.

The artifact is **unsigned**, so Grafana needs
`allow_loading_unsigned_plugins = michelin-snowflake-datasource` in `grafana.ini`. This
was already required for upstream's zip, which is also unsigned.

## Preferred long-term outcome

None of this is a good place to live. Better endings, roughly in order:

1. Upstream resumes cutting releases — we delete this fork and go back to
   `releases/latest`. Upstream's dependabot is active, so the deps are being bumped;
   only the release step is missing. Worth asking upstream to tag one.
2. We move to a maintained alternative Snowflake datasource.
3. We drop the datasource if it stops being used.

Until then, re-run `go get -u ./... && go mod tidy` and cut a new `-mz.N` tag whenever
Inspector lights up again.
