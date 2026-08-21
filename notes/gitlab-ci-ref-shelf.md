# gitlab-ci-ref.yml review — shelf

Findings from reviewing `notes/.gitlab-ci-ref.yml` against enterprise
practice. Not our pipeline — tracked separately from `CLAUDE.md`'s
Shelf. Nothing here has been changed; review and prioritize before
acting on any item.

## Job categorization

| Lane | Jobs | Trigger | Gate pattern |
|---|---|---|---|
| Security scans | `secret_detection`, `sast` | every push | none |
| Cert/truststore reconcile | `dev-phase-1/2`, `preprod-phase-1/2` | `feature/integration` + `changes:` | canary auto, rest manual |
| Config/XML import | `xml-dev-phase-1/2`, `xml-preprod-phase-1/2` | `feature/integration` + `changes:` | canary auto, rest manual |
| Tomcat patch | `tomcat-dev-phase-1/2`, `tomcat-preprod-phase-1/2` | `feature/integration` (dev) / `develop` (preprod) | dev canary auto, dev-phase-2 manual, **preprod-phase-1 ungated**, preprod-phase-2 manual |
| Shared (hidden) | `.awx-runner`, `.xml-changed` | n/a | n/a |

3 parallel lanes reimplement the same dev→preprod, phase-1→phase-2
skeleton for 3 different AWX job templates.

## Critical

1. `TOMCAT_APPLY: "true"` and `CONFIG_APPLY: "true"` (lines 56, 58)
   contradict the file's own documented rule ("never hardcode ...
   true here"). Apply-mode is the default, not dry-run. Live
   safety-rail violation — highest priority fix.
2. `tomcat-preprod-phase-1` has its `dependencies:` and `when: manual`
   commented out (not removed), and triggers on `develop` instead of
   `feature/integration` like every other job. Ungated preprod deploy
   path.

## High

3. Secrets (`$CONTROLLER_TOKEN`) passed as CLI args — visible via
   `/proc/<pid>/cmdline` on shared runners, not just masked in logs.
4. `before_script` installs `git`/`pip3 install awxkit` unpinned, at
   runtime, on every job (8+ times/pipeline). No version pin, no
   baked-in image — reliability and supply-chain risk, especially in
   the IL5/DoD context the comments themselves call out.
5. Header comments promise Dependency Scanning + Container Scanning;
   `include:` only wires up SAST + Secret Detection. Dependency
   Scanning specifically matters given finding 4.

## Medium

6. `dependencies:` used where `needs:` is the correct keyword —
   `dependencies:` controls artifact download, not execution
   ordering; only "works" here because stage order happens to align.
7. No `resource_group:` on any deploy job — two concurrent pipeline
   runs could launch the same AWX job template against the same
   hosts at once.
8. No `environment:` declared on any job — GitLab Environments
   (deploy history, rollback UI, protected-environment approval
   rules) isn't wired up at all.
9. No production stage — stages stop at `itc-preprod-phase-2`. May be
   intentional (prod handled elsewhere) or an actual gap — confirm
   which.

## Low

10. `.xml-changed`'s `HEAD~1` fallback (when `CI_COMMIT_BEFORE_SHA` is
    unset) can diff against the wrong base on a new branch or after a
    force-push — silent under/over-selection of files to import.
11. `$CHANGED` (JSON array string) interpolated raw into a larger JSON
    literal for `--extra_vars` — no escaping if a filename ever
    contained a quote or backslash.
12. Trailing whitespace (line 79); no pipeline-lint stage to catch
    it or the `feature/integration`/`develop` branch inconsistency.

## DRY violations

1. **Biggest one**: the actual `awx job_templates launch ... --monitor`
   command is duplicated 12 times, varying only in
   `JOB_TEMPLATE_NAME`, `--limit`, and the `--extra_vars` payload.
   `.awx-runner` only factors out `image`/`before_script` — the launch
   command itself isn't shared via a second hidden job, even though
   the file already knows the `!reference` technique (uses it for
   `.xml-changed`).
2. `--conf.host "$CONTROLLER_HOST" --conf.token "$CONTROLLER_TOKEN"`
   repeated verbatim 12 times.
3. `rules:` blocks copy-pasted per phase within a lane — phase-1 and
   phase-2 share identical `changes:` paths, differing only in
   branch-gate and `when: manual`.
4. The whole dev→preprod / phase-1→phase-2 skeleton is triplicated
   across 3 lanes instead of one template parameterized by
   job-template-name + extra_vars-builder + limit-target (the same
   problem our own `.deploy-template` + `extends` pattern solves).
5. `itc-<env>-phase<N>` inventory-group naming is a repeated string
   literal across all 12 jobs.
