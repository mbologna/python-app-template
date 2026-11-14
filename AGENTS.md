# AGENTS.md — python-app-template

A Copier template (`copier.yml`, files under `template/`). User-facing docs:
`README.md`.

- Files ending in `.jinja` are rendered; others are copied as they are.
  Conditional files and directories use Jinja in their names
  (`{% if forge == 'github' %}.github{% endif %}`).
- Public repository: keep examples generic. No names of private projects,
  hosts or the services they run.
- Every pin in `template/` must stay visible to Renovate: a new pinned file
  type needs a manager pattern or a regex manager in `renovate.json`.
- A change to the rendered output is a `feat` or `fix` (it releases a new
  template version, which Renovate offers every generated repository);
  changes to this repository's own CI or docs are `ci` or `docs`.
- Check a change by rendering it (README, "Development") in both forges.
