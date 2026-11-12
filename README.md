# post_maintenance_checks

An Ansible role that validates system health after a reboot, OS update, or patching window. It runs three categories of checks — application processes, systemd services, and PostgreSQL — with built-in retry logic so transient startup delays don't cause false failures.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Role Variables](#role-variables)
  - [Retry Behaviour](#retry-behaviour)
  - [Application Checks](#application-checks)
  - [Service Checks](#service-checks)
  - [PostgreSQL Checks](#postgresql-checks)
- [Check Reference](#check-reference)
  - [Application Checks](#application-checks-1)
  - [Service Checks](#service-checks-1)
  - [PostgreSQL Checks](#postgresql-checks-1)
- [Quick Start](#quick-start)
- [Example Playbook](#example-playbook)
- [Inventory Examples](#inventory-examples)
- [Usage Scenarios](#usage-scenarios)
- [Project Structure](#project-structure)
- [Supported Platforms](#supported-platforms)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

After maintenance events (reboots, kernel updates, package upgrades, database migrations) it is easy to miss a service that failed to restart or a database that is no longer accepting connections. This role automates that verification step and fails loudly before the next change window begins.

```
Maintenance window ends
        │
        ▼
ansible-playbook post_maintenance_checks.yml
        │
        ├─► Application checks  (process alive, port open, HTTP 200)
        ├─► Service checks      (systemd active + enabled at boot)
        └─► PostgreSQL checks   (pg_isready, DB exists, R/W test, lag)
                │
                ▼
         PASS → pipeline continues
         FAIL → operator is alerted before the next change is applied
```

---

## Requirements

| Requirement | Notes |
|---|---|
| Ansible | 2.14 or later |
| `postgresql-client` | Required on the managed host only when PostgreSQL checks are enabled (`psql`, `pg_isready`) |
| `curl` / `urllib` | Used internally by the `ansible.builtin.uri` module for HTTP probes |
| Privilege escalation | Only needed for PostgreSQL tasks (`become_user: postgres`); the rest runs unprivileged |

---

## Role Variables

All variables are defined in [`roles/post_maintenance_checks/defaults/main.yml`](roles/post_maintenance_checks/defaults/main.yml) and can be overridden at the playbook, host, or group level.

### Retry Behaviour

| Variable | Default | Description |
|---|---|---|
| `health_check_retries` | `5` | Maximum attempts per check before failing |
| `health_check_delay` | `10` | Seconds to wait between retries |

Increase both values after a cold reboot where services take longer to initialise.

---

### Application Checks

```yaml
health_check_applications: []   # list of application definitions
```

Each item in the list is a dict. All keys except `name` are optional — only define the checks that apply to your application.

| Key | Type | Description |
|---|---|---|
| `name` | string | Human-readable label shown in task output |
| `process` | string | Process name matched with `pgrep -x` |
| `pid_file` | string | Path to a PID file; validates both existence and that the PID is alive |
| `port` | integer | TCP port that must be in `LISTEN` state |
| `host` | string | Host to probe for port check (default `127.0.0.1`) |
| `url` | string | HTTP/HTTPS URL to probe with a GET request |
| `url_status` | integer | Expected HTTP response code (default `200`) |
| `url_validate_certs` | bool | Whether to verify TLS certificates (default `true`) |

**Example:**

```yaml
health_check_applications:
  - name: Gunicorn API
    process: gunicorn
    port: 8000
    url: http://localhost:8000/healthz

  - name: Redis Cache
    process: redis-server
    port: 6379

  - name: Internal Dashboard
    url: https://localhost:8443/status
    url_status: 200
    url_validate_certs: false
```

---

### Service Checks

```yaml
health_check_services: []   # list of systemd unit definitions
```

| Key | Type | Default | Description |
|---|---|---|---|
| `name` | string | — | systemd unit name (without `.service` suffix) |
| `enabled` | bool | `true` | Also assert the unit is enabled at boot |

**Example:**

```yaml
health_check_services:
  - name: nginx
  - name: sshd
  - name: firewalld
  - name: postgresql
  - name: redis
    enabled: false   # running but boot-disabled is acceptable here
```

---

### PostgreSQL Checks

| Variable | Default | Description |
|---|---|---|
| `health_check_postgresql_enabled` | `true` | Set to `false` to skip all PostgreSQL checks |
| `health_check_postgresql_os_user` | `postgres` | OS user used to run `psql` / `pg_isready` |
| `health_check_postgresql_host` | `localhost` | Database host |
| `health_check_postgresql_port` | `5432` | Database port |
| `health_check_postgresql_databases` | `[]` | List of database names that must exist |
| `health_check_postgresql_run_rw_test` | `false` | Run a CREATE / INSERT / SELECT / DROP temp table test in each database |
| `health_check_postgresql_min_version` | `""` | Minimum acceptable major version (e.g. `"14"`); empty string skips the check |
| `health_check_postgresql_max_replication_lag_bytes` | `0` | Maximum WAL lag in bytes on a primary; `0` skips the check |

**Example:**

```yaml
health_check_postgresql_enabled: true
health_check_postgresql_host: localhost
health_check_postgresql_port: 5432
health_check_postgresql_databases:
  - production_db
  - analytics_db
health_check_postgresql_run_rw_test: true
health_check_postgresql_min_version: "15"
health_check_postgresql_max_replication_lag_bytes: 10485760   # 10 MB
```

---

## Check Reference

### Application Checks

| Check | How it works | Retries |
|---|---|---|
| Process alive | `pgrep -x <process>` | Yes |
| PID file exists | `stat` the file path | Yes |
| PID in file is live | `kill -0 <pid>` | No (depends on stat result) |
| Port listening | `ansible.builtin.wait_for port` | Built-in timeout |
| HTTP endpoint | `ansible.builtin.uri` GET + status code assertion | Yes |

### Service Checks

| Check | How it works | Retries |
|---|---|---|
| Service is active | `service_facts` → assert `state == running` | Yes |
| Service is enabled | `service_facts` → assert `status == enabled` | No |

### PostgreSQL Checks

| # | Check | Variable to disable |
|---|---|---|
| 1 | PostgreSQL systemd service is running | `health_check_postgresql_enabled: false` |
| 2 | `pg_isready` accepts connections | `health_check_postgresql_enabled: false` |
| 3 | Server meets minimum major version | `health_check_postgresql_min_version: ""` |
| 4 | Required databases exist in `pg_database` | `health_check_postgresql_databases: []` |
| 5 | Read-write temp table test per database | `health_check_postgresql_run_rw_test: false` |
| 6 | Replication lag below threshold (primary) | `health_check_postgresql_max_replication_lag_bytes: 0` |
| 7 | Warning on queries running > 5 minutes | Always runs (non-fatal) |

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/your-org/app-testing.git
cd app-testing

# 2. Run against your production inventory
ansible-playbook playbooks/post_maintenance_checks.yml \
  -i inventories/production \
  --ask-become-pass

# 3. Run only on the database servers
ansible-playbook playbooks/post_maintenance_checks.yml \
  -i inventories/production \
  --limit db_servers

# 4. Dry-run (check mode shows what would be asserted)
ansible-playbook playbooks/post_maintenance_checks.yml \
  -i inventories/production \
  --check
```

---

## Example Playbook

```yaml
---
- name: Post-maintenance health checks
  hosts: all
  gather_facts: true
  become: false

  vars:
    health_check_retries: 6
    health_check_delay: 10

    health_check_services:
      - name: nginx
      - name: sshd
      - name: postgresql
      - name: firewalld

    health_check_applications:
      - name: API Server
        process: gunicorn
        port: 8000
        url: http://localhost:8000/healthz

      - name: Celery Worker
        process: celery

      - name: Redis
        process: redis-server
        port: 6379

    health_check_postgresql_enabled: true
    health_check_postgresql_host: localhost
    health_check_postgresql_port: 5432
    health_check_postgresql_databases:
      - myapp_production
    health_check_postgresql_run_rw_test: true
    health_check_postgresql_min_version: "14"

  roles:
    - role: post_maintenance_checks
```

---

## Inventory Examples

### Static inventory

```ini
[webservers]
web01.example.com
web02.example.com

[db_servers]
db01.example.com

[all:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### Group-specific overrides (`group_vars/db_servers.yml`)

```yaml
# PostgreSQL checks are only meaningful on database hosts
health_check_postgresql_enabled: true
health_check_postgresql_databases:
  - production_db
health_check_postgresql_run_rw_test: true
health_check_postgresql_max_replication_lag_bytes: 5242880  # 5 MB

# Web-layer checks are not relevant here
health_check_applications: []
```

### Group-specific overrides (`group_vars/webservers.yml`)

```yaml
health_check_postgresql_enabled: false

health_check_services:
  - name: nginx
  - name: sshd

health_check_applications:
  - name: Web Application
    process: gunicorn
    port: 8080
    url: http://localhost:8080/health
```

---

## Usage Scenarios

### After a reboot

```yaml
# Increase retries to give services extra time to start
health_check_retries: 10
health_check_delay: 15
```

### After an OS / package update (no reboot)

```yaml
# Services may restart quickly; default retries are usually fine
health_check_retries: 5
health_check_delay: 10
```

### After a PostgreSQL major version upgrade

```yaml
health_check_postgresql_enabled: true
health_check_postgresql_min_version: "16"   # enforce the new version
health_check_postgresql_run_rw_test: true
```

### After a database migration

```yaml
health_check_postgresql_databases:
  - myapp_production
health_check_postgresql_run_rw_test: true
```

### Primary + standby replication check

```yaml
# Run on the primary only
health_check_postgresql_max_replication_lag_bytes: 10485760  # 10 MB
```

---

## Project Structure

```
app-testing/
├── playbooks/
│   └── post_maintenance_checks.yml   # ready-to-run example playbook
├── roles/
│   └── post_maintenance_checks/
│       ├── defaults/
│       │   └── main.yml              # all variables and their defaults
│       ├── tasks/
│       │   ├── main.yml              # dispatcher — includes sub-task files
│       │   ├── application_checks.yml
│       │   ├── service_checks.yml
│       │   └── postgresql_checks.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── meta/
│       │   └── main.yml              # Galaxy metadata and platform support
│       └── README.md                 # role-scoped reference
└── tests/
    └── test_post_maintenance_checks.yml  # local smoke-test playbook
```

---

## Supported Platforms

| OS | Versions |
|---|---|
| Red Hat Enterprise Linux / Rocky / AlmaLinux | 8, 9 |
| Ubuntu | 20.04 (Focal), 22.04 (Jammy), 24.04 (Noble) |
| Debian | 11 (Bullseye), 12 (Bookworm) |

---

## Contributing

1. Fork the repository and create a feature branch.
2. Make your changes inside `roles/post_maintenance_checks/`.
3. Test locally with the smoke-test playbook:
   ```bash
   ansible-playbook tests/test_post_maintenance_checks.yml -c local -i localhost,
   ```
4. Open a pull request against `main` with a clear description of what changed and why.

---

## License

MIT
