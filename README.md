# python-app-template

[![CI](https://github.com/mbologna/python-app-template/actions/workflows/ci.yml/badge.svg)](https://github.com/mbologna/python-app-template/actions/workflows/ci.yml)

A [Copier](https://copier.readthedocs.io/) template for a small Python
application shipped as a container image: a CronJob, a bot or a small web
service. It gives every such repository the same skeleton, so a fix to the
skeleton reaches all of them as an ordinary pull request.

## What you get

| | |
|---|---|
| Layout | `src/<package>/` built with `uv_build`, the console script in `[project.scripts]`, `requires-python` matching the image's Python |
| Lint and tests | ruff (lint and format), pytest with pytest-cov and `[tool.coverage.report] fail_under` (80 by default) |
| Image | Two stages: a hash-pinned uv (`requirements-uv.txt`) runs `uv sync --locked --no-dev` into `/opt/venv`; the runtime stage is the same `python:<version>-slim` or `-alpine` digest without pip or ensurepip, running as uid 1000, with OCI labels |
| Dependencies | `uv.lock`, a 7-day `exclude-newer` cooldown, `renovate.json` extending [`mbologna/.github`](https://github.com/mbologna/.github)'s shared preset |
| Notifications | Optional: [apprise-notify-py](https://github.com/mbologna/apprise-notify-py), with `APPRISE_URL` holding one Apprise API URL or several space-separated ones tried in order |
| CI | GitHub: callers of `mbologna/.github`'s `python-ci` and `docker-ci` (with a gating `smoke_cmd` and auto-release's `conventional` strategy), pinned by SHA. GitLab: `base`, `container-app` (which brings `conventional-title`, hadolint, kaniko-build and auto-release) and `python-test` from [ci-templates](https://gitlab.com/mbologna/ci-templates), all at one version |
| Repository files | `AGENTS.md`, `README.md`, `.pre-commit-config.yaml`, `.dockerignore`, `.gitignore`, `LICENSE` (MIT) |

## New project

```sh
uvx copier copy --trust https://github.com/mbologna/python-app-template.git my-app
```

The questions: the project and package names, a description, the forge
(GitHub or GitLab), the base image variant, whether the app sends
notifications, whether to generate the example package and tests, the
GitHub smoke test command, and the coverage threshold. `--trust` lets the
template run `uv lock` after rendering; without it, run `uv lock` yourself.

## Adopting it in an existing repository

```sh
cd existing-app
uvx copier copy --trust --overwrite -d sample_code=false \
  https://github.com/mbologna/python-app-template.git .
git diff
```

Then put the repository's own parts back where the template overwrote
them: its dependencies and console scripts in `pyproject.toml`, anything
extra its Dockerfile needs (system packages, a second binary), and its
CI inputs. `README.md`, `AGENTS.md` and `LICENSE` are never overwritten
when they exist. Move the code to `src/<package>/` if it isn't there yet,
and keep `.copier-answers.yml`: it is what updates read.

## Updates

Renovate's `copier` manager reads `.copier-answers.yml` and opens a pull
request when this template tags a new version. By hand:

```sh
uvx copier update --trust
```

Conflicts are left inline; resolve them in favour of the project's own
code.

## Versions

A generated project keeps `version = "0.0.0"` in `pyproject.toml`. Its
releases are git tags `vX.Y.Z` that auto-release creates on the weekly
schedule; only a tag builds and pushes the image, and the pipeline sets the
image's `org.opencontainers.image.version` label from that tag. Nothing
needs the version inside the package, so nothing has to be bumped.

This template is versioned the same way: Monday's CI run tags it when a
`feat`, `fix`, `perf`, `revert` or breaking commit landed.

## Development

CI renders the template in three combinations (GitHub and slim, GitLab and
alpine, GitHub without notifications) and runs each result's ruff, tests,
hadolint, image build and smoke test. Locally:

```sh
uvx copier copy --trust --defaults -d project_name=demo-app -d description=Demo . /tmp/demo-app
cd /tmp/demo-app && uv run pytest --cov && docker build -t demo-app . && docker run --rm demo-app --help
```

Renovate keeps the pins under `template/` current: `renovate.json` points
its managers at the `.jinja` files.
