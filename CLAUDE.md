# CLAUDE.md

Instructions for AI agents and contributors working on this repo.

## Wave rollout convention

Diagram: [docs/pipeline.md](docs/pipeline.md) — source of truth for the
gating shape described below.

Every tier (`db`, `app`, `web`) in every environment (`dev`, `preprod`,
`prod`) deploys in 3 sequential waves: `canary` → `wave2` → `final`.

- Wave = named inventory group `<env>_<tier>_<wave>`, e.g. `dev_app_canary`.
- CI job = `<env>:<tier>:<wave>`, e.g. `dev:app:canary`.
- Job targets its wave via `--limit <env>_<tier>_<wave>`. Never `--limit
  <env>_<tier>` directly in a wave job — that bypasses the wave gate.

### Gating rules

Every deploy job — all 27 of `<env>:<tier>:<wave>` — is `when: manual`,
in every environment including dev. Nothing after `lint` runs without
an explicit human click. Do not make any wave or tier auto-run "for
convenience": a human must approve each of
canary→wave2→final within a tier, and each db→app→web tier handoff
within an environment, and each dev→preprod→prod promotion. `lint` is
the only job that runs automatically, on every push to `main`.

### Adding a new environment or tier

1. Add `<env>_<tier>_canary` / `_wave2` / `_final` children groups under
   `<env>/<tier>` in `inventory/<env>/hosts.yml`, nested under the parent
   `<tier>` group (see existing files for the nesting shape). Declare
   each host exactly once, under the wave group it belongs to — do not
   list a host under both the flat tier group and a wave group; group
   nesting gives it membership in both automatically.
2. Add 3 job stanzas (`<env>:<tier>:canary/wave2/final`) to
   `.gitlab-ci.yml`, each `extends: .deploy-template`, following the
   `needs:` chain pattern of the existing jobs.
3. Do not add a 4th wave. If more granularity is needed, subdivide
   `wave2` into named subgroups — do not add wave stages without
   updating this file's gating rules.

### Empty wave groups

A wave group with zero hosts is valid and expected (e.g. a tier too
small to split into 3 batches). `.deploy-template`'s script detects
this via `ansible-inventory --list` and exits 0 with a log line,
without invoking `ansible-playbook`. Do not remove this check — an
empty `--limit` target makes `ansible-playbook` error out.

### Why not `serial:`

Ansible's `serial:` keyword also does batched rollout, but it runs all
batches in one playbook invocation with no pause for human review
between batches. That defeats the wave gating requirement above. Do
not replace the CI-job-per-wave structure with `serial:` batching
without discussing the tradeoff first.

### Why not `parallel: matrix`

Matrix-generated jobs can't cleanly express "job B needs this specific
instance of job A, in order" for a chain of 3+ steps. Named job stanzas
per wave, sharing one `.deploy-template`, is the DRY mechanism here —
the shared logic lives once in the template; only 4 lines
(`stage`/`needs`/`rules`/`variables`) repeat per job.

## Roles

`roles/db/`, `roles/app/`, `roles/web/` are placeholder skeletons
(single `debug` task each). This is a stack-agnostic template. Replace
placeholder tasks with real ones only once the actual technology for
that tier is decided — do not guess a stack.

## group_vars

Each `inventory/<env>/group_vars/<env>_<tier>.yml` is the single source
of config for that tier in that environment. Do not add `vars:` blocks
back into `hosts.yml` — `group_vars/` is the source of truth (see git
history for why this was consolidated).

## Secrets

`SSH_PRIVATE_KEY` is a required masked/protected GitLab CI/CD variable
(not committed). Any future secret a role needs (db passwords, API
keys, certs) must go through Ansible Vault or CI/CD variables — never
committed in plaintext to `group_vars/`.
