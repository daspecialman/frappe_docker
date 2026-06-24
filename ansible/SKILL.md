# Ansible App Deployment Skill

Deploy any Docker Compose application to servers provisioned by [ansible-infra](https://github.com/org/ansible-infra).

## Overview

This skill sets up a production-ready Ansible deployment inside an application repository. It generates the full `ansible/` directory with inventory, group vars, playbooks (deploy, verify, rollback), vault template, and CI-ready commands.

### Capabilities

| Capability | Description |
| :--- | :--- |
| Preflight checks | SSH, Docker daemon, `docker compose`, external networks, required collections |
| Git deploy | Clone/pull private repos using the server's deploy key |
| Local deploy | Deploy from local machine — creates directory and syncs project files to server automatically |
| Two-compose | First-class support for split `compose.db.yml` + `compose.yml` with separate project names |
| Tag-based deploys | `--tags db`, `--tags app`, or full deploy — each self-contained |
| Env sync | Local `.env` auto-synced to server (alternative to vault template) |
| Vault secrets | Optional vault-encrypted secrets with separate vault file |
| Health verification | Container status + HTTP health endpoint polling |
| Rollback | Revert app to previous ref and re-verify (DB data untouched) |
| Hooks | Pre/post deploy commands, DB migrations |
| Idempotency | Safe to re-run any playbook repeatedly |
| CI/CD | Non-interactive vault mode, pipeline stage commands |

### Assumptions

1. The target server is provisioned by `ansible-infra` (Docker, UFW, Traefik/Nginx, deploy key, `/projects`)
2. The server SSH key/connectivity works from your local machine
3. The project has a `docker-compose.yml` (optionally also `compose.db.yml`)
4. You have `ansible`, `ansible-galaxy`, and `rsync` installed locally

---

## Step 1 — Collect Inputs

Ask the user for every variable before generating files. Group them by category.

### Required

| Variable | Example | Description |
| :--- | :--- | :--- |
| `server_ip` | `203.0.113.50` | Production server IP address |
| `app_name` | `myapp` | Application name (used for directory paths and container naming) |
| `deploy_ref` | `main` | Git ref to deploy — branch, tag, or full SHA |

### Required with defaults

These have sensible defaults — confirm with user and let them override:

| Variable | Default | Description |
| :--- | :--- | :--- |
| `projects_root` | `/projects` | Server directory for app code |
| `compose_subdir` | `.` | Subdirectory containing `docker-compose.yml` (set to `""` for root) |
| `deploy_serial` | `1` | Max parallel hosts during deploy (use `1` for production safety) |
| `healthcheck_retries` | `10` | Number of health check retries |
| `healthcheck_delay` | `6` | Seconds between health check retries |
| `healthcheck_validate_certs` | `true` | Validate SSL cert on health check URL |

### Optional

| Variable | Default | Description |
| :--- | :--- | :--- |
| `staging_ip` | (not set) | If staging exists, generate staging inventory + vars |
| `app_repo` | (not set) | Git URL for clone/pull. Omit for local-only deploy (sync files from local) |
| `app_env_vars` | `{}` | Plain (non-secret) environment variables for `.env` file (vault mode) |
| `healthcheck_url` | (not set) | HTTP endpoint to poll after deploy (skipped if empty) |
| `compose_profiles` | `[]` | Docker Compose profile names to enable (e.g. `['analytics']`) |
| `required_networks` | `[traefik]` | External Docker networks that must exist before deploy |
| `app_pre_deploy_cmd` | (not set) | Shell command run before `compose up` |
| `app_post_deploy_cmd` | (not set) | Shell command run after `compose up` |
| `app_migrate_cmd` | (not set) | Shell command for database migrations (runs after `compose up`) |
| `traefik_network` | `traefik` | Traefik Docker network name (for label validation) |

### Two-compose variables (when project uses split compose files)

| Variable | Default | Description |
| :--- | :--- | :--- |
| `split_compose` | `false` | Set to `true` when project uses `compose.db.yml` + `compose.yml` |
| `compose_project_db` | `"{{ app_name }}-db"` | Docker Compose project name for the database |
| `compose_project_app` | `"{{ app_name }}"` | Docker Compose project name for the app services |

### Env strategy (ask the user)

> Ask: _"Where do environment variables come from?"_ Choose one pattern:

**Pattern A — Vault + template** (default for new projects without an existing `.env`)
- Secret vars go into `ansible/inventory/group_vars/vault.yml` (encrypted with `ansible-vault`)
- Non-secret vars go into `ansible/inventory/group_vars/all.yml` under `app_env_vars`
- Playbook renders `.env` from `templates/.env.j2`

**Pattern B — Local `.env` sync** (when user already maintains `.env` in their repo)
- `.env` lives in the project root on user's machine (gitignored)
- Playbook checks it exists locally, backs up the server copy, syncs local → server
- No vault, no template, no duplicate data entry
- User runs: `ansible-playbook ... deploy.yml` with no extra flags
- Set `sync_env_from_local: true` in `all.yml`

### Secrets (for vault)

| Variable | Example | Description |
| :--- | :--- | :--- |
| `vault_env_vars` | `{DATABASE_URL: ...}` | Secret env vars requiring encryption |

> Ask: _"Which environment variables contain secrets (passwords, tokens, URLs with credentials)?"_ — these go into `vault_env_vars` in the vault file. Non-secret env vars go into `app_env_vars` in `all.yml`.

---

## Step 2 — Validate Project (Pre-Generation)

Before generating files, inspect the project's `docker-compose.yml` and warn about issues.

### Traefik label validation (when server uses Traefik)

Check the compose file for:

1. **Label `traefik.enable=true`** — warn if missing
2. **Routing labels** — at minimum need `rule`, `entrypoints` (`websecure`), `tls.certresolver=letsencrypt`, `server.port`
3. **Network membership** — app service must join `{{ traefik_network }}` network declared as `external: true`
4. **`traefik.docker.network` label** — required when the container joins multiple networks
5. **No published ports** — remove `ports:` from the compose service (Traefik routes internally)

If labels are missing, show the user the required snippet:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.{{ app_name }}.rule=Host(`app.example.com`)"
  - "traefik.http.routers.{{ app_name }}.entrypoints=websecure"
  - "traefik.http.routers.{{ app_name }}.tls.certresolver=letsencrypt"
  - "traefik.http.services.{{ app_name }}.loadbalancer.server.port=3000"

networks:
  {{ traefik_network }}:
    external: true
```

### Nginx label validation (when server uses Nginx)

- Warn if compose file publishes ports `80` or `443` (Nginx owns those)
- Confirm `group_vars/all.yml` `sites` list includes the app domain and matches the internal port

### General compose checks

- File exists and is valid YAML
- No `container_name:` duplicates across projects on the same server
- Restart policy is set (`unless-stopped` recommended)

### Two-compose check (when `split_compose: true`)

- Verify both `compose.db.yml` and `compose.yml` exist
- DB compose must join the app's internal network so containers can reach it
- Use distinct `project_name` values so containers don't collide (`{{ app_name }}-db` and `{{ app_name }}`)
- Deploy order matters: DB must be up before app compose starts
- Rollback must only touch the app compose — never DB data

---

## Step 3 — Generate Files

Create the following structure. Replace `{{ placeholders }}` with user-provided values.

### File tree created

```
<project-root>/
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── production.ini
│   │   ├── staging.ini          # only if staging_ip is provided
│   │   └── group_vars/          # IMPORTANT: must be inside inventory/ for Ansible to find it
│   │       ├── all.yml
│   │       ├── production.yml   # only with environment split
│   │       ├── staging.yml      # only if staging_ip is provided
│   │       └── vault.yml        # encrypt before commit
│   ├── playbooks/
│   │   ├── deploy.yml
│   │   ├── verify.yml
│   │   └── rollback.yml
│   ├── requirements.yml
│   ├── templates/
│   │   └── .env.j2              # only if using vault mode
│   └── README.md                # generated runbook
├── docker-compose.yml           # existing — not modified
└── .gitignore                   # append to existing, do not overwrite
```

> **Critical**: `group_vars/` must live **inside** `inventory/`, not as a sibling of `inventory/`. When using `-i ansible/inventory/production.ini`, Ansible looks for `ansible/inventory/group_vars/`. Putting `group_vars/` at `ansible/group_vars/` will cause `undefined variable` errors at runtime.

### ansible/ansible.cfg

```ini
[defaults]
host_key_checking = False
retry_files_enabled = False
interpreter_python = auto_silent
stdout_callback = yaml

[privilege_escalation]
become = False
```

### ansible/inventory/production.ini

```ini
[production]
server ansible_host={{ server_ip }}

[production:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

### ansible/inventory/staging.ini

Generate **only** if the user provides `staging_ip`. Same structure:

```ini
[staging]
server ansible_host={{ staging_ip }}

[staging:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

### ansible/inventory/group_vars/all.yml

Shared variables across all environments.

```yaml
---
app_name: "{{ app_name }}"
projects_root: "{{ projects_root }}"
deploy_ref: "{{ deploy_ref }}"

# Whether project uses separate compose files for DB and app
# split_compose: true

# Docker Compose project names (prevents container name collisions)
compose_project_db: "{{ app_name }}-db"
compose_project_app: "{{ app_name }}"

# Path to .env on the server
env_file_path: "{{ projects_root }}/{{ app_name }}/.env"

# External Docker networks that must exist before deploy
required_networks:
  - traefik
  # - talynted-network

# Docker Compose profiles to enable (empty = no profiles)
compose_profiles: []

healthcheck_retries: 10
healthcheck_delay: 6
healthcheck_url: ""
healthcheck_validate_certs: true

# Environment strategy — pick ONE of the two patterns below
# Pattern A: Vault + template (uncomment to enable)
# app_env_vars:
#   NODE_ENV: "production"
#   DOMAIN: "example.com"

# Pattern B: Local .env sync (uncomment to enable — requires Pattern A to be disabled)
# sync_env_from_local: true
# env_backup: true

# Hooks — uncomment and fill to enable
# app_pre_deploy_cmd: ""
# app_post_deploy_cmd: ""
# app_migrate_cmd: ""

# Git repo (set for git clone/pull; omit for local-only deploy)
# app_repo: "git@github.com:org/{{ app_name }}.git"
```

> **When `split_compose` is true**: `compose_project_db` and `compose_project_app` define distinct Docker Compose project names. Containers start with these prefixes (e.g. `p78-cms-db-db-1`, `p78-cms-ghost-1`).

### ansible/inventory/group_vars/production.yml

Generate **only** if the user provides both `server_ip` and `staging_ip` (environment split). Without a staging, production vars live in `all.yml`.

```yaml
---
deploy_ref: "{{ deploy_ref_production | default('main') }}"
healthcheck_url: "{{ healthcheck_url_production }}"
app_env_vars:
  NODE_ENV: "production"
```

### ansible/inventory/group_vars/staging.yml

Generate **only** if the user provides `staging_ip`.

```yaml
---
deploy_ref: "{{ deploy_ref_staging | default('develop') }}"
healthcheck_url: "{{ healthcheck_url_staging }}"
app_env_vars:
  NODE_ENV: "staging"
```

### ansible/inventory/group_vars/vault.yml

Create with placeholder values. **Instruct the user to fill in secrets, then encrypt before committing.**

```yaml
---
# Secret environment variables — fill in values, then encrypt:
#   ansible-vault encrypt ansible/inventory/group_vars/vault.yml
vault_env_vars:
{% for key in vault_env_vars_keys %}
  {{ key }}: ""
{% endfor %}
```

> **Critical:** Tell the user:
> 1. Replace empty strings with real secrets
> 2. Run `ansible-vault encrypt ansible/inventory/group_vars/vault.yml`
> 3. Never commit the file while decrypted in plain text
> 4. Use `ansible-vault edit ansible/inventory/group_vars/vault.yml` to make changes later

### ansible/playbooks/deploy.yml

The deploy playbook uses a **self-contained tag system**: `--tags db` and `--tags app` each include their prerequisites (code sync, env sync, network checks) so they work from a fresh server.

```yaml
---
- name: Deploy {{ app_name }}
  hosts: all
  gather_facts: no
  serial: 1

  pre_tasks:
    - name: Verify SSH connectivity
      ansible.builtin.wait_for_connection:
        timeout: 10
      tags: [always, preflight]

    - name: Verify Docker is running
      ansible.builtin.command: docker info
      changed_when: false
      tags: [always, preflight]

    - name: Verify docker compose is available
      ansible.builtin.command: docker compose version
      changed_when: false
      tags: [always, preflight]

    - name: Verify required collections
      ansible.builtin.command: >
        ansible-galaxy collection list community.docker ansible.posix
      changed_when: false
      register: _collections
      failed_when: false
      tags: [always, preflight]

    - name: Fail if required collections are missing
      ansible.builtin.fail:
        msg: >
          Required Ansible collections not installed.
          Run: ansible-galaxy collection install -r ansible/requirements.yml
      when: >
        'community.docker' not in _collections.stdout
        or 'ansible.posix' not in _collections.stdout
      tags: [always, preflight]

  tasks:
    # --- Code delivery (git or local sync) ---
    - name: Clone or pull repository
      ansible.builtin.git:
        repo: "{{ app_repo }}"
        dest: "{{ projects_root }}/{{ app_name }}"
        version: "{{ deploy_ref }}"
        force: yes
        accept_hostkey: yes
      when: app_repo is defined and app_repo | length > 0
      tags: [deploy, git, app, db]

    - name: Ensure project directory exists (local mode)
      ansible.builtin.file:
        path: "{{ projects_root }}/{{ app_name }}"
        state: directory
        mode: "0755"
      when: app_repo is not defined or app_repo | length == 0
      tags: [deploy, check, app, db]

    - name: Sync project files from local to server (local mode)
      ansible.posix.synchronize:
        src: "{{ playbook_dir }}/../../"
        dest: "{{ projects_root }}/{{ app_name }}"
        delete: no
        rsync_opts:
          - "--exclude=.git"
          - "--exclude=data"
      when: app_repo is not defined or app_repo | length == 0
      tags: [deploy, check, app, db]

    # --- Env file (vault mode or local sync) ---
    - name: Write .env file from template (vault mode)
      ansible.builtin.template:
        src: .env.j2
        dest: "{{ env_file_path }}"
        mode: "0600"
      when: >
        sync_env_from_local is not defined or not sync_env_from_local
      tags: [deploy, env, app, db]

    - name: Check local .env exists (local sync mode)
      delegate_to: localhost
      ansible.builtin.stat:
        path: "{{ playbook_dir }}/../../.env"
      register: _local_env
      when: sync_env_from_local is defined and sync_env_from_local
      tags: [deploy, check, app, db]

    - name: Fail if local .env is missing
      ansible.builtin.fail:
        msg: >
          .env not found at {{ playbook_dir }}/../../.env.
          Create it from .env.example before deploying.
      when: >
        sync_env_from_local is defined and sync_env_from_local
        and not _local_env.stat.exists
      tags: [deploy, check, app, db]

    - name: Backup existing .env on server
      ansible.builtin.copy:
        src: "{{ env_file_path }}"
        dest: "{{ env_file_path }}.bak"
        remote_src: yes
      ignore_errors: yes
      when: sync_env_from_local is defined and sync_env_from_local
      tags: [deploy, check, app, db]

    - name: Sync .env from local to server
      ansible.builtin.copy:
        src: "{{ playbook_dir }}/../../.env"
        dest: "{{ env_file_path }}"
        mode: "0600"
      when: sync_env_from_local is defined and sync_env_from_local
      tags: [deploy, check, app, db]

    # --- Pre-deploy checks ---
    - name: Assert required Docker networks exist
      ansible.builtin.command: docker network inspect {{ item }}
      changed_when: false
      loop: "{{ required_networks }}"
      tags: [deploy, check, app, db]

    - name: Run pre-deploy hook
      ansible.builtin.command: "{{ app_pre_deploy_cmd }}"
      args:
        chdir: "{{ projects_root }}/{{ app_name }}"
      when: app_pre_deploy_cmd is defined and app_pre_deploy_cmd | length > 0
      tags: [deploy, hooks]

    # --- DB deploy (for split-compose projects) ---
    - name: Deploy database service
      community.docker.docker_compose_v2:
        project_src: "{{ projects_root }}/{{ app_name }}"
        files:
          - compose.db.yml
        project_name: "{{ compose_project_db }}"
        state: present
      tags: [deploy, db]
      when: split_compose is defined and split_compose

    - name: Wait for database to be healthy
      community.docker.docker_container_info:
        name: "{{ compose_project_db }}-db-1"
      register: _db_health
      until:
        - _db_health.exists
        - _db_health.container.State.Status == "running"
        - _db_health.container.State.Health is defined
        - _db_health.container.State.Health.Status == "healthy"
      retries: "{{ healthcheck_retries }}"
      delay: "{{ healthcheck_delay }}"
      tags: [deploy, db]
      when: split_compose is defined and split_compose

    # --- App deploy ---
    - name: Deploy application stack
      community.docker.docker_compose_v2:
        project_src: "{{ projects_root }}/{{ app_name }}"
        files: "{{ [compose.yml] if not (split_compose is defined and split_compose) else omit }}"
        project_name: "{{ compose_project_app if (split_compose is defined and split_compose) else omit }}"
        state: present
        pull: always
        profiles: "{{ compose_profiles if (compose_profiles | length > 0) else omit }}"
      tags: [deploy, app]

    - name: Run post-deploy hook
      ansible.builtin.command: "{{ app_post_deploy_cmd }}"
      args:
        chdir: "{{ projects_root }}/{{ app_name }}"
      when: app_post_deploy_cmd is defined and app_post_deploy_cmd | length > 0
      tags: [deploy, hooks]

    - name: Run database migrations
      ansible.builtin.command: "{{ app_migrate_cmd }}"
      args:
        chdir: "{{ projects_root }}/{{ app_name }}"
      when: app_migrate_cmd is defined and app_migrate_cmd | length > 0
      tags: [deploy, hooks]

    - name: Prune unused Docker images
      ansible.builtin.command: docker image prune -f
      changed_when: false
      tags: [deploy]

  post_tasks:
    - name: Check app container status
      community.docker.docker_container_info:
        name: "{{ compose_project_app }}-{{ item }}"
      loop: "{{ app_service_names | default(['app']) }}"
      register: _container_check
      when: split_compose is defined and split_compose
      tags: [deploy, verify]

    - name: Report container status
      ansible.builtin.debug:
        msg: >
          Container {{ _container.item }} — {{ _container.container.State.Status }}
          {% if _container.container.State.Health is defined %}
          (health: {{ _container.container.State.Health.Status }})
          {% endif %}
      loop: "{{ _container_check.results }}"
      loop_control:
        loop_var: _container
        label: "{{ _container.item }}"
      when:
        - split_compose is defined and split_compose
        - _container.container is defined
      tags: [deploy, verify]

    - name: Health check endpoint
      ansible.builtin.uri:
        url: "{{ healthcheck_url }}"
        method: GET
        status_code: 200
        timeout: 30
        validate_certs: "{{ healthcheck_validate_certs | default(true) }}"
      register: _health
      until: _health.status == 200
      retries: "{{ healthcheck_retries | default(10) }}"
      delay: "{{ healthcheck_delay | default(6) }}"
      when: healthcheck_url is defined and healthcheck_url | length > 0
      tags: [deploy, verify]

    - name: Deployment successful
      ansible.builtin.debug:
        msg: |
          ✓ {{ app_name }} deployed successfully
          ✓ All containers running and healthy
          {% if healthcheck_url is defined and healthcheck_url | length > 0 %}
          ✓ Health check {{ healthcheck_url }} returned 200
          {% endif %}
      tags: [deploy, verify]
```

> **Path note**: `{{ playbook_dir }}` resolves to `ansible/playbooks/`. Use `{{ playbook_dir }}/../../` to reach the project root. Using `../` would point to `ansible/` instead — a common mistake.

### ansible/playbooks/verify.yml

Standalone verification — run independently or from CI to confirm deploy health.

```yaml
---
- name: Verify {{ app_name }}
  hosts: all
  gather_facts: no

  tasks:
    - name: Check DB container (split compose)
      community.docker.docker_container_info:
        name: "{{ compose_project_db }}-db-1"
      register: _db
      when: split_compose is defined and split_compose
      tags: [verify]

    - name: Assert DB is healthy
      ansible.builtin.assert:
        that:
          - _db.exists
          - _db.container.State.Status == "running"
          - _db.container.State.Health is defined
          - _db.container.State.Health.Status == "healthy"
        fail_msg: "DB container is not healthy"
        success_msg: "DB: running and healthy"
      when: split_compose is defined and split_compose
      tags: [verify]

    - name: Check app containers
      community.docker.docker_container_info:
        name: "{{ compose_project_app }}-{{ item }}"
      loop: "{{ app_service_names | default(['app']) }}"
      register: _app_containers
      when: split_compose is defined and split_compose
      tags: [verify]

    - name: Assert app containers running
      ansible.builtin.assert:
        that:
          - _c.container is defined
          - _c.container.State.Status == "running"
        fail_msg: >
          Container {{ _c.item }} is not running
          (state: {{ _c.container.State.Status | default('unknown') }}).
      loop: "{{ _app_containers.results }}"
      loop_control:
        loop_var: _c
        label: "{{ _c.item }}"
      when: split_compose is defined and split_compose
      tags: [verify]

    - name: Health check endpoint
      ansible.builtin.uri:
        url: "{{ healthcheck_url }}"
        method: GET
        status_code: 200
        timeout: 30
        validate_certs: "{{ healthcheck_validate_certs | default(true) }}"
      register: _health
      until: _health.status == 200
      retries: "{{ healthcheck_retries | default(10) }}"
      delay: "{{ healthcheck_delay | default(6) }}"
      when: healthcheck_url is defined and healthcheck_url | length > 0
      tags: [verify]

    - name: Verification passed
      ansible.builtin.debug:
        msg: "✓ {{ app_name }} is healthy — all containers running, health check passing."
      tags: [verify]
```

### ansible/playbooks/rollback.yml

Revert app to a previous deploy ref and re-verify. **DB data is never rolled back** — this playbook only touches the app compose stack.

```yaml
---
- name: Rollback {{ app_name }} (app only — DB data untouched)
  hosts: all
  gather_facts: no
  serial: 1

  vars:
    rollback_ref: "{{ rollback_ref_override | default('HEAD@{1}') }}"

  tasks:
    - name: Record current deploy ref
      ansible.builtin.command:
        cmd: git rev-parse HEAD
        chdir: "{{ projects_root }}/{{ app_name }}"
      register: _current_ref
      when: app_repo is defined and app_repo | length > 0
      tags: [rollback]

    - name: Print current vs rollback ref
      ansible.builtin.debug:
        msg: >
          Rolling back from {{ _current_ref.stdout[:8] | default('unknown') }}
          to {{ rollback_ref }}
      tags: [rollback]

    - name: Checkout rollback ref
      ansible.builtin.git:
        repo: "{{ app_repo }}"
        dest: "{{ projects_root }}/{{ app_name }}"
        version: "{{ rollback_ref }}"
        force: yes
      when: app_repo is defined and app_repo | length > 0
      tags: [rollback]

    - name: Redeploy app stack
      community.docker.docker_compose_v2:
        project_src: "{{ projects_root }}/{{ app_name }}"
        files:
          - "{{ 'compose.yml' if (split_compose is defined and split_compose) else 'docker-compose.yml' }}"
        project_name: "{{ compose_project_app if (split_compose is defined and split_compose) else omit }}"
        state: present
        pull: always
        profiles: "{{ compose_profiles if (compose_profiles | length > 0) else omit }}"
      tags: [rollback]

    - name: Rollback complete
      ansible.builtin.debug:
        msg: "{{ app_name }} rolled back to {{ rollback_ref }}"
      tags: [rollback]
```

### ansible/requirements.yml

```yaml
---
collections:
  - name: community.docker
  - name: ansible.posix
```

### ansible/templates/.env.j2

```jinja2
# Managed by Ansible — do not edit manually
# Generated at {{ ansible_date_time.iso8601 }}
{% for key, value in app_env_vars.items() %}
{{ key }}={{ value }}
{% endfor %}
{% for key, value in vault_env_vars.items() %}
{{ key }}={{ value }}
{% endfor %}
```

> **Only generate this file if using Pattern A (Vault + template)**. When using Pattern B (local `.env` sync), skip this template — the playbook copies `.env` directly from the local machine to the server using `ansible.builtin.copy`.

### .gitignore additions

Append these entries to the project's existing `.gitignore` (do not overwrite):

```gitignore
# Ansible
*.retry

# Secrets — never commit these
.env
ansible/inventory/group_vars/vault-plain.yml
```

> **Note:** Do NOT add `ansible/inventory/group_vars/vault.yml` to `.gitignore`. Once encrypted, vault files must be committed to version control. Only ignore unencrypted copies.

---

## Step 4 — Post-Generation Instructions

After generating all files, tell the user exactly what to do next. The steps differ based on which env strategy they chose.

### If using Pattern A (Vault + template)

```markdown
## Next Steps

1. **Fill in secrets** — Edit `ansible/inventory/group_vars/vault.yml` with real values
2. **Do NOT commit the vault file yet** — it is still in plain text
3. **Install dependencies:**
   ```bash
   ansible-galaxy collection install -r ansible/requirements.yml
   ```
4. **Encrypt the vault file:**
   ```bash
   ansible-vault encrypt ansible/inventory/group_vars/vault.yml
   ```
5. **Verify connectivity:**
   ```bash
   ansible -i ansible/inventory/production.ini all -m ping
   ```
6. **Deploy (first time):**
   ```bash
   ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml \
     -e @ansible/inventory/group_vars/vault.yml \
     --ask-vault-pass
   ```
```

### If using Pattern B (Local `.env` sync) — no vault

```markdown
## Next Steps

1. **Install dependencies:**
   ```bash
   ansible-galaxy collection install -r ansible/requirements.yml
   ```
2. **Verify connectivity:**
   ```bash
   ansible -i ansible/inventory/production.ini all -m ping
   ```
3. **Deploy DB only (first time):**
   ```bash
   ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml --tags db
   ```
4. **Full deploy:**
   ```bash
   ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml
   ```
```

### If using split compose (`split_compose: true`)

Add these extra commands:

```markdown
## Split compose commands

- **DB only:** `ansible-playbook ... --tags db`
- **App only:** `ansible-playbook ... --tags app`
- **Full deploy (DB then app):** `ansible-playbook ... deploy.yml` (no tag filter)
- **With profiles:** add `-e "compose_profiles=['analytics']"` etc.
- **Rollback app:** `ansible-playbook ... rollback.yml` (DB never touched)
- **Verify:** `ansible-playbook ... verify.yml`
```

---

## Runbook

### Install dependencies (first time)

```bash
ansible-galaxy collection install -r ansible/requirements.yml
ansible -i ansible/inventory/production.ini all -m ping
```

### Deploy (Pattern B — local .env sync, no vault)

```bash
# Full deploy (DB + app + verify) — for split compose projects
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml

# Deploy DB only
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml --tags db

# Deploy app only (DB must be running)
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml --tags app

# Deploy with profile(s)
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml \
  -e "compose_profiles=['analytics']"

# Update deploy ref
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml \
  -e "deploy_ref=v1.2.3"
```

### Deploy (Pattern A — Vault + template)

```bash
# Full deploy
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml \
  -e @ansible/inventory/group_vars/vault.yml \
  --ask-vault-pass

# Update deploy ref
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml \
  -e @ansible/inventory/group_vars/vault.yml \
  -e "deploy_ref=v1.2.3" \
  --ask-vault-pass
```

### Verify

```bash
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/verify.yml
```

### Rollback (app only, DB untouched)

```bash
# Rollback to previous deploy ref
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/rollback.yml

# Rollback to a specific tag/branch
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/rollback.yml \
  -e "rollback_ref_override=v1.1.0"
```

### Run only specific stages

```bash
# Preflight only
ansible-playbook ... --tags preflight

# Code + env sync only
ansible-playbook ... --tags check

# Deploy (db + app, no verify)
ansible-playbook ... --tags deploy
```

### Staging (if configured)

```bash
ansible-playbook -i ansible/inventory/staging.ini ansible/playbooks/deploy.yml \
  -e @ansible/inventory/group_vars/vault.yml \
  --ask-vault-pass
```

### CI/CD pipeline (vault mode)

For non-interactive CI, use a vault password file (never commit the password file):

```bash
# Pipeline stage 1: Install dependencies
ansible-galaxy collection install -r ansible/requirements.yml

# Pipeline stage 2: Preflight & deploy
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/deploy.yml \
  -e @ansible/inventory/group_vars/vault.yml \
  --vault-password-file /path/to/vault-password.txt \
  -e "deploy_ref=$CI_COMMIT_SHA"

# Pipeline stage 3: Verify
ansible-playbook -i ansible/inventory/production.ini ansible/playbooks/verify.yml \
  -e @ansible/inventory/group_vars/vault.yml \
  --vault-password-file /path/to/vault-password.txt
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| :--- | :--- | :--- |
| `unreachable` / SSH timeout | Wrong IP, key path, or user | Verify `ansible -i ... all -m ping`. Check `~/.ssh/id_ed25519` exists |
| `projects_root does not exist` | Server not provisioned by ansible-infra | Run `ansible-infra` setup playbooks first |
| `'compose_project_db' is undefined` | group_vars not found — wrong directory | Ensure `group_vars/all.yml` is at `ansible/inventory/group_vars/`, not `ansible/group_vars/` |
| `"/projects/p78-cms" is not a directory` | Code not on server and no `app_repo` set | Set `app_repo` or use `--tags app` which also triggers local file sync |
| Local `.env` sync fails silently | `{{ playbook_dir }}/../.env` resolves to `ansible/` not repo root | Use `{{ playbook_dir }}/../../.env` to reach the project root |
| `Permission denied (publickey)` on git clone | Deploy key not added to GitHub repo | Retrieve key: `ansible -i ... all -m shell -a "cat /home/ansible/.ssh/id_github.pub"`. Add to GitHub repo Settings → Deploy Keys |
| `repository not found` on git clone | Wrong repo URL or private repo without deploy key | Verify `app_repo` uses `git@github.com:org/repo.git` format |
| `No such file: docker-compose.yml` | `compose_subdir` points to wrong path | Set `compose_subdir` to the directory containing `docker-compose.yml` |
| DB never becomes healthy | MySQL still initializing or wrong credentials | Check `docker logs {{ compose_project_db }}-db-1` |
| App starts before DB is ready | split_compose not enabled, no health gate | Set `split_compose: true` so playbook waits for DB health |
| Container exits immediately | App crash, misconfiguration | Check `docker logs {{ app_name }}` on server |
| Health check timeout | App not ready within retry window | Increase `healthcheck_retries` or `healthcheck_delay` |
| `Vault password required` without prompt | Missing `--ask-vault-pass` or `--vault-password-file` | Add the flag to your playbook command |
| `community.docker not found` | Collection not installed | Run `ansible-galaxy collection install -r ansible/requirements.yml` |
| `ansible.posix not found` | Collection not installed (needed for local file sync) | Run `ansible-galaxy collection install -r ansible/requirements.yml` |
| Traefik 404 on app domain | Container not on `traefik` network or labels wrong | Verify labels and network in `docker-compose.yml`. See Step 2 validation |
| SSL cert not issued | Domain DNS not pointing to server IP or port 80 blocked | Verify DNS A record. Ensure port 80 is open in UFW |
| `.env` file readable by others | File permissions not restricted | Playbook enforces `0600` — re-run deploy |

---

## Guardrails

### Security

- **Never commit** an unencrypted vault file. Encrypt with `ansible-vault encrypt` before `git add`.
- **Never commit** `.env` files with secrets. `.env` is gitignored. The playbook either generates it (vault mode) or syncs it from local (local sync mode).
- **Never set** `ansible-password` in git-tracked inventory files. Use SSH keys or pass passwords via `-e` at runtime.
- **Always use** `mode: "0600"` for `.env` files — the playbook enforces this.
- **Separate** secrets from plain vars — secrets only in `vault.yml`, plain config only in `all.yml`.
- **When using local `.env` sync**: the `.env` on the server is overwritten on every deploy. Backups are created (`.env.bak`) in case of issues.

### Production safety

- **Always use** `serial: 1` for multi-host production deploys (default in playbooks).
- **Always specify** an explicit `deploy_ref` (branch, tag, or SHA). Never rely on implicit `main` in production.
- **Always verify** after deploy — either rely on the built-in post-tasks or run `verify.yml` independently.
- **Always keep** a rollback path. Without `app_repo` set (local mode), rollback is limited to re-running compose with cached images.
- **DB is never rolled back**: the rollback playbook only touches the app compose stack. DB data survives all rollbacks.
- **DB must start before app**: when using split compose, the deploy playbook deploys DB first, waits for it to be healthy, then deploys the app. Never start app without DB up.

### Idempotency

- `deploy.yml` is safe to re-run — it pulls latest code/compose and reconciles state.
- `verify.yml` is read-only and safe to run at any time.
- `rollback.yml` is destructive to the current deploy — confirm before running in production.
- `.env` sync is idempotent — it backs up the server copy first, then overwrites with the local version.

### File ownership

- All generated files on the server are owned by the `ansible` user.
- The `ansible` user is in the `docker` group — no `sudo` needed for container operations.
- Playbooks use `become: False` — they run as the `ansible` user, not root.
