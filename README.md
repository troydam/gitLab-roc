# gitLab-roc

Ansible-driven, GitLab CI-orchestrated deployment pipeline for a three-tier
application (db → app → web), promoted through three environments
(dev → preprod → prod).

## Folder structure

```
.
├── .gitlab-ci.yml
├── site.yml
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
| `.gitlab-ci.yml` | GitLab CI config | Defines the `dev` → `preprod` → `prod` pipeline stages and the `db` → `app` → `web` job ordering within each, via a shared `.deploy-template` and `needs:`. Gated to the `main` branch by a top-level `workflow: rules`. Preprod and prod `db` jobs are `when: manual` (human promotion gate). The template runs on the `cytopia/ansible` image and loads an SSH key from the `SSH_PRIVATE_KEY` CI/CD variable (masked, project settings) before invoking Ansible. |
| `site.yml` | Ansible playbook | Single entry point run by every CI job. Three plays, one per tier (`hosts: db`, `hosts: app`, `hosts: web`), each applying the matching role. The CI job scopes which hosts run via `--limit <env>_<tier>` (a subgroup of the play's `hosts:` group). |
| `roles/db/`, `roles/app/`, `roles/web/` | Ansible roles | Standard-shape role skeletons (`tasks/`, `defaults/`, `handlers/`, `meta/`). This repo is a generic pipeline template not tied to a specific stack, so each role's `tasks/main.yml` is currently a placeholder `debug` task — swap in real tasks (install engine/runtime/web server, deploy config) once the actual technology for that tier is decided. Each role's tasks comment references the `group_vars` already available to it. |
| `inventory/<env>/hosts.yml` | Ansible inventory | Static host list for one environment. Each tier is a parent group (`db`, `app`, `web`) — matching `site.yml`'s `hosts:` — with a `children:` subgroup named `<env>_db`/`<env>_app`/`<env>_web` holding the actual hosts. This is the standard Ansible pattern for "same role, many environments": `site.yml` targets the parent group, `--limit` narrows to the environment-specific child. Hosts are plain DNS names — no per-host vars here; all tier/environment config lives in `group_vars/`. |
| `inventory/<env>/group_vars/<env>_db.yml` | Ansible group vars | Database-tier config for that environment: `db_port`, `db_name`, `db_user`, `db_max_connections`, `postgres_version` (prod also sets `db_replication_enabled`). |
| `inventory/<env>/group_vars/<env>_app.yml` | Ansible group vars | App-tier config: `app_port`, `app_workers`, `app_env`, and `db_host` (resolved from the environment's `_db` group so app servers know which db to talk to). |
| `inventory/<env>/group_vars/<env>_web.yml` | Ansible group vars | Web-tier config: `web_port`, `server_name`, and `app_upstream_hosts`/`app_upstream_port` (resolved from the environment's `_app` group for reverse-proxy upstream config). |

Every `group_vars/*.yml` also sets `env: <dev|preprod|prod>` and `tier:
<db|app|web>` for use in templates/roles.

## Environments

| Env | Branch trigger | Promotion | Notes |
|---|---|---|---|
| dev | `main` | automatic | Runs first in the pipeline. |
| preprod | `main` | manual (`preprod:db`) | Gated behind all of `dev` completing (`needs: dev:web`). |
| prod | `main` | manual (`prod:db`) | Gated behind all of `preprod` completing (`needs: preprod:web`). |

## Pipeline flow

```
dev:db → dev:app → dev:web
              │
              ▼ (needs, manual gate)
preprod:db → preprod:app → preprod:web
              │
              ▼ (needs, manual gate)
prod:db → prod:app → prod:web
```

Each job runs:

```
ansible-playbook -i inventory/${ENVIRONMENT}/hosts.yml site.yml \
  --limit ${ENVIRONMENT}_${PHASE}
```

## Requirements to run for real

- A GitLab CI/CD variable named `SSH_PRIVATE_KEY` (masked, protected)
  holding the private key Ansible uses to SSH into target hosts. The
  corresponding public key must be authorized on every host listed in
  `inventory/<env>/hosts.yml`.
- Those hosts must already exist and be reachable from the GitLab
  Runner — this repo does not provision infrastructure, only configures
  hosts that already exist.

## Known gaps

- `roles/db/`, `roles/app/`, `roles/web/` currently only contain a
  placeholder `debug` task — this is a generic pipeline template, so
  real tasks (install engine/runtime/web server, deploy config) need to
  be added once the actual technology stack is decided.
- No `ansible.cfg` or `requirements.yml` is present.
