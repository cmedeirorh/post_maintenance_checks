# post_maintenance_checks

Ansible role that runs a suite of health checks after a system reboot, OS update, or patching window. It verifies that applications, systemd services, containers, and PostgreSQL are all healthy before returning control to the operator.

Every check category is **disabled by default** and must be explicitly opted in with its `_enabled` boolean.

## Checks performed

| Category | Enabled by | What is verified |
|---|---|---|
| **Application** | `health_check_applications_enabled: true` | Process is running (`pgrep`), PID file is live, TCP port is listening, HTTP endpoint returns expected status |
| **Service** | `health_check_services_enabled: true` | systemd unit is `active (running)` and optionally enabled at boot |
| **Container / Pod** | `health_check_containers_enabled: true` | Container or pod is running via podman or docker |
| **PostgreSQL** | `health_check_postgresql_enabled: true` | Service is running, `pg_isready` accepts connections, minimum version, required databases exist, optional read-write sanity test, replication lag (primary), long-running query warning |

## Role variables

All variables live in `defaults/main.yml` with inline documentation. The key ones:

```yaml
# Retry behaviour (useful right after a reboot)
health_check_retries: 5      # number of attempts before failing
health_check_delay: 10       # seconds between retries

# ── Enable / disable each check category (all off by default) ──────────────
health_check_applications_enabled: false
health_check_services_enabled: false
health_check_containers_enabled: false
health_check_postgresql_enabled: false

# Application checks – list of dicts (requires health_check_applications_enabled: true)
health_check_applications:
  - name: My Web App
    process: gunicorn         # pgrep -x <process>
    port: 8080                # TCP port that must be listening
    url: http://localhost:8080/health   # HTTP probe
    url_status: 200           # expected HTTP status (default 200)

# Service checks – list of dicts (requires health_check_services_enabled: true)
health_check_services:
  - name: nginx
  - name: postgresql
    enabled: true             # also assert enabled at boot (default true)

# Container / pod checks – list of dicts (requires health_check_containers_enabled: true)
health_check_containers:
  - name: my-web-app          # container name
  - name: nginx
    runtime: docker           # podman, docker, or auto (default auto)
  - name: my-pod
    runtime: podman
    type: pod                 # container or pod (default container)

# PostgreSQL checks (requires health_check_postgresql_enabled: true)
health_check_postgresql_host: localhost
health_check_postgresql_port: 5432
health_check_postgresql_os_user: postgres      # OS user for psql commands
health_check_postgresql_databases:             # must-exist databases
  - myappdb
health_check_postgresql_run_rw_test: false     # CREATE/INSERT/SELECT/DROP temp table
health_check_postgresql_min_version: "14"      # minimum major version ("" to skip)
health_check_postgresql_max_replication_lag_bytes: 0   # 0 = skip lag check
```

## Example playbook

```yaml
- name: Post-maintenance health checks
  hosts: all
  gather_facts: true
  vars:
    health_check_services_enabled: true
    health_check_services:
      - name: nginx
      - name: postgresql

    health_check_applications_enabled: true
    health_check_applications:
      - name: API server
        process: uvicorn
        port: 8000
        url: http://localhost:8000/healthz

    health_check_postgresql_enabled: true
    health_check_postgresql_databases:
      - production_db
    health_check_postgresql_run_rw_test: true
    health_check_postgresql_min_version: "15"
  roles:
    - role: post_maintenance_checks
```

## Supported platforms

- RHEL / Rocky / AlmaLinux 8, 9
- Ubuntu 20.04, 22.04, 24.04
- Debian 11, 12

## Requirements

- `postgresql-client` (for `psql` / `pg_isready`) must be installed on the managed host when PostgreSQL checks are enabled.
- Ansible 2.14+
