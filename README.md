# Post Maintenance Checks

An Ansible role that validates system health after a reboot, OS update, or patching window. It runs seven categories of checks — system/OS state, application processes, systemd services, container/pod health, PostgreSQL, filesystem integrity, and log scanning — with built-in retry logic so transient startup delays don't cause false failures.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Role Variables](#role-variables)
  - [Retry Behaviour](#retry-behaviour)
  - [System / OS State Checks](#system--os-state-checks)
  - [Application Checks](#application-checks)
  - [Service Checks](#service-checks)
  - [Container / Pod Checks](#container--pod-checks)
  - [PostgreSQL Checks](#postgresql-checks)
  - [Filesystem Checks](#filesystem-checks)
  - [Log / Journal Checks](#log--journal-checks)
- [Check Reference](#check-reference)
  - [System / OS State Checks](#system--os-state-checks-1)
  - [Application Checks](#application-checks-1)
  - [Service Checks](#service-checks-1)
  - [Container / Pod Checks](#container--pod-checks-1)
  - [PostgreSQL Checks](#postgresql-checks-1)
  - [Filesystem Checks](#filesystem-checks-1)
  - [Log / Journal Checks](#log--journal-checks-1)
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

After maintenance events (reboots, kernel updates, package upgrades, database migrations) it is easy to miss a service that failed to restart, a mount that did not come back, or a kernel that did not update. This role automates that verification step and fails loudly before the next change window begins.

```
Maintenance window ends
        │
        ▼
ansible-playbook post_maintenance_checks.yml
        │
        ├─► System / OS checks        (uptime, kernel, disk space, pending reboot)
        ├─► Application checks        (process alive, port open, HTTP 200)
        ├─► Service checks            (systemd active + enabled at boot)
        ├─► Container / Pod checks    (podman/docker containers and podman pods)
        ├─► PostgreSQL checks         (pg_isready, DB exists, R/W test, lag)
        ├─► Filesystem checks         (mount points, LVM, mdadm RAID)
        └─► Log / journal checks      (journal error scan, custom patterns)
                │
                ▼
         PASS → pipeline continues
         FAIL → operator is alerted before the next change is applied
```

All categories are **disabled by default**. Enable only the categories relevant to each host or group.

---

## Requirements

| Requirement | Notes |
|---|---|
| Ansible | 2.14 or later |
| `postgresql-client` | Required on managed hosts when PostgreSQL checks are enabled (`psql`, `pg_isready`) |
| `podman` or `docker` | At least one must be present when `health_check_containers` is non-empty |
| `lvm2` | Required on managed hosts when `health_check_filesystem_check_lvm: true` |
| `dnf-utils` | Required on RHEL/CentOS hosts when `health_check_system_check_pending_reboot: true` (`needs-restarting`) |
| Privilege escalation | Only needed for PostgreSQL tasks (`become_user: postgres`); all other checks run unprivileged |

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

### System / OS State Checks

```yaml
health_check_system_enabled: false
```

| Variable | Default | Description |
|---|---|---|
| `health_check_system_enabled` | `false` | Set to `true` to enable this check category |
| `health_check_system_max_uptime_seconds` | `0` | Assert uptime is below this value (proves a reboot occurred). `0` disables the check. Example: `3600` for a 1-hour window |
| `health_check_system_expected_kernel` | `""` | Expected `uname -r` string after patching. Empty string disables the check |
| `health_check_system_disk_min_free_percent` | `15` | Minimum free disk space percentage on each listed mount |
| `health_check_system_disk_check_mounts` | `[]` | Mount points to check for free space. Example: `["/", "/boot", "/var"]` |
| `health_check_system_check_pending_reboot` | `false` | Check whether a secondary reboot is still required. Uses `/var/run/reboot-required` on Debian/Ubuntu and `needs-restarting -r` on RHEL/CentOS |

**Example:**

```yaml
health_check_system_enabled: true
health_check_system_max_uptime_seconds: 3600        # reboot must have happened in the last hour
health_check_system_expected_kernel: "5.14.0-427.13.1.el9_4.x86_64"
health_check_system_disk_min_free_percent: 15
health_check_system_disk_check_mounts:
  - /
  - /boot
  - /var
health_check_system_check_pending_reboot: true      # explicitly opt in
```

---

### Application Checks

```yaml
health_check_applications_enabled: false
health_check_applications: []
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
health_check_applications_enabled: true
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
health_check_services_enabled: false
health_check_services: []
```

| Key | Type | Default | Description |
|---|---|---|---|
| `name` | string | — | systemd unit name (without `.service` suffix) |
| `enabled` | bool | `true` | Also assert the unit is enabled at boot |

**Example:**

```yaml
health_check_services_enabled: true
health_check_services:
  - name: nginx
  - name: sshd
  - name: firewalld
  - name: postgresql
  - name: redis
    enabled: false   # running but boot-disabled is acceptable here
```

---

### Container / Pod Checks

```yaml
health_check_containers_enabled: false
health_check_containers: []
```

Each item in the list is a dict. Only `name` is required.

| Key | Type | Default | Description |
|---|---|---|---|
| `name` | string | — | Container or pod name as known to the runtime |
| `runtime` | string | `auto` | `podman`, `docker`, or `auto`. `auto` uses podman when available, docker otherwise |
| `type` | string | `container` | `container` or `pod`. `pod` uses `podman pod inspect` and is podman-only |

Runtime detection runs automatically with `which podman` / `which docker`. The task fails early if neither runtime is found on the host.

**Example:**

```yaml
health_check_containers_enabled: true
health_check_containers:
  - name: my-web-app              # auto-detect runtime, type=container
  - name: nginx
    runtime: docker
  - name: my-api
    runtime: podman
  - name: my-pod
    runtime: podman
    type: pod
```

---

### PostgreSQL Checks

| Variable | Default | Description |
|---|---|---|
| `health_check_postgresql_enabled` | `false` | Set to `true` to enable all PostgreSQL checks |
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

### Filesystem Checks

```yaml
health_check_filesystem_enabled: false
```

| Variable | Default | Description |
|---|---|---|
| `health_check_filesystem_enabled` | `false` | Set to `true` to enable this check category |
| `health_check_filesystem_check_mounts` | `[]` | List of mount point paths that must be present. Useful for NFS, SAN, and secondary local mounts |
| `health_check_filesystem_check_lvm` | `false` | Run `vgs` to verify LVM volume groups are accessible and not in a partial state |
| `health_check_filesystem_check_raid` | `false` | Check `/proc/mdstat` for degraded mdadm RAID arrays |

**Example:**

```yaml
health_check_filesystem_enabled: true
health_check_filesystem_check_mounts:
  - /mnt/data
  - /mnt/backup
  - /srv/nfs/exports
health_check_filesystem_check_lvm: true
health_check_filesystem_check_raid: true
```

---

### Log / Journal Checks

```yaml
health_check_log_enabled: false
```

| Variable | Default | Description |
|---|---|---|
| `health_check_log_enabled` | `false` | Set to `true` to enable this check category |
| `health_check_log_journal_errors` | `false` | Scan `journalctl -p 3 -b 0` for errors and above in the current boot |
| `health_check_log_fail_on_journal_errors` | `false` | When `true`, fail the play if journal errors are found. Default `false` warns only |
| `health_check_log_patterns` | `[]` | Additional regex patterns to search for in the current boot journal using `journalctl --grep`. Matches are reported as warnings |

**Example:**

```yaml
health_check_log_enabled: true
health_check_log_journal_errors: true               # explicitly opt in
health_check_log_fail_on_journal_errors: false
health_check_log_patterns:
  - "segfault"
  - "OOM killer"
  - "kernel panic"
  - "SCSI error"
```

---

## Check Reference

### System / OS State Checks

| Check | How it works | Fails play |
|---|---|---|
| Reboot confirmed | `ansible_uptime_seconds` ≤ `health_check_system_max_uptime_seconds` | Yes |
| Kernel version | `uname -r` compared against `health_check_system_expected_kernel` | Yes |
| Disk space | `ansible_mounts` fact — `size_available / size_total` per mount | Yes |
| Pending reboot (Debian/Ubuntu) | `stat /var/run/reboot-required` | Yes |
| Pending reboot (RHEL/CentOS) | `needs-restarting -r` exit code | Yes |

### Application Checks

| Check | How it works | Retries |
|---|---|---|
| Process alive | `pgrep -x <process>` | Yes |
| PID file exists | `stat` the file path | Yes |
| PID in file is live | `kill -0 <pid>` | Yes |
| Port listening | `ansible.builtin.wait_for port` | Built-in timeout |
| HTTP endpoint | `ansible.builtin.uri` GET + status code assertion | Yes |

### Service Checks

| Check | How it works | Retries |
|---|---|---|
| Service is active | `systemctl is-active <name>` | Yes |
| Service is enabled | `systemctl is-enabled <name>` | Yes |

### Container / Pod Checks

| Check | How it works | Retries |
|---|---|---|
| Container is running (podman) | `podman inspect --format '{{.State.Status}}'` → `running` | Yes |
| Container is running (docker) | `docker inspect --format '{{.State.Status}}'` → `running` | Yes |
| Pod is running (podman) | `podman pod inspect --format '{{.State}}'` → `running` | Yes |

Runtime auto-detection (`auto`) prefers podman over docker. If `type: pod` is set but podman is not available, that item is skipped.

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

### Filesystem Checks

| Check | How it works | Fails play |
|---|---|---|
| Mount points present | `ansible_mounts` fact compared against `health_check_filesystem_check_mounts` | Yes |
| LVM volume groups healthy | `vgs` exit code and absence of `partial` in output | Yes |
| No degraded RAID arrays | `/proc/mdstat` searched for `[U_]`-style degraded drive pattern | Yes |

### Log / Journal Checks

| Check | How it works | Fails play |
|---|---|---|
| No critical journal errors | `journalctl -p 3 -b 0 --no-pager -q` — non-empty output triggers warning | Configurable via `health_check_log_fail_on_journal_errors` |
| Custom pattern matches | `journalctl -b 0 --grep "<pattern>"` per entry in `health_check_log_patterns` | No (warning only) |

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/your-org/post-maintenance-checks.git
cd post-maintenance-checks

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

    # ── System checks ──────────────────────────────────────────────────────────
    health_check_system_enabled: true
    health_check_system_max_uptime_seconds: 3600
    health_check_system_disk_min_free_percent: 15
    health_check_system_disk_check_mounts:
      - /
      - /boot
    health_check_system_check_pending_reboot: true

    # ── Service checks ─────────────────────────────────────────────────────────
    health_check_services_enabled: true
    health_check_services:
      - name: nginx
      - name: sshd
      - name: postgresql
      - name: firewalld

    # ── Application checks ─────────────────────────────────────────────────────
    health_check_applications_enabled: true
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

    # ── Container checks ───────────────────────────────────────────────────────
    health_check_containers_enabled: true
    health_check_containers:
      - name: my-web-app
      - name: nginx
        runtime: docker

    # ── PostgreSQL checks ──────────────────────────────────────────────────────
    health_check_postgresql_enabled: true
    health_check_postgresql_databases:
      - myapp_production
    health_check_postgresql_run_rw_test: true
    health_check_postgresql_min_version: "14"

    # ── Log checks ─────────────────────────────────────────────────────────────
    health_check_log_enabled: true
    health_check_log_journal_errors: true
    health_check_log_patterns:
      - "segfault"
      - "OOM killer"

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
# System checks — verify reboot and disk space
health_check_system_enabled: true
health_check_system_max_uptime_seconds: 7200
health_check_system_disk_check_mounts:
  - /
  - /var/lib/pgsql

# PostgreSQL checks are only meaningful on database hosts
health_check_postgresql_enabled: true
health_check_postgresql_databases:
  - production_db
health_check_postgresql_run_rw_test: true
health_check_postgresql_max_replication_lag_bytes: 5242880  # 5 MB

# Filesystem: verify data and backup mounts came back
health_check_filesystem_enabled: true
health_check_filesystem_check_mounts:
  - /mnt/pgdata
health_check_filesystem_check_lvm: true

# Log scan
health_check_log_enabled: true

# Web-layer checks are not relevant here
health_check_applications_enabled: false
health_check_containers_enabled: false
```

### Group-specific overrides (`group_vars/webservers.yml`)

```yaml
health_check_system_enabled: true
health_check_system_disk_check_mounts:
  - /
  - /var

health_check_services_enabled: true
health_check_services:
  - name: nginx
  - name: sshd

health_check_applications_enabled: true
health_check_applications:
  - name: Web Application
    process: gunicorn
    port: 8080
    url: http://localhost:8080/health

health_check_postgresql_enabled: false
health_check_filesystem_enabled: false
```

---

## Usage Scenarios

### After a reboot — verify the reboot actually happened

```yaml
health_check_system_enabled: true
health_check_system_max_uptime_seconds: 1800   # fail if uptime > 30 min
health_check_system_check_pending_reboot: true
# Increase retries to give services extra time to start
health_check_retries: 10
health_check_delay: 15
```

### After a kernel update — verify the new kernel is running

```yaml
health_check_system_enabled: true
# Capture `uname -r` on the host after the update and set it here
health_check_system_expected_kernel: "5.14.0-427.13.1.el9_4.x86_64"
```

### After an OS / package update (no reboot)

```yaml
# Services may restart quickly; default retries are usually fine
health_check_retries: 5
health_check_delay: 10
health_check_log_enabled: true
health_check_log_journal_errors: true
```

### After a PostgreSQL major version upgrade

```yaml
health_check_postgresql_enabled: true
health_check_postgresql_min_version: "16"   # enforce the new version
health_check_postgresql_run_rw_test: true
```

### After a container host update

```yaml
health_check_containers_enabled: true
health_check_containers:
  - name: app-api
    runtime: podman
  - name: app-worker
    runtime: podman
  - name: app-pod
    runtime: podman
    type: pod
```

### After a storage change (new NFS mounts, LVM resize, RAID rebuild)

```yaml
health_check_filesystem_enabled: true
health_check_filesystem_check_mounts:
  - /mnt/nfs-data
  - /mnt/san-vol
health_check_filesystem_check_lvm: true
health_check_filesystem_check_raid: true
```

### Primary + standby replication check

```yaml
# Run on the primary only
health_check_postgresql_enabled: true
health_check_postgresql_max_replication_lag_bytes: 10485760  # 10 MB
```

---

## Project Structure

```
post_maintenance_checks/
├── playbooks/
│   ├── post_maintenance_checks.yml        # ready-to-run example playbook
│   └── post_maintenance_checks_simple.yml # minimal playbook (role defaults only)
├── roles/
│   └── post_maintenance_checks/
│       ├── defaults/
│       │   └── main.yml                   # all variables and their defaults
│       ├── tasks/
│       │   ├── main.yml                   # dispatcher — includes sub-task files
│       │   ├── system_checks.yml          # uptime, kernel, disk, pending reboot
│       │   ├── application_checks.yml     # process, PID file, port, HTTP
│       │   ├── service_checks.yml         # systemd active + enabled
│       │   ├── container_checks.yml       # podman/docker containers and pods
│       │   ├── postgresql_checks.yml      # pg_isready, DBs, R/W test, lag
│       │   ├── filesystem_checks.yml      # mounts, LVM, RAID
│       │   └── log_checks.yml             # journal error scan, custom patterns
│       ├── handlers/
│       │   └── main.yml
│       ├── meta/
│       │   └── main.yml                   # Galaxy metadata and platform support
│       └── README.md                      # role-scoped reference
├── setup/                                 # test-environment provisioning playbooks
│   ├── setup_test_env.yml
│   ├── setup_nginx.yml
│   ├── setup_gunicorn_app.yml
│   ├── setup_celery_worker.yml
│   ├── setup_redis.yml
│   ├── setup_postgresql.yml
│   ├── setup_podman_resources.yml
│   └── setup_java_app.yml
└── tests/
    └── test_post_maintenance_checks.yml   # local smoke-test playbook
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
