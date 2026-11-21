# python-app-template

[![CI](https://github.com/mbologna/python-app-template/actions/workflows/ci.yml/badge.svg)](https://github.com/mbologna/python-app-template/actions/workflows/ci.yml)

A [Copier](https://copier.readthedocs.io/) template for a small Python
application shipped as a container image: a CronJob, a bot or a small web
service. Every such repository gets the same skeleton, and a fix to the
skeleton reaches all of them as an ordinary pull request. The questions
below cover what those repositories need to differ in, so their files can
stay as the template renders them. A Python project that ships no image (a
library, a tool, a generator that CI runs) uses the same skeleton in
[package mode](#package-mode).

## What you get

| | |
|---|---|
| Layout | `src/<package>/` built with `uv_build`, console scripts in `[project.scripts]`, `requires-python` matching the image's Python |
| Lint and tests | ruff (lint and format), pytest with pytest-cov and `[tool.coverage.report] fail_under` (80 by default), mypy and bandit in pre-commit |
| Image | Two stages. A hash-pinned uv (`requirements-uv.txt`) runs `uv sync --locked --no-dev` into `/opt/venv`, dependencies in a layer of their own. The runtime stage is the same `python:<version>-slim` or `-alpine` digest without pip or ensurepip, with the distribution's security updates applied at build time, running as a non-root UID with OCI labels and a read-only root filesystem |
| Dependencies | `uv.lock`, a 7-day `exclude-newer` cooldown, `renovate.json` extending [`mbologna/.github`](https://github.com/mbologna/.github)'s shared preset |
| Notifications | Optional: [apprise-notify-py](https://github.com/mbologna/apprise-notify-py), with `APPRISE_URL` holding one Apprise API URL or several space-separated ones tried in order |
| CI | GitHub: callers of `mbologna/.github`'s `python-ci` and `docker-ci`, pinned by SHA, with a gating smoke test and one of three [release modes](#release-modes), with an optional Cloud Run deploy when every push publishes. GitLab: `base`, `container-app` (commit check, hadolint, kaniko-build and auto-release) and `python-test` from [ci-templates](https://gitlab.com/mbologna/ci-templates), all at one version |
| Repository files | `AGENTS.md`, `CLAUDE.md`, `README.md`, `.pre-commit-config.yaml`, `.dockerignore`, `.gitignore`, `LICENSE` (MIT, Apache-2.0 or GPL-3.0-only) |

## New project

```sh
uvx copier copy https://github.com/mbologna/python-app-template.git my-app
cd my-app && uv lock
```

Copier asks the questions below; every one has a default. The template
runs no tasks, so it needs no `--trust`, and `uv lock` creates the lock
file.

## Questions

Answers are recorded in `.copier-answers.yml`. Lists are written as words
separated by spaces, for example `-d 'runtime_packages=tzdata curl'`.

### The project

| Question | Default | Effect |
|---|---|---|
| `project_name` | | Distribution, console script and image name |
| `package_name` | `project_name` with underscores | Import name under `src/` |
| `description` | | `pyproject.toml`, README and the image's description label |
| `kind` | `app` | `app`: an application shipped as a container image. `package`: the same project without the image; see [Package mode](#package-mode) |
| `python_floor` | `3.15` | With `kind: package`: the oldest Python the package supports. See [Package mode](#package-mode) |
| `author` | Michele Bologna | `pyproject.toml` and the copyright line |
| `license` | `MIT` | `MIT`, `Apache-2.0` and `GPL-3.0-only` write `LICENSE`, `license` in `pyproject.toml` and the image's licences label. `other` takes `license_expression`. `none` writes no licence file or metadata |
| `license_expression` | | With `license: other`: an SPDX expression such as `MIT AND GPL-3.0-only`. The project provides `LICENSE` and the other texts in `LICENSES/`; both go into the wheel and the image |
| `forge` | `github` | Picks the CI and the image registry (GHCR or the GitLab project registry) |
| `notifications` | yes | Adds apprise-notify and `APPRISE_URL` |
| `sample_code` | yes | Generates the example package and its tests; answer no when adopting the template |
| `main_module` | yes | The package has a `__main__.py`; coverage leaves that file out |
| `coverage_fail_under` | 80 | The coverage gate |

### The image

`kind: package` skips every question in this table.

| Question | Default | Effect |
|---|---|---|
| `base_variant` | `slim` | `slim` (Debian, glibc, takes every wheel) or `alpine` (musl, smaller) |
| `entrypoint` | `entrypoint` | How the image starts; see [Entry points](#entry-points) |
| `script_target` | `<package>:main` | The function behind the `project_name` console script, as `module:function`. A package asks it too, and an empty answer renders no console script |
| `image_uid` | 1000 | `USER uid:gid`, with the GID equal to the UID |
| `image_user` | empty | Creates a named account for that UID, for software that looks the UID up in `/etc/passwd`. Empty runs as the bare UID |
| `runtime_packages` | empty | Distribution packages installed in the runtime stage, in the same layer as the security updates, with apt or apk as the base needs and the hadolint ignore for unpinned packages |
| `build_packages` | empty | Packages installed in the build stage only, such as a compiler for a dependency without a wheel. The runtime stage takes nothing but `/opt/venv` |
| `runtime_env` | empty | Extra `NAME=value` pairs for an `ENV` line after the template's own. `$PATH` expands, so `PATH=/opt/tool/bin:$PATH` works |
| `writable_dirs` | empty | Absolute paths created and owned by the image's UID, so a new volume mounted there starts writable |
| `port` | 0 | A web service's port: sets `ENV PORT` and `EXPOSE`. Cloud Run and most runtimes pass `PORT`; 0 means the app serves nothing |

### CI and delivery

| Question | Default | Effect |
|---|---|---|
| `releases` | yes | GitHub app. Yes: image releases, where only `vX.Y.Z` tags publish the image. No: every push to main publishes it. See [Release modes](#release-modes) |
| `changelog_releases` | no | GitHub app with `releases: false`, or any package: auto-release tags `vX.Y.Z` weekly with a GitHub or GitLab Release, as a changelog. See [Release modes](#release-modes) and [Package mode](#package-mode) |
| `pypi_publish` | no | GitHub package with `changelog_releases`: each tag is published to PyPI. See [Package mode](#package-mode) |
| `cloud_run` | `none` | GitHub app with `releases: false`: `service` or `job` deploys each image published from main to Cloud Run (docker-ci's `deploy_type`) |
| `codecov` | no | GitHub: uploads the coverage report to Codecov |
| `smoke_cmd` | GitHub: `<project_name> --help`; GitLab: `auto` | Apps only. GitHub: a shell command docker-ci runs in the image (entry point overridden, no network) that must exit 0. GitLab: container-app's `smoke_cmd`, which is `auto` (import every installed package), `none`, or the arguments to run the image with |
| `extra_images` | empty | GitLab app: subdirectories with their own `Dockerfile`. Each builds `$CI_REGISTRY_IMAGE/<subdirectory>` through container-app's `images` input. See [Several images](#several-images) |
| `changed_images_only` | yes | GitLab app, with `extra_images`: container-app's `changed_images_only`. Yes: a release rebuilds only the images whose files changed. No: every release rebuilds every image. See [Several images](#several-images) |
| `owned_files` | none | Files the project owns; see [Owning a file](#owning-a-file) |

Docker Hub pushes are off in every generated project.

Every question added since v1.1.3 defaults to what v1.1.3 rendered: an
update that answers none of them brings only the release's fixes.

## Release modes

On GitHub, `releases` and `changelog_releases` pick one of three modes.
The two that tag use auto-release's `conventional` strategy, which tags on
Monday's schedule when a `feat`, `fix`, `perf`, `revert` or breaking commit
landed or dependency updates wait for a maintenance release, and on a push
to main only for a CVE fix.

| Mode | Answers | Tags and Releases | Images | Cloud Run |
|---|---|---|---|---|
| Image releases | `releases: true` (the default) | docker-ci's `release_strategy` tags `vX.Y.Z` and dispatches the tag's build | Only tags publish, as `X.Y.Z`, `X.Y` and `latest`; the registry cleanup keeps every version | No |
| Changelog | `releases: false`, `changelog_releases: true` | A separate job calls `auto-release.yml` with `dispatch_tag_build: false`: the tag and its Release record what main already shipped, and build nothing | Every push to main publishes `latest` and `sha-<commit>` | Optional |
| None | `releases: false` | None: no tags, no schedule, no `contents: write` | Every push to main publishes `latest` and `sha-<commit>` | Optional |

The changelog mode suits an app deployed per commit that still wants weekly
release notes. Its docker job keeps `contents: read`, the workflow has no
`tags:` trigger, and auto-release gets no `image` input: with no `X.Y.Z`
image to scan, it never cuts a fresh-rebuild release, while its base image
check still reads `Dockerfile`. On GitLab, ci-templates' `container-app`
builds images only from release tags, so an app there has only image
releases and both questions are skipped. A package has no image, and
`changelog_releases` alone picks its mode on either forge.

When every push publishes, the `sha-*` images are the rollback window, and
a Cloud Run deploy points the service or job at the pushed image's digest.

## Package mode

`kind: package` renders the project without an image, for a library, a
command-line tool or a generator that CI runs. It keeps the `src/` layout,
uv, ruff, pytest with the coverage gate, pre-commit and Renovate. It drops
`Dockerfile`, `.dockerignore`, every question under
[The image](#the-image) except `script_target`, and `smoke_cmd`,
`cloud_run`, `extra_images` and `changed_images_only`. `README.md` and
`AGENTS.md` describe a package, with no image, UID or entry point.

| Question | Default | Effect |
|---|---|---|
| `python_floor` | `3.15` | The oldest Python the package supports, from 3.10 to 3.15: `requires-python = ">=<floor>"` and ruff's `target-version`. CI still tests on Python 3.15 only |
| `script_target` | `<package>:main` | The `project_name` console script, as for an app. Empty renders no `[project.scripts]`, for a library |
| `changelog_releases` | no | Yes: auto-release tags `vX.Y.Z` on Monday's schedule with a GitHub or GitLab Release of the commit subjects; a tag builds nothing. No: no tags and no releases |
| `pypi_publish` | no | GitHub, with `changelog_releases`: adds `.github/workflows/publish.yml`, which builds each tag with the tag's version and uploads it to PyPI with trusted publishing |

What the CI file holds:

- GitHub: `ci.yml` calls `python-ci` and, with `changelog_releases`,
  `auto-release.yml` with `dispatch_tag_build: false`. With `pypi_publish`
  the auto-release job also gets `actions: write` and
  `dispatch_workflow: publish.yml`, so the new tag starts `publish.yml`.
  There is no `docker-ci` job and no `requirements-uv.txt`.
- GitLab: `.gitlab-ci.yml` includes the `base`, `commits` and `python-test`
  components and, with `changelog_releases`, `auto-release`, all at the
  ci-templates version an app uses. The `test` job runs on the
  `python:<version>-slim` image. `requirements-uv.txt` stays, because
  python-test installs uv from it.

PyPI's trusted publisher names the repository, the workflow file and the
GitHub environment, so `publish.yml` keeps its name and its publish job
runs in the `pypi` environment. Add the publisher on PyPI before the first
release. `version` in `pyproject.toml` stays `0.0.0`: `publish.yml` sets
the tag's version before it builds.

A package whose CI does more than test and release, such as building a site
and deploying it to GitHub Pages, owns its CI file: `owned_files: [ci]`
writes `ci.yml` or `.gitlab-ci.yml` on the first copy as a starting point,
and updates leave it alone while the rest of the skeleton keeps arriving.
One or two extra jobs at the end of the file can instead stay a local
change, which survives updates (see [Owning a file](#owning-a-file)).

A tool that hosts also run as a plain script, downloaded from a release
tag and started with `python3`, keeps its code in
`src/<package>/__init__.py`, imports only the standard library, and sets
`python_floor` to the oldest Python those hosts run.

### Not a fit

A uv workspace with several packages and several images needs a layout
and a pipeline per member, which neither mode renders. Keep such a
repository's skeleton by hand.

## Several images

On GitLab, `extra_images` adds an image for each subdirectory, and
`changed_images_only` decides what a release rebuilds.

With `changed_images_only: true`, the default, a release rebuilds only the
images whose files changed since the previous release; the others get the
new version's tags on their previous digest, with no build. That saves CI
minutes and registry space. The cost is security updates. The runtime stage
applies the distribution's updates at build time, and auto-release cuts a
fresh-rebuild release (a patch release when the last image has fixable
findings in the packages its Dockerfile installs) only for a project with
one Dockerfile. With several, nothing ever rebuilds an image whose files
don't change, so it keeps the packages of its last build.

With `changed_images_only: false`, every release rebuilds every image, and
each one picks up the current security updates. Releases take longer and
store a new copy of the layers that changed upstream. Choose it when an
extra image installs distribution packages and its own files rarely
change.

The default renders no input, so a project that answers nothing gets the
`.gitlab-ci.yml` it had before the question existed.

## Entry points

`entrypoint` follows one rule. An image with one program makes it the
`ENTRYPOINT` (`entrypoint`, the default): its flags go in the container's
arguments, `docker run <image> --dry-run` or Kubernetes `args:`. An image
with several programs sets `CMD` to the default one (`cmd`) or sets none
(`none`), and a deployment picks the program with its `command:` or with
`docker run <image> <script>`. With `none`, the template adds no console
script of its own (the sample code's excepted), and the project lists its
scripts in `[project.scripts]`. docker-ci's smoke test overrides the
entry point with `sh`, so `smoke_cmd` works with all three.

## Owning a file

The questions cover the common differences. Two mechanisms handle the rest.

A local change to a template file survives updates. `copier update` diffs
the project against the template version it came from and replays that
diff onto the new version, so a line the template doesn't touch stays as
the project wrote it. Keep such changes where the template rarely changes:
extra build stages above the first `FROM`, `COPY --from=` lines after
`LABEL`, extra inputs after the ones the template sets. A conflict appears
only when the template changes the lines next to the local change, and
Copier leaves it inline to resolve.

`owned_files` hands a file to the project outright: Copier writes it on the
first copy and never again. The choices are `Dockerfile`, `ci` (the forge's
CI file), `.dockerignore`, `.gitignore` and `.pre-commit-config.yaml`; a
package, which has no `Dockerfile` or `.dockerignore`, can't choose those
two.
`README.md`, `AGENTS.md`, `CLAUDE.md` and `LICENSE` are always owned.

The cost of owning a file is that template fixes to it stop arriving: a new
base image digest, a security change to the build, a CI input. Own a file
when its local part is larger than the template's, such as a Dockerfile
built around a downloaded application or a pipeline with its own stages,
and port template changes by hand from the release notes. Otherwise keep
the local change; a few lines of conflict now and then are cheaper than
drifting from every fix.

## Adopting it in an existing repository

```sh
cd existing-app
uvx copier copy --overwrite --defaults -d sample_code=false \
  -d project_name=existing-app -d 'description=What it does' -d forge=gitlab \
  https://github.com/mbologna/python-app-template.git .
git diff
```

Add the answers that describe the repository (its UID, packages, entry
point, licence), then put back what the template overwrote and no question
covers: its dependencies and console scripts in `pyproject.toml`, extra
build stages, extra CI inputs. Keep the repository's lock file and lock
again, which keeps every existing pin:

```sh
git checkout HEAD -- uv.lock && uv lock
```

Move the code to `src/<package>/` if it isn't there yet, and commit
`.copier-answers.yml`: updates read it. `README.md`, `AGENTS.md`,
`CLAUDE.md` and `LICENSE` are never overwritten when they exist; delete a
`LICENSE` the repository should not have, and stop ignoring `CLAUDE.md` in
`.gitignore` if it is.

The template's `requires-python` is the image's Python, which ruff also
takes as its target version (a package takes `python_floor` for both). Moving an older repository to it can reformat
code: from Python 3.14, `ruff format` writes `except (A, B):` as
`except A, B:`.

## Updates

Renovate's `copier` manager reads `.copier-answers.yml` and opens a pull
request when this template tags a new version. By hand:

```sh
uvx copier update --defaults && uv lock
```

To change an answer, pass it to the update:

```sh
uvx copier update --defaults -d runtime_packages=tzdata -d image_uid=65532
```

An answer that replaces lines the project had edited for the same purpose
conflicts once; resolve it in favour of the template's version, and remove
the project's own copy of what the answer now generates. Other conflicts
are left inline; resolve them in favour of the project's code. Copier
deletes a file the new answers no longer render, even one the project has
changed: `license: none` removes `LICENSE`. With `license: other` the
template still renders a placeholder `LICENSE`, which never overwrites the
project's own.

## Versioning

A generated project keeps `version = "0.0.0"` in `pyproject.toml`. With
image releases, its releases are SemVer git tags `vX.Y.Z` that
[auto-release](https://github.com/mbologna/auto-release) creates on the
weekly schedule from the Conventional Commit types (`feat` minor; `fix`,
`perf`, `revert` patch; a breaking change major); only a tag builds and
pushes the image, as `X.Y.Z`, `X.Y` and `latest`, and the pipeline sets the
image's `org.opencontainers.image.version` label from that tag. In the
changelog mode the tags are release notes only, and each push to main
publishes `latest` and `sha-<commit>`, as it does with no releases at all.
A package's tags build nothing, unless `pypi_publish` builds each one for
PyPI with the tag's version. Nothing needs the version inside the package,
so nothing has to be bumped.

This template is versioned the same way: Monday's CI run tags `vX.Y.Z`
with a GitHub Release when a `feat`, `fix`, `perf`, `revert` or breaking
commit landed. Nothing builds on a tag; the tag is the version Renovate's
copier manager offers generated projects. See the
[versioning standard](https://github.com/mbologna/.github/blob/main/VERSIONING.md).

## Development

CI renders the template in ten answer combinations. Seven are apps,
covering both forges, the three release modes, both base images, the three
entry-point styles, every licence kind, the image questions and both
`changed_images_only` settings. Three are packages: GitHub without
releases, a GitHub library published to PyPI with a 3.10 floor, and GitLab
with changelog releases. Each result runs ruff, its tests with the coverage
gate, pre-commit and, on GitHub, actionlint. An app also runs hadolint, an
image build, a smoke test on a read-only root filesystem and checks of the
image's user, exposed port and the absence of pip, uv and git. A package
is checked for the files it must not have, and for a `README.md` and
`AGENTS.md` free of image terms. A second job copies a project from the latest release,
updates it to the commit under test and checks that the result equals a
fresh copy. Locally:

```sh
uvx copier copy --defaults --vcs-ref HEAD -d project_name=demo-app -d description=Demo . /tmp/demo-app
cd /tmp/demo-app && uv lock && uv run pytest --cov && docker build -t demo-app . && docker run --rm demo-app --help
```

With uncommitted changes, `--vcs-ref HEAD` renders the working tree.
Renovate keeps the pins under `template/` current: `renovate.json` points
its managers at the `.jinja` files.
