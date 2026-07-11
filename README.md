# security-scan-workflows

Reusable GitHub Actions for container image and Kubernetes manifest security
scanning. Thin, parameterized wrappers around upstream scanners
([Trivy](https://github.com/aquasecurity/trivy)) — this repo does not
implement any scanning logic itself, only policy (severity thresholds,
blocking behavior) and wiring.

Generic by design: no project-specific values (account IDs, domains, secret
paths, etc.) are hardcoded anywhere in this repo. Everything varies by
`inputs:`/`secrets:` passed from the caller.

## What's here

- `actions/trivy-image-scan/` — a **composite action**. Use as a step inside
  the job that builds your image, so it can scan a locally built image
  (`docker build --load`) *before* that image is pushed anywhere.
- `.github/workflows/trivy-manifest-scan.yml` — a **`workflow_call` reusable
  workflow**. Renders a Kustomize directory and scans the output for
  misconfigurations and accidentally-committed secrets. Runs as its own job
  (no shared build state needed).

These are two different reuse mechanisms on purpose — see
[Why two mechanisms](#why-two-mechanisms) below.

## Usage

### Image scan (composite action)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v6

      - name: Build (local only, for scanning)
        uses: docker/build-push-action@v7
        with:
          context: .
          push: false
          load: true
          tags: local-scan:${{ github.sha }}

      - name: Security scan
        uses: leofreiheit/security-scan-workflows/actions/trivy-image-scan@v0
        with:
          image-ref: local-scan:${{ github.sha }}
          severity: CRITICAL
          exit-code: "0" # report-only while baselining; flip to "1" once ignore-list is stable

      # ... only after the scan step succeeds, do the real push ...
```

### Manifest scan (reusable workflow)

```yaml
jobs:
  scan:
    uses: leofreiheit/security-scan-workflows/.github/workflows/trivy-manifest-scan.yml@v0
    with:
      manifests-path: manifests/cashier
      severity: CRITICAL,HIGH
      exit-code: "1"
```

## Versioning & releases

SemVer via [release-please](https://github.com/googleapis/release-please),
driven by [Conventional Commits](https://www.conventionalcommits.org/)
(`feat:`, `fix:`, `chore:`, etc.) on `main`. Merging the release PR that
release-please opens cuts the tag and GitHub Release — nothing publishes
automatically just from pushing to `main`.

Starting at `0.1.0`: the `workflow_call`/composite-action input contracts
haven't been validated against a real caller yet, so backwards compatibility
isn't promised until `1.0.0`.

Callers can pin to:
- an exact tag (`@v0.1.0`) for full immutability, or
- a floating major tag (`@v0`, later `@v1`) to automatically pick up
  non-breaking updates — moved by the release workflow on every release.

**First release caveat:** the manifest file seeds `0.1.0`, but release-please
correlates against actual GitHub Releases/tags, and none exist yet on a brand
new repo. If the first release-please PR doesn't propose the expected
version, create the `v0.1.0` tag/release manually once and let release-please
take over from `0.2.0` onward.

## Why two mechanisms

A `workflow_call` reusable workflow always runs as its **own job** — it does
not share a runner or Docker daemon with the job that called it. That's fine
for the manifest scan (it only needs `checkout` + `kustomize build`), but it
rules out "scan the image I just built locally, before pushing" — the image
wouldn't exist in that separate job. A **composite action**, by contrast,
runs as a step *inside* the caller's existing job, so it can see whatever the
prior steps in that job produced. Hence: composite action for image scanning,
reusable workflow for manifest scanning.

## Status

Early scaffold, not yet validated against a real caller. Flag names for the
underlying Trivy invocation (`scan-type`, `scan-ref`, etc.) should be
double-checked against
[aquasecurity/trivy-action](https://github.com/aquasecurity/trivy-action)'s
current docs before the first real run.
