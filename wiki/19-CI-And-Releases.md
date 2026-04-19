# 19 · CI and Releases

Hermes uses GitHub Actions for continuous integration, Docker Hub for
image publishing, GitHub Pages + Vercel for the docs site, and dated
`RELEASE_v*.md` files for versioned change logs.

## Workflows in `.github/workflows/`

| Workflow | When it runs | What it does |
|---|---|---|
| `tests.yml` | push to `main`, PRs | pytest (unit + e2e) with pytest-xdist parallelism |
| `docker-publish.yml` | push to `main`, PRs, release published | build multi-arch image, smoke test, push to Docker Hub |
| `nix.yml` | changes to Nix files / pyproject.toml | `nix flake check` on Linux + macOS, build package |
| `supply-chain-audit.yml` | PRs | scan diff for suspicious patterns, comment findings |
| `deploy-site.yml` | release published, push to `main` (`website/**`), manual | build Docusaurus, deploy GitHub Pages + POST Vercel hook |
| `skills-index.yml` | cron (6am & 6pm UTC) + manual | rebuild `website/static/api/skills-index.json` |
| `contributor-check.yml` | PRs | verify contributor attribution |
| `docs-site-checks.yml` | PRs to `website/` | lint + build |

## tests.yml

Two jobs:

- **test** — `pytest tests/ -q --ignore=tests/integration --ignore=tests/e2e -n auto`
  - Uses `uv` + Python 3.11
  - Clears provider env vars before run to catch network-reliant tests
  - Skips integration tests by default (`@pytest.mark.integration`)
- **e2e** — `pytest tests/e2e/ -v --tb=short`

Running the same locally:

```bash
pytest tests/ -q            # skip integration, parallel
pytest tests/ -m integration    # only integration (needs API keys)
pytest tests/e2e/ -v            # e2e
```

## docker-publish.yml

1. Checkout with submodules
2. Setup QEMU + Docker Buildx
3. Build amd64 for smoke test, load to local daemon
4. Smoke test: `docker run <image> --help`
5. Login with `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets
6. Push multi-arch (amd64 + arm64) as `nousresearch/hermes-agent:latest`
7. On tag: also push `nousresearch/hermes-agent:<tag>`

60-minute timeout on the build job.

## supply-chain-audit.yml

Scans PR diffs for:

- `.pth` files
- `base64` + `exec` combos
- `subprocess` with encoded commands
- Network calls in new code
- `setup.py` install hooks
- `marshal` / `pickle` of user-controlled data
- Workflow file changes (extra scrutiny)
- Dockerfile changes
- Dependency manifests
- Unpinned GitHub Actions

Findings post as a PR comment. **CRITICAL** findings fail the job. See
[[17-Security-Model]].

## deploy-site.yml

- Triggered on release publish or `website/**` pushes
- Two jobs:
  - **deploy-vercel**: POST to `VERCEL_DEPLOY_HOOK` secret (on release)
  - **deploy-docs**: Extract skill metadata (`website/scripts/extract-skills.py`),
    build with `npm run build`, deploy to GitHub Pages

## skills-index.yml

Runs `scripts/build_skills_index.py` which scans `skills/` and
`optional-skills/` for `SKILL.md` front-matter, emits
`website/static/api/skills-index.json` for Skills Hub to consume.
Scheduled twice daily + manual.

## Local pre-submit

```bash
pytest tests/ -q                        # unit tests
ruff check . && ruff format --check .   # style (if ruff is configured)
python scripts/build_skills_index.py    # skill manifest sanity
```

See [`CONTRIBUTING.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/CONTRIBUTING.md)
for the full list.

## Release cadence

Releases are numbered `v0.X.0` and tagged on `main`. Each has:

- A `RELEASE_vX.Y.Z.md` file at the repo root
- A GitHub release with the same text
- Docker Hub tags (`latest` + `X.Y.Z`)
- Docs redeploy via `deploy-site.yml`

Release automation lives in
[`scripts/release.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/scripts/release.py).

Typical cadence: every 2–4 weeks. Latest is v0.10.0 (April 16, 2026),
called the "Tool Gateway" release.

## Post-release checklist

1. `RELEASE_v*.md` file committed and merged
2. `pyproject.toml` version bump
3. Tag: `git tag v0.X.0 && git push --tags`
4. GitHub release created (triggers Docker + docs deploys)
5. Check Docker Hub for multi-arch image
6. Check [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/docs/)
   for docs refresh

## Secrets expected in CI

- `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`
- `VERCEL_DEPLOY_HOOK`
- `GITHUB_TOKEN` (provided by GitHub Actions)

## Source of truth

- [`.github/workflows/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/.github/workflows)
- [`scripts/release.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/scripts/release.py)
- [`CONTRIBUTING.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/CONTRIBUTING.md)
- `RELEASE_v*.md` files at the repo root

## See also

- [[17-Security-Model]]
- [[18-Deployment]]
- [[22-Contributing]]
