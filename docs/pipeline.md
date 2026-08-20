# Pipeline diagram

Source of truth for the wave-rollout gating described in `CLAUDE.md`.

## Environment / tier flow

Each `<tier>` box below is the 3-wave sequence detailed in the next
section. `db:canary` of `preprod`/`prod` is also the environment
promotion gate.

```mermaid
flowchart TD
    lint["lint\n(yamllint + ansible-lint)"]

    subgraph DEV["dev"]
        direction LR
        devdb["db"] --> devapp["app"] --> devweb["web"]
    end

    subgraph PREPROD["preprod"]
        direction LR
        ppdb["db"] --> ppapp["app"] --> ppweb["web"]
    end

    subgraph PROD["prod"]
        direction LR
        proddb["db"] --> prodapp["app"] --> prodweb["web"]
    end

    lint --> devdb
    devweb -->|"manual: promote to preprod"| ppdb
    ppweb -->|"manual: promote to prod"| proddb

    classDef gate fill:#e8a33d,stroke:#8a5a12,color:#1a1300,font-weight:bold;
    class ppdb,proddb gate
```

## Wave gating (applies inside every tier box above)

`canary` auto-runs once the previous tier's `final` passes. `wave2` and
`final` always require a human click — even in `dev` — so an issue in
`wave2` can't silently reach `final` unreviewed. A wave targeting an
empty inventory group exits 0 without touching any host.

```mermaid
flowchart LR
    canary["canary\n(auto)"] -->|"human reviews canary"| wave2["wave2\n(manual)"]
    wave2 -->|"human reviews wave2"| final["final\n(manual)"]

    canary -.->|"group empty"| skip1["skip, exit 0"]
    wave2 -.->|"group empty"| skip2["skip, exit 0"]
    final -.->|"group empty"| skip3["skip, exit 0"]

    classDef gate fill:#e8a33d,stroke:#8a5a12,color:#1a1300,font-weight:bold;
    classDef skip fill:none,stroke:#888,color:#888,stroke-dasharray: 3 3;
    class wave2,final gate
    class skip1,skip2,skip3 skip
```
