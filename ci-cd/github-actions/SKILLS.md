---
name: github-actions
description: >
  Stack-agnostic skill for writing GitHub Actions YAML for CI, tests, builds, and releases. Use
  whenever creating or editing files under .github/workflows/, adding CI checks, matrix builds,
  caching, deployments, or release automation for any language or framework. Enforces the
  repository rule that workflows must never create intermediate GitHub-hosted workflow artifacts.
---

# GitHub Actions: Stack-Agnostic CI Without Intermediate Artifacts

Use this skill when adding or reviewing GitHub Actions workflows. Workflows may use any language
or framework, but they must never park intermediate build outputs in GitHub's workflow artifact
storage.

## Non-Negotiable Rule

**Do not create intermediate workflow artifacts.**

A workflow must not build something in one job, upload it with `actions/upload-artifact` (or any
equivalent), and download it in another job to deploy, release, test, or inspect it.

Do not use:

- `actions/upload-artifact`
- `actions/download-artifact`
- `actions/upload-pages-artifact` and other deployment-artifact helpers
- third-party actions that upload/download workflow artifacts
- matrix jobs or reusable workflows that hand off build files through artifacts

These files count against GitHub artifact storage and create stale, duplicated build outputs.

Artifacts are allowed only as **final release assets attached to the GitHub Release page**, and only
when the user wants downloadable release binaries or packages. Upload them directly from the job
that built them with `gh release upload`, `gh release create ... files...`, or an equivalent
release-upload action. Never route them through workflow artifact storage first.

If a platform (for example GitHub Pages) requires an intermediate artifact, stop and ask the user
for an explicit exception. Do not silently add one.

## Accepted Workflow Shapes

Prefer one self-contained job that does everything for a given target:

```text
checkout -> setup toolchain -> restore dependency cache -> test -> build -> publish/deploy
```

- Run lint, tests, and build in the same job when they share a toolchain.
- Deploy from the same job that built the deployable output.
- For a release, build the final package in the job and upload it straight to the release.
- For a matrix, make every matrix job self-contained. Each job may build and upload its own final
  release assets directly. Do not have matrix jobs exchange files through artifacts.
- Use `actions/cache` or built-in setup caching for dependencies. Cache entries are not workflow
  artifacts, but never cache build outputs as a substitute for artifact handoff.
- Use job outputs only for tiny values such as a version string, tag, or boolean. Never pass file
  contents or paths to files that do not exist in the same job.
- If two stages genuinely need the same expensive build, combine them into one job or rebuild. Do
  not introduce an artifact handoff.

## Generic CI Workflow

Every CI workflow should validate and build without uploading anything.

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - name: Check out
        uses: actions/checkout@v4

      # Add one stack-specific setup, install, test, and build sequence here.
```

Do not add an upload step for coverage reports, binaries, logs, screenshots, test reports, or
dependency folders unless the user explicitly asks for a final release asset. Print summaries in
the job log instead. Building into the workspace or a temporary directory is local disk, not GitHub
artifact storage.

## Stack Examples

### Node.js: React, Vite, Next.js, Vue

```yaml
      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: frontend
        run: npm ci

      - name: Lint
        working-directory: frontend
        run: npm run lint --if-present

      - name: Test
        working-directory: frontend
        run: npm test --if-present

      - name: Build
        working-directory: frontend
        run: npm run build
```

Do not upload `dist/`, `.next/`, coverage, or dependency folders. If the build is a static site,
deploy it directly from this same job (for example `npx wrangler pages deploy dist`, a hosting
CLI, or `rsync`) instead of passing `dist/` to another job.

### Python

```yaml
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: requirements*.txt

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint
        run: ruff check .

      - name: Test
        run: pytest

      - name: Build package
        run: python -m build
```

Do not upload test reports, coverage files, virtual environments, or intermediate build folders.
For a release, upload the final wheel and sdist directly to the GitHub Release.

### Java: Maven or Gradle

```yaml
      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Test and package
        run: mvn -B --no-transfer-progress verify

      # Gradle alternative:
      # uses: gradle/actions/setup-gradle@v4
      # run: ./gradlew build
```

Do not upload `target/`, `build/`, `.gradle/`, surefire reports, or test-results XML. For a release,
attach the final JAR/WAR/native package directly to the GitHub Release.

### Go

```yaml
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Verify formatting
        run: test -z "$(gofmt -l .)"

      - name: Vet
        run: go vet ./...

      - name: Test
        run: go test ./...

      - name: Build
        run: go build ./...
```

Do not upload binaries produced by CI. For a release, build final binaries in the release job and
attach them directly to the GitHub Release.

### Other Stacks

The same pattern applies to Rust, PHP, Ruby, .NET, C/C++, Elixir, and others:

1. Set up the toolchain in the job.
2. Restore dependency caches.
3. Lint and test.
4. Build the final output in the same job.
5. Deploy or release directly, or do nothing.

Never insert an artifact upload/download step between these stages.

## Release Workflow Template

Release assets are the only artifacts this repository should produce. Build them in the same job
that attaches them to the release.

```yaml
name: Release

on:
  push:
    tags: ["v*"]

permissions:
  contents: write

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - name: Check out
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Set up the stack and run tests here.

      - name: Build final release assets
        run: |
          mkdir -p dist
          # Replace with the stack-specific build command.
          # Examples:
          #   Node/Vite: npm ci && npm run build && tar -C dist -czf ../dist/app.tar.gz .
          #   Python:    python -m build --outdir dist
          #   Java:      mvn -B package && cp target/*.jar dist/
          #   Go:        go build -trimpath -o dist/app .

      - name: Generate checksums
        run: |
          cd dist
          sha256sum * > checksums.txt

      - name: Publish release assets
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "${GITHUB_REF_NAME}" \
            --verify-tag \
            --generate-notes \
            dist/*
```

Guidelines for release workflows:

- Attach only deliverables users should download: final binaries, installers, archives, packages,
  wheels, JARs, checksums, signatures, and release notes.
- Never attach intermediate object files, dependency caches, test reports, coverage files, or
  temporary packages.
- Create checksums for every released binary or package.
- Keep the release job reproducible from the tagged commit.
- Do not create one release job per stage. Build and publish in the same job, or let independent
  platform jobs upload their final assets directly.
- If multiple jobs upload to one release, make each job idempotent (`gh release upload --clobber`)
  and coordinate with a stable release tag. Do not exchange files through artifacts.

## Cross-Platform Releases

If a target requires its native OS runner, use a matrix where each entry is self-contained:

```yaml
jobs:
  release:
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: ubuntu-latest
            target: linux-x64
          - os: macos-latest
            target: darwin-arm64
          - os: windows-latest
            target: windows-x64
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      # Set up the stack, test, and build the final asset in this job.
      # Upload this job's final asset directly to the release. No upload-artifact.
```

If release creation races are a concern, create the release once in a tiny preparation job and let
each build job upload its final asset directly. That is metadata coordination, not an artifact
handoff, and is acceptable. The preparation job must not upload or download build files.

## Anti-Patterns

Reject these patterns regardless of language:

```yaml
# FORBIDDEN: intermediate artifact handoff
- uses: actions/upload-artifact@v4
  with:
    name: build
    path: dist/

# FORBIDDEN: second job depends on downloaded artifact
- uses: actions/download-artifact@v4
  with:
    name: build
```

```yaml
# FORBIDDEN: build cache used as an artifact workaround
- uses: actions/cache/save@v4
  with:
    path: dist/
    key: build-${{ github.sha }}
```

```yaml
# FORBIDDEN: keep artifacts "just in case"
- uses: actions/upload-artifact@v4
  with:
    name: logs
    path: "*.log"
    retention-days: 30
```

Do not use `needs:` to chain a build job to a deploy/release job when the only connection is
uploading and downloading files. Merge the jobs or rebuild in the consumer.

## Review Checklist

Before finishing any workflow change:

1. Search the workflow for `upload-artifact`, `download-artifact`, `upload-pages-artifact`, and
   `artifact` and confirm there are no intermediate artifact uploads.
2. Confirm every deploy/release step runs in a job that has the built files available locally.
3. Confirm release assets are final user-facing deliverables only, attached directly to the release.
4. Confirm permissions are minimal; release jobs alone receive `contents: write`.
5. Confirm secrets are not exposed to pull requests from forks.
6. Confirm actions are pinned to major versions (or SHAs where the repository requires it).
7. Confirm `timeout-minutes` and `concurrency` are set for long or publish-related workflows.
8. If caches are used, confirm they cache dependencies/toolchains, not build outputs.

Useful local checks:

```bash
grep -RInE 'upload-artifact|download-artifact|upload-pages-artifact' .github/workflows
grep -RIn 'actions/cache' .github/workflows
```

Any artifact-related match must be justified as a final release asset upload. Otherwise remove it.

## Security and Reliability

- Use least-privilege `permissions`; default to `contents: read`.
- Use `github.token`/`GH_TOKEN` only in release or deploy steps.
- Never run untrusted pull-request code with repository write tokens.
- Set `timeout-minutes` so hung builds do not consume runners.
- Use `concurrency` for CI cancelation and to serialize releases.
- Prefer official actions and pin versions deliberately.
- Keep logs useful but never print secrets or tokens.
- Build release assets from the tagged commit and verify tags with `--verify-tag`.
