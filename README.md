# gitLab-roc

Ansible-driven, GitLab CI-orchestrated deployment pipeline for a three-tier
application (db → app → web), promoted through three environments
(dev → preprod → prod).

Diagram: [docs/pipeline.md](docs/pipeline.md) (source of truth) — nicer
static view: [docs/wave-rollout-pipeline.html](docs/wave-rollout-pipeline.html)

## Folder structure

```
.
├── .gitlab-ci.yml
├── CLAUDE.md
├── ansible.cfg
├── .yamllint
├── site.yml
├── docs/
│   └── pipeline.md
├── roles/
│   ├── db/
│   │   ├── tasks/main.yml
│   │   ├── defaults/main.yml
│   │   ├── handlers/main.yml
│   │   └── meta/main.yml
│   ├── app/
│   │   └── ... (same layout as db/)
│   └── web/
│       └── ... (same layout as db/)
└── inventory/
    ├── dev/
    │   ├── hosts.yml
    │   └── group_vars/
    │       ├── dev_db.yml
    │       ├── dev_app.yml
    │       └── dev_web.yml
    ├── preprod/
    │   ├── hosts.yml
    │   └── group_vars/
    │       ├── preprod_db.yml
    │       ├── preprod_app.yml
    │       └── preprod_web.yml
    └── prod/
        ├── hosts.yml
        └── group_vars/
            ├── prod_db.yml
            ├── prod_app.yml
            └── prod_web.yml
```

## File reference

| Path | Type | Purpose |
|---|---|---|
| `docs/pipeline.md` | Diagram | Mermaid diagram of the env/tier flow and wave-gating pattern (renders natively on GitLab/GitHub). Source of truth for the shape described in `CLAUDE.md`. |
| `docs/wave-rollout-pipeline.html` | Diagram (generated) | Static styled render of the same diagram, for a nicer view than plain Mermaid. Not the source of truth — regenerate from `docs/pipeline.md` after any change to it, don't hand-edit. |
| `.gitlab-ci.yml` | GitLab CI config | Defines a `lint` stage followed by `dev` → `preprod` → `prod`. Within each environment: `db` → `app` → `web` tier ordering, and within each tier: `canary` → `wave2` → `final` rollout waves, all via a shared `.deploy-template` and `needs:`. See `CLAUDE.md` for the full wave-rollout convention. Gated to the `main` branch by a top-level `workflow: rules`. The deploy template runs on the `cytopia/ansible` image and loads an SSH key from the `SSH_PRIVATE_KEY` CI/CD variable (masked, project settings) before invoking Ansible. |
| `ansible.cfg` | Ansible config | Repo-local Ansible defaults: inventory path, `host_key_checking`, disables retry-file clutter, `yaml` stdout for readable CI logs, connection `pipelining` for speed. Keeps behavior consistent across whatever runner/image executes the job, instead of relying on image defaults. |
| `.yamllint` | Lint config | Rules for the `lint` CI job's `yamllint` pass — relaxes line-length to 120 and accepts `true`/`false` as the only truthy spellings. |
| `site.yml` | Ansible playbook | Single entry point run by every CI job. Three plays, one per tier (`hosts: db`, `hosts: app`, `hosts: web`), each applying the matching role. The CI job scopes which hosts run via `--limit <env>_<tier>` (a subgroup of the play's `hosts:` group). |
| `roles/db/`, `roles/app/`, `roles/web/` | Ansible roles | Standard-shape role skeletons (`tasks/`, `defaults/`, `handlers/`, `meta/`). This repo is a generic pipeline template not tied to a specific stack, so each role's `tasks/main.yml` is currently a placeholder `debug` task — swap in real tasks (install engine/runtime/web server, deploy config) once the actual technology for that tier is decided. Each role's tasks comment references the `group_vars` already available to it. |
| `inventory/<env>/hosts.yml` | Ansible inventory | Static host list for one environment. Each tier is a parent group (`db`, `app`, `web`) — matching `site.yml`'s `hosts:` — nesting down to `<env>_db`/`<env>_app`/`<env>_web`, which in turn nests to three wave groups: `<env>_<tier>_canary`, `_wave2`, `_final`. Each host is declared exactly once, under whichever wave group it's assigned to; group nesting (not duplication) gives it membership in every parent group. CI targets a wave group directly via `--limit`. Hosts are plain DNS names — no per-host vars here; all tier/environment config lives in `group_vars/`. |
| `inventory/<env>/group_vars/<env>_db.yml` | Ansible group vars | Database-tier config for that environment: `db_port`, `db_name`, `db_user`, `db_max_connections`, `postgres_version` (prod also sets `db_replication_enabled`). |
| `inventory/<env>/group_vars/<env>_app.yml` | Ansible group vars | App-tier config: `app_port`, `app_workers`, `app_env`, and `db_host` (resolved from the environment's `_db` group so app servers know which db to talk to). |
| `inventory/<env>/group_vars/<env>_web.yml` | Ansible group vars | Web-tier config: `web_port`, `server_name`, and `app_upstream_hosts`/`app_upstream_port` (resolved from the environment's `_app` group for reverse-proxy upstream config). |

Every `group_vars/*.yml` also sets `env: <dev|preprod|prod>` and `tier:
<db|app|web>` for use in templates/roles.

## Environments

| Env | Branch trigger | Promotion | Notes |
|---|---|---|---|
| dev | `main` | `db:canary` automatic | Runs first in the pipeline. |
| preprod | `main` | manual (`preprod:db:canary`) | Gated behind all of `dev` completing. |
| prod | `main` | manual (`prod:db:canary`) | Gated behind all of `preprod` completing. |

## Pipeline flow

Each tier rolls out in 3 waves — `canary` → `wave2` → `final`. A tier's
`canary` job auto-runs once the previous tier's `final` job passes (except
each environment's first job, `db:canary`, which is a manual promotion
gate). Every `wave2`/`final` job is `when: manual` — a human reviews the
prior wave before promoting further. See `CLAUDE.md` for the full
convention, including how empty wave groups are handled.

```
lint
  │
  ▼
dev:db:canary → dev:db:wave2 (manual) → dev:db:final (manual)
                                              │
                                              ▼
dev:app:canary → dev:app:wave2 (manual) → dev:app:final (manual)
                                              │
                                              ▼
dev:web:canary → dev:web:wave2 (manual) → dev:web:final (manual)
                                              │
                                              ▼ (manual: promote to preprod)
preprod:db:canary → ... → preprod:web:final
                                              │
                                              ▼ (manual: promote to prod)
prod:db:canary → ... → prod:web:final
```

Each job runs:

```
ansible-playbook -i inventory/${ENVIRONMENT}/hosts.yml site.yml \
  --limit ${ENVIRONMENT}_${PHASE}_${WAVE}
```

If `${ENVIRONMENT}_${PHASE}_${WAVE}` has no hosts, the job logs that and
exits 0 without invoking Ansible.

## Requirements to run for real

- A GitLab CI/CD variable named `SSH_PRIVATE_KEY` (masked, protected)
  holding the private key Ansible uses to SSH into target hosts. The
  corresponding public key must be authorized on every host listed in
  `inventory/<env>/hosts.yml`.
- Those hosts must already exist and be reachable from the GitLab
  Runner — this repo does not provision infrastructure, only configures
  hosts that already exist.

## Known gaps

Tracked, prioritized, in `CLAUDE.md` under **Shelf — deferred, revisit
when scope grows**.
