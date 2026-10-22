# Ansible Best Practices Guide

> A practical guide to writing clean, readable, and reusable Ansible code.

---

## Table of Contents

1. [Why Best Practices Matter](#1-why-best-practices-matter)
2. [Guiding Principles](#2-guiding-principles)
3. [Project Structure](#3-project-structure)
4. [Inventory](#4-inventory)
5. [Variables](#5-variables)
6. [Playbooks](#6-playbooks)
7. [Roles](#7-roles)
8. [Tasks](#8-tasks)
9. [Tags](#9-tags)
10. [Idempotency](#10-idempotency)
11. [Templates and Files](#11-templates-and-files)
12. [Handlers](#12-handlers)
13. [Secrets Management](#13-secrets-management)
14. [Collections and Dependencies](#14-collections-and-dependencies)
15. [Error Handling](#15-error-handling)
16. [Performance](#16-performance)
17. [Testing and Linting](#17-testing-and-linting)
18. [Version Control and CI/CD](#18-version-control-and-cicd)
19. [Documentation](#19-documentation)
20. [Quick Reference Checklist](#20-quick-reference-checklist)

---

## 1. Why Best Practices Matter

Ansible is easy to start with — a few lines of YAML and you're configuring servers. That low barrier is also its biggest risk: without conventions, automation quickly becomes a pile of one-off playbooks that nobody fully understands or trusts.

Following a shared set of best practices gives us:

| Benefit | What it means in practice |
|---|---|
| **Reliability** | Idempotent, tested code produces the same result every run. No surprises in production. |
| **Readability** | Anyone on the team can open a playbook and understand what it does within minutes. |
| **Reusability** | Well-designed roles are written once and used across many projects and environments. |
| **Maintainability** | Consistent structure makes changes safe and quick; bugs are easier to find and fix. |
| **Onboarding** | New team members become productive faster when everything follows the same patterns. |
| **Security** | Secrets are handled properly, privileges are explicit, and changes are auditable. |
| **Scalability** | Code that works for 5 hosts keeps working for 5,000. |
| **Auditability** | Infrastructure-as-code in version control provides a full history of who changed what and why. |

**The cost of skipping this:** duplicated logic, hard-coded values, playbooks that only work on one person's laptop, "works on the second run" bugs, leaked credentials, and fear of running automation in production.

---

## 2. Guiding Principles

Keep these in mind above all the specific rules that follow:

1. **Simplicity first** — Ansible is meant to be readable by humans. If it's clever, it's probably too complex.
2. **Declare state, don't script steps** — Describe *what* the system should look like, not *how* to get there.
3. **Idempotency always** — Running a playbook twice should change nothing the second time.
4. **Don't Repeat Yourself (DRY)** — Extract repeated logic into roles, variables, or loops.
5. **Convention over configuration** — Use Ansible's standard directory layouts so tooling and people know where to look.
6. **Explicit over implicit** — Name everything, set defaults, and be clear about privilege escalation.
7. **Treat it like code** — Version control, code review, linting, and testing apply to Ansible too.

---

## 3. Project Structure

Use a consistent, predictable layout. A recommended structure:

```text
ansible-project/
├── ansible.cfg                 # Project-level configuration
├── requirements.yml            # Collection and role dependencies
├── README.md
├── .ansible-lint               # Linter configuration
├── .yamllint                   # YAML linter configuration
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   └── webservers.yml
│   │   └── host_vars/
│   │       └── web01.example.com.yml
│   └── staging/
│       ├── hosts.yml
│       ├── group_vars/
│       └── host_vars/
├── playbooks/
│   ├── site.yml                # Master playbook — imports the others
│   ├── webservers.yml
│   └── databases.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   └── postgresql/
├── collections/                # Locally installed collections (usually git-ignored)
└── filter_plugins/             # Custom plugins, if needed
```

**Why:** Separate inventories per environment prevent accidental production changes, and keeping playbooks thin while logic lives in roles makes everything reusable.

### `ansible.cfg`

Keep a project-level `ansible.cfg` so behaviour is consistent for everyone:

```ini
[defaults]
inventory           = inventories/staging/hosts.yml
roles_path          = roles
collections_path    = collections
host_key_checking   = True
retry_files_enabled = False
stdout_callback     = yaml
forks               = 20
interpreter_python  = auto_silent

[ssh_connection]
pipelining = True
```

> Default the inventory to **staging**, never production. Running against production should be a deliberate act (`-i inventories/production`).

---

## 4. Inventory

- **Use YAML inventories** — they're more readable and support nested structure better than INI.
- **One inventory per environment** — `production`, `staging`, `dev`.
- **Group by function and location**, not by individual host.
- **Use dynamic inventory** for cloud environments (AWS, Azure, GCP, VMware) instead of maintaining static lists.

```yaml
# inventories/production/hosts.yml
all:
  children:
    webservers:
      hosts:
        web01.example.com:
        web02.example.com:
    databases:
      hosts:
        db01.example.com:
    datacenter_london:
      children:
        webservers:
        databases:
```

- Put variables in `group_vars/` and `host_vars/`, **not** inline in the inventory file.

---

## 5. Variables

Variables are where most Ansible projects become messy. Be disciplined.

### Naming

- Use **`snake_case`** only — lowercase letters, numbers and underscores.
- **Name role variables `role_name__variable_name`**: the role name, a **double underscore**, then the variable name.
- Use descriptive names; avoid abbreviations.

```yaml
# Good
nginx__worker_processes: 4
nginx__listen_port: 80
postgresql__max_connections: 200
web_server__document_root: /var/www/html

# Bad
port: 80                        # Which service?
nginx_listen_port: 80           # Single underscore: where does the role name end?
WorkerProcs: 4                  # Wrong case
pg_mc: 200                      # Unclear abbreviation
```

**Why the double underscore:** role names and variable names often both contain underscores. With a single underscore you can't tell whether `web_server_document_root` belongs to a `web` role or a `web_server` role. With `web_server__document_root` the split is obvious. It also means:

- You can find every variable for a role with one search (`grep -r 'web_server__'`).
- Role variables never clash with each other, with other roles, or with Ansible's own variables.
- When reading a playbook or `group_vars` file, you know straight away which role a value configures.

**Rules:**

- The prefix must match the role's folder name exactly.
- Use it for **every** variable the role defines or uses: `defaults/`, `vars/`, `register:`, `set_fact`, and loop variables (`loop_control: loop_var:`).
- Variables that aren't tied to a role (for example shared values in `group_vars/all`) don't take a role prefix.

### Where to define variables

Ansible has [22 levels of variable precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence). Keep it simple by using only a few:

| Location | Use for |
|---|---|
| `roles/<role>/defaults/main.yml` | Sensible, overridable defaults for every role variable. **Most variables live here.** |
| `group_vars/` | Environment or group-specific values. |
| `host_vars/` | Values unique to a single host (use sparingly). |
| `roles/<role>/vars/main.yml` | Internal constants that should *not* be overridden (e.g. OS-specific package names). |
| `--extra-vars` | One-off runtime overrides only. |

> **Rule of thumb:** If you find yourself fighting precedence, your variable design is too complicated.

### Other variable tips

- **Every variable a role uses should have a default** in `defaults/main.yml`, even if it's empty — this documents the role's interface.
- **Never hard-code values** in tasks that might differ between environments.
- **Quote strings** that could be misinterpreted (`"yes"`, `"no"`, `"on"`, `"0755"`, version numbers like `"1.10"`).
- Use the `| default()` filter for optional values and `| mandatory` for required ones.
- Registered results follow the same format, e.g. `register: nginx__config_result`.

---

## 6. Playbooks

Playbooks should be **thin** — they map hosts to roles and little else.

```yaml
# playbooks/webservers.yml
---
- name: Configure web servers
  hosts: webservers
  become: true

  roles:
    - role: common
    - role: nginx
      vars:
        nginx__listen_port: 8080
```

A master playbook ties everything together:

```yaml
# playbooks/site.yml
---
- name: Configure web tier
  ansible.builtin.import_playbook: webservers.yml

- name: Configure database tier
  ansible.builtin.import_playbook: databases.yml
```

### Guidelines

- **Always name every play.** Names appear in output and serve as documentation.
- **Start every YAML file with `---`.**
- **Set `become` explicitly** at the play or task level — only escalate privileges where required.
- **Avoid long lists of tasks in playbooks** — move them into a role.
- Use **tags** to allow partial runs (`--tags nginx`). See [Section 9](#9-tags).

### Pre-tasks and post-tasks

`pre_tasks` and `post_tasks` let a play do work **before** and **after** its roles, without putting that work inside a reusable role where it doesn't belong.

**Execution order in a play:**

```text
1. pre_tasks
2. handlers notified by pre_tasks
3. roles
4. tasks
5. handlers notified by roles and tasks
6. post_tasks
7. handlers notified by post_tasks
```

> Note the handler flush after `pre_tasks`. A handler notified in `pre_tasks` runs **before** any role starts, which is useful when, for example, a package cache refresh must finish first.

**Good uses for `pre_tasks`:**

- Checking preconditions with `ansible.builtin.assert` (correct OS, enough disk space, required variables set)
- Taking a host out of a load balancer or monitoring before changing it
- Waiting for a host to become reachable (`ansible.builtin.wait_for_connection`)
- Refreshing package caches once, rather than in every role

**Good uses for `post_tasks`:**

- Smoke tests that check the service actually works (`ansible.builtin.uri`)
- Putting the host back into the load balancer and re-enabling monitoring
- Sending a notification that the deployment has finished

**Example: rolling update with a load balancer:**

```yaml
# playbooks/webservers.yml
---
- name: Rolling update of web servers
  hosts: webservers
  become: true
  serial: 2

  pre_tasks:
    - name: Check the host is a supported OS
      ansible.builtin.assert:
        that:
          - ansible_facts['os_family'] == "RedHat"
        fail_msg: "This playbook only supports RedHat-family hosts."
      tags: always

    - name: Disable host in the load balancer
      community.general.haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        backend: web_backend
      delegate_to: "{{ item }}"
      loop: "{{ groups['loadbalancers'] }}"
      tags: always

  roles:
    - role: common
      tags: common
    - role: nginx
      tags: nginx

  post_tasks:
    - name: Check the site responds
      ansible.builtin.uri:
        url: "http://{{ inventory_hostname }}/health"
        status_code: 200
      register: webservers__health_check
      retries: 5
      delay: 5
      until: webservers__health_check.status == 200
      tags: always

    - name: Enable host in the load balancer
      community.general.haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        backend: web_backend
      delegate_to: "{{ item }}"
      loop: "{{ groups['loadbalancers'] }}"
      tags: always
```

**Guidelines:**

- **Keep pre/post tasks short.** They're for play-level setup and teardown. If they grow past a handful of tasks, or you copy them between playbooks, move them into a role (e.g. `lb_drain`) or a shared task file.
- **Keep host-specific steps out of roles.** A role like `nginx` shouldn't know about your load balancer. Draining and re-enabling the host belongs in the playbook's `pre_tasks` and `post_tasks`, so the role stays reusable.
- **Tag safety steps with `always`.** If you run `--tags nginx`, untagged pre/post tasks are **skipped**. That means the role would restart nginx while the host is still taking live traffic. Tag load balancer, assert, and health check steps with `always` so they run on every partial run.
- **Pre/post tasks keep the play's `serial` batching**, so each batch is drained, updated, checked and re-enabled before the next batch starts.
- Pre/post tasks belong to the playbook, not a role, so they don't need a task file prefix. Their registered variables still use a prefix to avoid clashes (`webservers__health_check` above).

---

## 7. Roles

Roles are the primary unit of reuse in Ansible. Design them like you would a function or a library.

### Standard role layout

Generate with `ansible-galaxy role init <role_name>`:

```text
roles/nginx/
├── README.md            # What it does, variables, examples
├── defaults/main.yml    # Default variables (public interface)
├── vars/main.yml        # Internal constants
├── tasks/
│   ├── main.yml         # Entry point — includes the files below
│   ├── install.yml
│   ├── configure.yml
│   └── service.yml
├── handlers/main.yml
├── templates/
│   └── nginx.conf.j2
├── files/
├── meta/
│   ├── main.yml         # Metadata, platforms, dependencies
│   └── argument_specs.yml
└── molecule/            # Tests
    └── default/
```

### Role design principles

- **Single responsibility** — one role, one purpose. `nginx` installs and configures nginx; it doesn't also deploy your application.
- **Self-contained** — a role should work on its own with only its defaults.
- **Configurable through variables**, not by editing the role.
- **Avoid hard dependencies** in `meta/main.yml` where possible; let playbooks compose roles explicitly.
- **Support multiple OS families** by loading OS-specific variables:

```yaml
# roles/nginx/tasks/main.yml
---
- name: main | Load OS-specific variables
  ansible.builtin.include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
    - "{{ ansible_facts['os_family'] }}.yml"
    - default.yml

- name: main | Install nginx
  ansible.builtin.import_tasks: install.yml

- name: main | Configure nginx
  ansible.builtin.import_tasks: configure.yml
```

### Validate inputs with argument specs

`meta/argument_specs.yml` validates role inputs automatically and documents them:

```yaml
---
argument_specs:
  main:
    short_description: Install and configure nginx
    options:
      nginx__listen_port:
        type: int
        default: 80
        description: Port nginx listens on.
      nginx__server_name:
        type: str
        required: true
        description: The primary server name.
```

---

## 8. Tasks

### Always name tasks

Names should describe the **desired state**, starting with a capital letter.

```yaml
# Good
- name: Ensure nginx is installed
  ansible.builtin.package:
    name: nginx
    state: present

# Bad — no name, short-form syntax
- package: name=nginx state=present
```

### Prefix role task names with their file name

In roles, start each task name with the name of the task file it lives in (without `.yml`), then ` | `, then the description:

```yaml
# roles/webserver/tasks/install_software.yml
---
- name: install_software | Install httpd
  ansible.builtin.package:
    name: httpd
    state: present

- name: install_software | Install mod_ssl
  ansible.builtin.package:
    name: mod_ssl
    state: present
```

```yaml
# roles/webserver/tasks/main.yml
---
- name: main | Install software
  ansible.builtin.import_tasks: install_software.yml

- name: main | Configure httpd
  ansible.builtin.import_tasks: configure.yml
```

When the role runs, the output shows exactly where each task comes from:

```text
TASK [webserver : install_software | Install httpd] ****************************
changed: [web01.example.com]

TASK [webserver : configure | Deploy httpd.conf] *******************************
ok: [web01.example.com]
```

**Why:** Ansible shows the role name in the output but not the file. In a large role with many task files, the prefix tells you straight away which file to open when a task fails, is slow, or reports an unexpected change. It also makes `--start-at-task` and `--list-tasks` output easier to read.

**Rules:**

- Use the file name exactly as it is (`install_software`, not `Install Software`), so it matches what you'd search for.
- Use ` | ` (space, pipe, space) as the separator every time.
- Apply it to every task file in a role, including `main.yml` and any nested folders (e.g. `tasks/os/redhat.yml` → `redhat | ...`).
- Rename the prefix if you rename or move the file. A wrong prefix is worse than none.
- Tasks written directly in a playbook (not in a role) don't need a prefix. The play name already gives context.

> **Enforce it with ansible-lint:** the opt-in `name[prefix]` rule checks for this same `<file> | ` format (it skips `main.yml`, so check those in code review):
>
> ```yaml
> # .ansible-lint
> enable_list:
>   - name[prefix]
> ```

### Use Fully Qualified Collection Names (FQCN)

```yaml
# Good
- name: Copy configuration file
  ansible.builtin.copy:
    src: app.conf
    dest: /etc/app/app.conf

# Avoid
- name: Copy configuration file
  copy:
    src: app.conf
    dest: /etc/app/app.conf
```

**Why:** FQCNs remove ambiguity about which module runs and are required by modern `ansible-lint` profiles.

### Use native YAML syntax, not `key=value`

```yaml
# Good
- name: Create application user
  ansible.builtin.user:
    name: appuser
    shell: /bin/bash
    groups: www-data
    append: true

# Bad
- name: Create application user
  ansible.builtin.user: name=appuser shell=/bin/bash groups=www-data append=yes
```

### Prefer modules over `command` / `shell`

Modules are idempotent, cross-platform, and report changes accurately. Reach for `command`/`shell` only when no module exists.

```yaml
# Good
- name: Ensure app directory exists
  ansible.builtin.file:
    path: /opt/app
    state: directory
    owner: appuser
    mode: "0755"

# Bad
- name: Create app directory
  ansible.builtin.shell: mkdir -p /opt/app && chown appuser /opt/app
```

If you must use `command`/`shell`, make it idempotent:

```yaml
- name: Initialise the database
  ansible.builtin.command: /opt/app/bin/init-db
  args:
    creates: /opt/app/.db_initialised

- name: Check current schema version
  ansible.builtin.command: /opt/app/bin/schema-version
  register: schema_version
  changed_when: false
```

### Formatting conventions

- **2-space indentation**, no tabs.
- **Use `true` / `false`** for booleans — not `yes`, `no`, `True`, `on`.
- **Always quote file modes** — `mode: "0644"`, never `mode: 644`.
- **Spaces inside Jinja braces** — `"{{ my_var }}"`, not `"{{my_var}}"`.
- **Quote any value starting with `{{`.**
- Keep lines under ~160 characters; use YAML folding (`>`) for long strings.
- Order task keys consistently: `name`, then `when`/`loop`/`become`, then the module, then `register`, `notify`, `tags`.

### Use loops instead of repetition

```yaml
# Good
- name: Ensure required packages are installed
  ansible.builtin.package:
    name: "{{ common__packages }}"
    state: present

# Bad
- name: Install git
  ansible.builtin.package:
    name: git
- name: Install curl
  ansible.builtin.package:
    name: curl
- name: Install vim
  ansible.builtin.package:
    name: vim
```

> Many package modules accept a list directly — this is faster than looping.

### `import_*` vs `include_*`

| | `import_tasks` / `import_role` | `include_tasks` / `include_role` |
|---|---|---|
| When processed | Parse time (static) | Runtime (dynamic) |
| Tags & `--list-tasks` | Fully supported | Limited |
| Loops & dynamic filenames | Not supported | Supported |
| **Use when** | Default choice | You need conditionals on the whole file, loops, or variable file names |

---

## 9. Tags

Tags let you run or skip part of a playbook without editing it, e.g. pushing only a config change rather than running the full build. They're a speed tool. Used carelessly, they make runs unpredictable.

```bash
ansible-playbook playbooks/site.yml --tags nginx              # Only nginx
ansible-playbook playbooks/site.yml --tags nginx__configure   # Only nginx config
ansible-playbook playbooks/site.yml --skip-tags common        # Everything except common
ansible-playbook playbooks/site.yml --list-tags               # Show every tag
ansible-playbook playbooks/site.yml --tags nginx --list-tasks # Preview what would run
```

### A consistent tagging scheme

Use two levels of tags:

| Level | Where applied | Format | Example |
|---|---|---|---|
| **Role** | On the role in the playbook | `role_name` | `nginx` |
| **Task file** | On each `import_tasks` in the role's `tasks/main.yml` | `role_name__task_file` | `nginx__install_software`, `nginx__configure` |

The task file tag uses the same `role_name__` prefix as variables, and the same name as the task file, so a tag, a variable prefix, and the `task_file |` prefix in the output all point to the same place.

```yaml
# playbooks/webservers.yml
roles:
  - role: nginx
    tags: nginx
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: main | Install software
  ansible.builtin.import_tasks: install_software.yml
  tags: nginx__install_software

- name: main | Configure nginx
  ansible.builtin.import_tasks: configure.yml
  tags: nginx__configure
```

**Why role-prefixed task file tags:** tags are combined with OR, not AND. `--tags nginx,configure` runs *all* of nginx plus *every* role's `configure` tasks, not just nginx's config. A tag like `nginx__configure` targets exactly one thing.

### How tags are inherited

This is the most common source of tag bugs:

| Applied to | Do tasks inside get the tag? |
|---|---|
| `roles:` entry in a play | Yes, every task in the role |
| `import_tasks` / `import_role` / `import_playbook` | Yes, every task inside |
| `block` | Yes, every task in the block |
| `include_tasks` / `include_role` | **No.** Only the include statement itself is tagged |

To pass tags through a dynamic include, use `apply`:

```yaml
- name: main | Configure OS-specific settings
  ansible.builtin.include_tasks:
    file: "{{ ansible_facts['os_family'] | lower }}.yml"
    apply:
      tags: nginx__configure
  tags: nginx__configure   # Needed too, so the include itself runs
```

### Special tags

| Tag | Behaviour | Use for |
|---|---|---|
| `always` | Runs on every run, unless `--skip-tags always` is used | Asserts, load balancer drain/enable, loading variables, facts or registered values other tasks depend on |
| `never` | Never runs unless asked for by another tag on the same task | Dangerous or occasional tasks: wiping data, forced reinstalls, debug output |

```yaml
- name: maintenance | Remove all cached data
  ansible.builtin.file:
    path: /var/cache/nginx
    state: absent
  tags:
    - never
    - nginx__clear_cache   # Runs only with --tags nginx__clear_cache
```

### Tagging guidelines

- **A full run with no tags must always work.** Tags are a shortcut, never a requirement.
- **Tag at the role and import level, not on every task.** Tagging each task by hand is noisy and quickly becomes inconsistent.
- **Watch for hidden dependencies.** If a tagged task uses a variable registered by an untagged task, a partial run will fail or behave wrongly. Tag the task it depends on as well, or with `always`.
- **Tag safety steps in `pre_tasks` / `post_tasks` with `always`** (see [Pre-tasks and post-tasks](#pre-tasks-and-post-tasks)).
- **Use lowercase `snake_case`**. No spaces, hyphens or capitals.
- **List a role's tags in its `README.md`** so people know what partial runs are possible.
- **Check before running** with `--list-tags` and `--list-tasks`, especially against production.

---

## 10. Idempotency

Idempotency means running the same playbook repeatedly produces the same end state, and reports **no changes** after the first successful run.

- **Use `state:` parameters** (`present`, `absent`, `started`) rather than actions.
- **Set `changed_when`** on read-only commands so they don't falsely report changes.
- **Use `creates` / `removes`** on `command` and `shell`.
- **Test it:** run the playbook twice — the second run should show `changed=0`. Molecule's `idempotence` step automates this.
- **Use check mode** (`--check --diff`) to preview changes before applying them.

```yaml
- name: Get list of installed plugins
  ansible.builtin.command: /usr/bin/app plugin list
  register: installed_plugins
  changed_when: false
  check_mode: false   # Safe to run even in --check mode

- name: Install missing plugin
  ansible.builtin.command: /usr/bin/app plugin install analytics
  when: "'analytics' not in installed_plugins.stdout"
```

---

## 11. Templates and Files

- Use **`templates/`** (with `.j2` extension) for files that contain variables; **`files/`** for static content.
- **Add a managed-by header** to templated files so people don't edit them by hand:

```jinja
# {{ ansible_managed }}
# Do not edit manually — changes will be overwritten.

worker_processes {{ nginx__worker_processes }};

events {
    worker_connections {{ nginx__worker_connections }};
}
```

- **Keep logic out of templates** where possible — compute values in variables, keep templates mostly presentational.
- **Validate configs before deploying** using the `validate` parameter:

```yaml
# roles/nginx/tasks/configure.yml
- name: configure | Deploy nginx configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    validate: nginx -t -c %s
  notify: Reload nginx
```

- Always set **`owner`, `group`, and `mode`** explicitly on files and templates.

---

## 12. Handlers

Handlers run once at the end of a play, only when notified — ideal for restarts and reloads.

```yaml
# roles/nginx/handlers/main.yml
---
- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded

- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

- **Name handlers clearly** and capitalise them — the `notify` value must match exactly.
- **Prefer `reloaded` over `restarted`** when the service supports it, to avoid downtime.
- Use `listen:` to let multiple handlers respond to one generic topic.
- Use `ansible.builtin.meta: flush_handlers` if a service must be restarted before later tasks run.

---

## 13. Secrets Management

**Never commit plaintext secrets to version control.**

### Ansible Vault

Encrypt sensitive variable files:

```bash
ansible-vault create inventories/production/group_vars/all/vault.yml
ansible-vault edit   inventories/production/group_vars/all/vault.yml
```

Use the **vault-prefix pattern** so variables stay discoverable with `grep`:

```yaml
# group_vars/all/vault.yml  (encrypted)
vault_db_password: "s3cr3t"

# group_vars/all/vars.yml   (plain text)
db_password: "{{ vault_db_password }}"
```

### Other recommendations

- Prefer an **external secrets manager** (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, CyberArk) via lookup plugins for larger environments.
- Use **`no_log: true`** on tasks that handle secrets so they don't appear in output or logs.
- Store the vault password in a password manager or CI secret store — never in the repo. Use `--vault-password-file` or `--vault-id`.

```yaml
- name: Create database user
  community.postgresql.postgresql_user:
    name: app
    password: "{{ db_password }}"
  no_log: true
```

---

## 14. Collections and Dependencies

- **Declare all dependencies** in `requirements.yml` and **pin versions** for reproducible builds:

```yaml
---
collections:
  - name: community.general
    version: ">=9.0.0,<10.0.0"
  - name: ansible.posix
    version: "1.6.2"

roles:
  - name: geerlingguy.docker
    version: "7.4.1"
```

```bash
ansible-galaxy install -r requirements.yml
```

- **Consider Execution Environments** (`ansible-builder` + `ansible-navigator`) to package Ansible, collections, and Python dependencies into a consistent container image.
- Package frequently shared roles into an **internal collection** for distribution across teams.

---

## 15. Error Handling

Use `block` / `rescue` / `always` for structured error handling:

```yaml
- name: Deploy application with rollback
  block:
    - name: Deploy new release
      ansible.builtin.unarchive:
        src: "app-{{ app__version }}.tar.gz"
        dest: /opt/app/releases/

    - name: Switch symlink to new release
      ansible.builtin.file:
        src: "/opt/app/releases/{{ app__version }}"
        dest: /opt/app/current
        state: link

  rescue:
    - name: Roll back to previous release
      ansible.builtin.file:
        src: "/opt/app/releases/{{ app__previous_version }}"
        dest: /opt/app/current
        state: link

  always:
    - name: Clean up temporary files
      ansible.builtin.file:
        path: /tmp/app-deploy
        state: absent
```

- **Avoid `ignore_errors: true`** — it hides real failures. Use `failed_when` to define what failure actually means.
- **Use `ansible.builtin.assert`** to validate preconditions early and fail with a helpful message.
- **Use `serial` and `max_fail_percentage`** for rolling updates so a bad change doesn't take down every host at once.

```yaml
- name: Rolling update of web servers
  hosts: webservers
  serial: "25%"
  max_fail_percentage: 10
```

---

## 16. Performance

- **Enable SSH pipelining** (`pipelining = True`) — large speed improvement.
- **Increase `forks`** for large inventories.
- **Disable fact gathering** (`gather_facts: false`) on plays that don't need facts, or use `gather_subset` to collect only what's needed.
- **Enable fact caching** (e.g. `jsonfile` or `redis`) for repeated runs.
- **Pass lists to package modules** rather than looping one package at a time.
- Use `async` / `poll` for long-running tasks.
- Use `strategy: free` where hosts don't need to stay in lockstep.
- Profile slow playbooks with the `ansible.posix.profile_tasks` callback.

---

## 17. Testing and Linting

Untested automation is a liability. Build quality checks into your workflow.

| Tool | Purpose |
|---|---|
| **`ansible-lint`** | Enforces best practices, catches deprecated syntax and common mistakes. |
| **`yamllint`** | Checks YAML syntax and formatting. |
| **`ansible-playbook --syntax-check`** | Quick syntax validation. |
| **`--check --diff`** | Dry run showing what would change. |
| **Molecule** | Full role testing in containers/VMs: converge, idempotence, verify. |
| **`ansible-test`** | Sanity and unit tests for collections. |
| **pre-commit** | Runs linters automatically before each commit. |

### Example `.ansible-lint`

```yaml
---
profile: production
enable_list:
  - name[prefix]   # Role task names must start with "<task file> | "
exclude_paths:
  - .cache/
  - collections/
```

### Example `.pre-commit-config.yaml`

```yaml
---
repos:
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
  - repo: https://github.com/ansible/ansible-lint
    rev: v24.12.2
    hooks:
      - id: ansible-lint
```

> Update the `rev` values to the latest releases when you set this up.

### Molecule test cycle

```bash
cd roles/nginx
molecule test   # create → converge → idempotence → verify → destroy
```

---

## 18. Version Control and CI/CD

- **Everything in Git** — playbooks, roles, inventories, and configuration.
- **Use a `.gitignore`** for `*.retry`, `collections/`, `.cache/`, vault password files, and local overrides.
- **Use branches and pull requests** — all changes reviewed before merging.
- **Run linting and Molecule tests in CI** on every pull request.
- **Tag releases** of roles and collections using semantic versioning.
- **Deploy from CI/CD or AWX / Ansible Automation Platform**, not from individual laptops, for consistency and an audit trail.
- **Promote changes through environments**: dev → staging → production.
- **Write meaningful commit messages** explaining *why* a change was made.

---

## 19. Documentation

- **Every role gets a `README.md`** covering:
  - What the role does
  - Supported platforms
  - All variables, with defaults and descriptions
  - Dependencies
  - An example playbook
- **Comment the *why*, not the *what*** — well-named tasks already explain what they do.
- **Keep a top-level `README.md`** explaining the project layout, how to install dependencies, and how to run playbooks.
- Use `meta/argument_specs.yml` — it doubles as machine-readable documentation.

```yaml
# Good comment — explains reasoning
# Kernel 5.x requires this sysctl for the app's connection pooling to work
- name: Increase maximum socket backlog
  ansible.posix.sysctl:
    name: net.core.somaxconn
    value: "4096"

# Bad comment — repeats the task name
# Set somaxconn
```

---

## 20. Quick Reference Checklist

Use this during code review.

### Structure
- [ ] Follows the standard project and role directory layout
- [ ] Separate inventories per environment
- [ ] Playbooks are thin; logic lives in roles
- [ ] Each role has a single responsibility
- [ ] Play-level setup/teardown (asserts, load balancer, smoke tests) is in `pre_tasks` / `post_tasks`, not inside roles

### Tags
- [ ] Roles tagged with their name in the playbook; task file imports tagged `role_name__task_file`
- [ ] Tags passed through `include_*` with `apply`
- [ ] Safety steps in pre/post tasks tagged `always`
- [ ] Destructive or occasional tasks tagged `never`
- [ ] Playbook still works with no tags

### Readability
- [ ] Every play, task, and handler has a descriptive name
- [ ] Role task names start with their task file name (`install_software | Install httpd`)
- [ ] Fully Qualified Collection Names (FQCN) used for all modules
- [ ] Native YAML syntax — no `key=value` shorthand
- [ ] Consistent 2-space indentation
- [ ] Booleans are `true` / `false`
- [ ] File modes are quoted strings

### Variables
- [ ] All variables use `snake_case`
- [ ] Role variables use `role_name__variable_name` (double underscore)
- [ ] Every role variable has a default in `defaults/main.yml`
- [ ] No hard-coded environment-specific values

### Reliability
- [ ] Modules preferred over `command` / `shell`
- [ ] `command` / `shell` tasks use `creates`, `removes`, or `changed_when`
- [ ] Second run reports `changed=0`
- [ ] Config files validated before deployment
- [ ] No `ignore_errors: true` without strong justification

### Security
- [ ] No plaintext secrets in the repository
- [ ] Secrets encrypted with Vault or pulled from a secrets manager
- [ ] `no_log: true` on tasks that handle sensitive data
- [ ] `become` used only where required

### Quality
- [ ] Passes `ansible-lint` and `yamllint`
- [ ] Molecule tests pass (for roles)
- [ ] Dependencies pinned in `requirements.yml`
- [ ] README updated

---

## Further Reading

- [Ansible Documentation — Tips and Tricks](https://docs.ansible.com/ansible/latest/tips_tricks/index.html)
- [Ansible Lint Documentation](https://ansible.readthedocs.io/projects/lint/)
- [Molecule Documentation](https://ansible.readthedocs.io/projects/molecule/)
- [Red Hat Community of Practice — Good Practices for Ansible](https://redhat-cop.github.io/automation-good-practices/)
