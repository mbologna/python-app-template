# AGENTS.md: python-app-template

A Copier template: `copier.yml` holds the questions, `template/` the files.
`README.md` documents every question and the ownership rules; update it
with any question you add or change.

## Files

- Files ending in `.jinja` are rendered; others are copied as they are.
  Conditional files and directories use Jinja in their names
  (`{% if forge == 'github' %}.github{% endif %}`). The three licence texts
  are one `LICENSE` each, named by a condition on `license`.
- `_skip_if_exists` lists the files a project owns after the first copy.
  `owned_files` adds to it through one templated entry per choice; an entry
  that renders empty matches nothing.
- No `_tasks`: a template with tasks needs `--trust`, and Renovate's copier
  manager runs without it.
- GitHub's three release modes come from two questions: `releases` (image
  releases) and, when it is false, `changelog_releases` (tags and GitHub
  Releases as a changelog, images per push). A template that branches on
  the mode tests `releases` first, then `changelog_releases`, so an answers
  file from before `changelog_releases` renders as it did. README, "Release
  modes".

## Rules for changes

- Existing projects must not change without opting in. A new question's
  default renders exactly what the template rendered before it, and an
  existing question's default never changes. Check it: render a few answer
  sets at the latest tag and at your commit and diff the two trees
  (README, "Development").
- A change to rendered output is a `feat` or `fix` (it releases a new
  template version, which Renovate offers every generated repository); a
  build-speed change to the Dockerfile is a `perf`. Changes to this
  repository's own CI or docs are `ci` or `docs`. A change that alters an
  existing project's output without an opt-in is breaking: `!` and a
  `BREAKING CHANGE:` footer, and avoid it.
- Keep the image's security properties: hash-pinned uv, no pip or
  ensurepip at run time, the distribution's security updates at build
  time, a non-root UID, nothing written outside `/tmp` and declared
  writable directories, OCI labels, and the 7-day `exclude-newer`.
- The entry point follows one rule: one program is the `ENTRYPOINT`;
  several programs get a default `CMD`, or none, and the deployment picks
  one (README, "Entry points").
- Every pin in `template/` must stay visible to Renovate: a new pinned
  file type needs a manager pattern or a regex manager in `renovate.json`.
- Public repository: keep examples generic. No names of private projects,
  hosts or the services they run.
- Watch Jinja whitespace: a `{%-` or `-%}` in the wrong place joins or
  splits lines, and the only check is rendering and diffing.
- A new question gets a row in the README's question tables and, when it
  changes the image or CI, a matrix entry in `.github/workflows/ci.yml`.

## Versioning

Plain SemVer: `vX.Y.Z` tags, each a GitHub Release and the version
Renovate's copier manager offers generated projects (nothing builds on a
tag), per the
[versioning standard](https://github.com/mbologna/.github/blob/main/VERSIONING.md).

- Never create, move or delete tags, and never edit a version number to cut
  a release: auto-release tags from the commit types (`feat` minor; `fix`,
  `perf`, `revert` patch; `!` or a `BREAKING CHANGE:` footer major).
- Write Conventional Commit subjects; they decide the next version. "Rules
  for changes" says which type a change to rendered output takes.
- A generated project's `README.md` and `AGENTS.md` carry a Versioning
  section for its release mode; keep them in step with `copier.yml` and the
  CI templates.
