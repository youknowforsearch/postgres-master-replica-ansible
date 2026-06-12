# PostgreSQL master + replicas (Ansible)

Ansible project to deploy **one PostgreSQL master** and **any number of streaming replicas** in Docker (`network_mode: host`), with optional LVM data-disk setup, air-gapped Docker install, auto-tuning from hardware profile, and configurable log rotation.

No application-specific coupling — configure your own databases, users, and networks via `group_vars/`.

## Features

- **1 master + N replicas** — add hosts under `postgresql-replica` in inventory
- **Auto-tuning** — derives memory, WAL, parallelism, and I/O settings from vCPU, RAM, disk size, and storage type
- **RAM/CPU profiles** — `postgres_ram_to_cpu_ratio: 2` (1:2) or `4` (1:4 GB RAM per vCPU)
- **Storage profiles** — `ssd`, `hdd`, or `iops` (with `postgres_disk_iops`)
- **Log rotation** — PostgreSQL `log_rotation_age` / `log_rotation_size` plus host `logrotate`
- **Metrics** — `postgres_exporter` on each node (Prometheus scrape target)
- **Air-gapped Docker** — offline bundle via Git LFS (`files/docker-deb.tgz`)

## Layout

```
postgres-master-replica-ansible/
├── ansible.cfg
├── requirements.yaml
├── site.yml
├── inventory/hosts.yml          # example hosts (sanitized)
├── group_vars/
│   ├── all.yml                  # SSH, private_network
│   └── postgresql.yml           # secrets, databases, tuning inputs
├── .gitattributes               # Git LFS tracking
├── files/
│   ├── docker-deb.tgz           # offline Docker bundle (Git LFS)
│   └── README.md
└── roles/
    ├── lvm_preflight/
    ├── docker_airgapped/
    └── postgresql/
```

## Quick start

```bash
git lfs install
git lfs pull   # if files/docker-deb.tgz is only a few hundred bytes, run this

ansible-galaxy collection install -r requirements.yaml

# 1. Copy and edit configuration
cp inventory/hosts.yml inventory/hosts.local.yml   # optional local override
# Edit group_vars/postgresql.yml — passwords, disks, databases

# 2. Deploy
ansible-playbook site.yml

# Limit scope: only on given hosts
ansible-playbook site.yml --limit postgresql-master
ansible-playbook site.yml --limit postgresql-replica
```

Point Ansible at a local inventory override:

```bash
ansible-playbook -i inventory/hosts.local.yml site.yml
```

## Inventory (master + replicas)

```yaml
all:
  children:
    postgresql:
      children:
        postgresql-master:
          hosts:
            pg-master-01:
              ansible_host: 192.168.1.10
              bind_ip: 192.168.1.10
              pg_role: master
        postgresql-replica:
          hosts:
            pg-replica-01:
              ansible_host: 192.168.1.11
              bind_ip: 192.168.1.11
              pg_role: replica
            pg-replica-02:   # add more replicas here
              ansible_host: 192.168.1.12
              bind_ip: 192.168.1.12
              pg_role: replica
```

## Auto-tuning

Set in `group_vars/postgresql.yml` (or per-host):

| Variable | Description |
|---|---|
| `postgres_auto_tune` | `true` (default) computes settings; `false` uses static role defaults |
| `postgres_vcpus` | vCPU count (auto-detected from facts if omitted) |
| `postgres_ram_gb` | RAM in GB (auto-detected if omitted) |
| `postgres_ram_to_cpu_ratio` | `2` or `4` — expected GB RAM per vCPU |
| `postgres_disk_size_gb` | Data disk size — drives WAL sizing (~5% of disk, max 16 GB) |
| `postgres_disk_storage_type` | `ssd`, `hdd`, or `iops` |
| `postgres_disk_iops` | Used when `storage_type` is `iops` |

**Memory (typical):** `shared_buffers` ≈ 25% RAM, `effective_cache_size` ≈ 75% RAM, `work_mem` scaled by `max_connections`.

**Connections:** ratio `4` → default 100 connections; ratio `2` → default 200 (override with `postgres_max_connections`).

**I/O:**

| Type | `effective_io_concurrency` | `random_page_cost` |
|---|---|---|
| `ssd` | 200 | 1.1 |
| `hdd` | 2 | 4.0 |
| `iops` | `min(iops/10, 1000)` | 1.1–4.0 by IOPS tier |

**Replication:** `max_wal_senders` = replica count + 4.

Per-host overrides (e.g. smaller replica nodes):

```yaml
pg-replica-02:
  postgres_ram_gb: 8
  postgres_vcpus: 2
  postgres_disk_storage_type: hdd
```

## Log rotation

PostgreSQL internal rotation (`roles/postgresql/defaults/main.yml` or `group_vars`):

```yaml
pg_log_rotation_age: "1d"      # e.g. 1h, 12h, 3d
pg_log_rotation_size: "100MB"
postgres_logrotate_enabled: true
postgres_logrotate_frequency: daily   # daily | weekly | monthly
postgres_logrotate_rotate: 14         # keep N rotated files
postgres_logrotate_maxage: 30         # drop files older than N days
```

## Secrets

Replace `CHANGE_ME_*` placeholders in `group_vars/postgresql.yml` before deploy.

For production, use [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html):

```bash
ansible-vault encrypt group_vars/postgresql.yml
ansible-playbook site.yml --ask-vault-pass
```

## Application databases

Define `postgres_databases` in `group_vars/postgresql.yml`:

```yaml
postgres_databases:
  - name: myapp
    owner:
      username: myapp_owner
      password: "{{ app_db_password }}"
    extensions:
      - uuid-ossp
      - pgcrypto
    readers:
      - username: myapp_reader
        password: "CHANGE_ME_reader"
    writers:
      - username: myapp_writer
        password: "CHANGE_ME_writer"
```

Connection strings (write → master, read → replica):

```
postgresql://myapp_owner:<password>@<master-ip>:5432/myapp
postgresql://myapp_owner:<password>@<replica-ip>:5432/myapp
```

## Metrics

`postgres_exporter` listens on `<bind_ip>:9187` on every node. Scrape both master and replicas in Prometheus.

## Air-gapped Docker

The offline Docker bundle (`files/docker-deb.tgz`, ~110 MB) is tracked with **Git LFS** so it can live in the GitHub repo without hitting the 100 MB file limit.

```bash
git lfs install          # once per machine
git lfs pull             # fetch the real archive after clone
ls -lh files/docker-deb.tgz   # should be ~110 MB, not a tiny pointer
```

The `docker_airgapped` role unarchives it on each host when Docker is not already installed. See `files/README.md`.

Set `docker_airgapped_enabled: false` if Docker is already installed.

## License

Use and modify freely. Review passwords and network rules before any production deployment.
