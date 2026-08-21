# lab-devspaces-ansible-exercise2

## What is an Ansible role?

A **role** is a reusable unit that groups tasks, data, templates, and metadata under a **name** (`wildfly_os_deps`, `nginx`, etc.). The playbook only declares which roles to apply and in which order; Ansible automatically loads each role’s conventional files (`tasks/main.yml`, `defaults/main.yml`, etc.). That way you avoid huge playbooks, you can **version and share** a role in Git (or Galaxy), and **reuse it** across several projects without copy-pasting YAML.

You do not have to use every folder: many roles only have `tasks/`, `defaults/`, and `meta/`. Folders that do not exist are ignored.

### Typical role structure (`tree` view)

```text
role_name/
├── defaults/
│   └── main.yml           # default variables (low precedence)
├── files/
│   └── example.conf       # static files for copy/fetch
├── handlers/
│   └── main.yml           # actions triggered with notify (restarts, reloads)
├── meta/
│   └── main.yml           # role dependencies and Galaxy information
├── tasks/
│   └── main.yml           # main role tasks
├── templates/
│   └── app.conf.j2        # Jinja2 templates for the template module
├── vars/
│   └── main.yml           # role-internal variables (high precedence)
└── README.md              # human documentation (optional but recommended)
```

### Folders and files: what they are for and a minimal example

#### `tasks/main.yml`

List of **tasks** Ansible runs when applying the role. This is the core of the role.

```yaml
---
- name: Asegurar que un paquete está instalado
  ansible.builtin.dnf:
    name: httpd
    state: present
```

#### `handlers/main.yml`

**Handlers**: tasks that only run when a previous task **notifies** them (`notify`), usually at the end of the play. They avoid restarting a service on every run if nothing changed.

```yaml
---
- name: recargar httpd
  ansible.builtin.service:
    name: httpd
    state: reloaded
```

#### `defaults/main.yml`

Variables with **lowest precedence**: default values that the playbook, inventory, or play `vars` can override without touching the role code.

```yaml
---
httpd_port: 80
httpd_package: httpd
```

#### `vars/main.yml`

Variables with **higher precedence** than `defaults`; they are usually role constants or values you do not want the user to change without editing the role.

```yaml
---
httpd_config_path: /etc/httpd/conf/httpd.conf
```

#### `files/`

**Static** files referenced in tasks with short paths: `ansible.builtin.copy: src: motd` looks for `files/motd` inside the role.

Example: a `files/motd` file with a single line of text that is copied to `/etc/motd` on the node.

#### `templates/`

**Jinja2** templates (`.j2` extension). The `template` module expands them at the destination substituting `{{ variables }}`.

Example in `templates/virtualhost.conf.j2`:

```jinja2
Listen {{ httpd_port }}
```

#### `meta/main.yml`

**Dependencies** between roles (Ansible can install or run other roles first) and metadata for documentation or **Ansible Galaxy**.

```yaml
---
dependencies:
  - role: common

galaxy_info:
  author: equipo
  description: Instala y configura Apache
  license: MIT
  min_ansible_version: "2.14"
```

#### `README.md`

Human documentation: what the role does, available variables, and a playbook usage example. Ansible does not interpret it, but it is essential in shared repos.

---

In this lab’s practice, **`tasks/`**, **`defaults/`**, and **`meta/`** will be enough for the sample role; **`handlers/`**, **`files/`**, **`templates/`**, and **`vars/`** come into play when the role grows or needs templates and coordinated restarts.

---

## What this lab consists of

**Exercise goal:** extract a role from the **`lab-devspaces-ansible-exercise2`** playbook into a **Git repository of its own** and **consume it from another playbook project** using `ansible-galaxy` and a **`requirements.yml`** file.

As a reference, the suggested role **`wildfly_os_deps`** is used (package installation: Java, `tar`, `gzip`) from the table in section **4.1** of the exercise2 README. The same steps work for `wildfly_account`, `wildfly_install`, `wildfly_bind`, `wildfly_systemd`, or `wildfly_sample_app`, adjusting tasks, variables, and dependencies between roles.

### Concrete objectives

1. Have the role code in an **independent Git repository**, with the standard Ansible role structure.
2. In the **playbook project** (for example the exercise2 one, or another), **do not** duplicate that tree under `roles/` in the playbook repository; instead **declare the source in `requirements.yml`** and install the role with `ansible-galaxy`.
3. The playbook still references the role by **name** (`wildfly_os_deps`) as if it were in `roles/`, but the content comes from Git.

---

## Prerequisites

- Have followed **exercise2** far enough to understand section **4** (suggested roles) or have already extracted at least one role locally in `roles/<role_name>/`.
- `git`, `ansible`, and `ansible-galaxy` available in the workspace (for example the Devfile container of this lab).
- A **Git remote** where you can publish the role: GitHub, GitLab, or another (HTTPS or SSH URL according to lab policy).

---

## Part A — Create the role repository (separate project)

### A.1 Create the empty repository on the forge

On your Git platform, create a new repository, for example `ansible-role-wildfly-os-deps`. **Do not** initialize with a README if you are going to push an already prepared local tree (avoids conflicts on the first push).

### A.2 Clone and structure at the repo root

Usual convention for a **single-role** repo: the **repository root is the role root** (not a `roles/wildfly_os_deps` subfolder inside the repo). That way `ansible-galaxy install -r requirements.yml` installs the content with the name you declare.

Minimum tree:

```text
ansible-role-wildfly-os-deps/
├── defaults/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
└── README.md          # optional but recommended
```

### A.3 Contents of `tasks/main.yml`

Migrate only the tasks that belong to this role. For `wildfly_os_deps`, equivalent to the dependencies block of the exercise2 monolithic playbook:

```yaml
---
- name: Instalar dependencias (Java 21+ para WildFly 39)
  ansible.builtin.dnf:
    name:
      - java-21-openjdk-devel
      - tar
      - gzip
    state: present
```

If the role combined more blocks, add the rest of the tasks here and use variable names consistent with `defaults/main.yml`.

### A.4 Contents of `defaults/main.yml`

Centralize default values (even if this particular role may not need variables, it is good practice to leave the file):

```yaml
---
# Default values for the wildfly_os_deps role (extend if you add variables to the tasks)
```

If you parameterize packages or Java versions, declare them here and use them in `tasks/main.yml` with `{{ variable_name }}`.

### A.5 `meta/main.yml` (role metadata)

Helps with documentation, compatibility, and future Galaxy publications:

```yaml
---
galaxy_info:
  author: tu_usuario_o_equipo
  description: Dependencias de sistema para WildFly (Java, tar, gzip)
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Fedora
      versions:
        - all
  galaxy_tags:
    - wildfly
    - system

dependencies: []
```

Adjust `author`, `license`, and `platforms` to what your organization requires.

### A.6 First commit and push

```bash
cd ansible-role-wildfly-os-deps
git init
git add defaults meta tasks README.md
git commit -m "Initial import: wildfly_os_deps role"
git remote add origin <GIT_REPOSITORY_URL>
git branch -M main
git push -u origin main
```

For stable versions, create **tags** (`v1.0.0`) and use the tag in `requirements.yml` (see part B).

---

## Part B — Consume the role from the playbook project

Work in the **playbook** directory (for example `lab-devspaces-ansible-exercise2` or a copy with only inventory and playbooks).

### B.1 Remove the duplicated role from the playbook repo (recommended)

If you already had `roles/wildfly_os_deps/` in the same repo as the playbook:

- Delete that folder from the playbook repository (or stop versioning it) so the only source of truth is the role Git repo.
- Make sure `.gitignore` **does not** ignore `roles/` if you are going to version other local roles; if you only use roles installed by Galaxy, you can ignore `roles/wildfly_os_deps` after installing, or not version `roles/` and document that `ansible-galaxy install` must be run after clone (common strategy).

### B.2 Create `requirements.yml` at the playbook project root

`requirements.yml` file (**roles** only in this example):

```yaml
---
roles:
  - name: wildfly_os_deps
    src: git+<GIT_REPOSITORY_URL>
    scm: git
    version: main
```

Replace:

- `<GIT_REPOSITORY_URL>` with the cloneable URL (HTTPS or SSH). Examples:
  - `git+https://github.com/org/ansible-role-wildfly-os-deps.git`
  - `git+git@gitlab.example.com:ansible/ansible-role-wildfly-os-deps.git`
- `version` with the branch (`main`, `develop`) or with a **tag** (`v1.0.0`) for reproducible installs.

Equivalent alternative format (without `git+` prefix in some environments):

```yaml
---
roles:
  - name: wildfly_os_deps
    src: https://github.com/org/ansible-role-wildfly-os-deps.git
    scm: git
    version: main
```

### B.3 Install roles in the playbook project

From the directory where `requirements.yml` is:

```bash
ansible-galaxy install -r requirements.yml
```

By default roles are installed in `~/.ansible/roles` or in the path defined by `roles_path` in `ansible.cfg`. To place them **next to the playbook** in `./roles/` (typical in lab projects), use:

```bash
ansible-galaxy install -r requirements.yml --roles-path ./roles
```

Check that `roles/wildfly_os_deps/tasks/main.yml` exists (or another path under `roles_path`).

### B.4 Configure `ansible.cfg` (optional but clear)

If you always use `--roles-path ./roles`, you can set it in `ansible.cfg` at the playbook project root:

```ini
[defaults]
roles_path = ./roles
```

That way `ansible-playbook` resolves the role by name without extra flags.

### B.5 Playbook that references the role

The play does not change compared to having the role “at hand” in the repo: the name must match `name` in `requirements.yml`.

```yaml
---
- name: Instalación de WildFly con rol externo (ejemplo parcial)
  hosts: servers
  become: true
  roles:
    - role: wildfly_os_deps
    # - role: wildfly_account
    # - role: wildfly_install
    # ... remaining local roles or also in requirements.yml
```

If the other roles remain only in the playbook repo, leave them in local `roles/`; mixing **Galaxy/Git** roles and **local** roles is common.

### B.6 Verification

```bash
ansible-playbook -i inventory deploy-wildfly.yaml --syntax-check
ansible-playbook -i inventory deploy-wildfly.yaml --check
```

(Adjust the playbook name if you use one that only includes the test role.)

---

## Step summary (checklist)

| Step | Where | Action |
|------|--------|--------|
| 1 | Git forge | Create the role repo (e.g. `ansible-role-wildfly-os-deps`). |
| 2 | Role repo | `tasks/`, `defaults/`, `meta/` structure at the root. |
| 3 | Role repo | Migrate tasks and variables from the chosen exercise2 block. |
| 4 | Role repo | `git commit` and `git push` (and tags if you version). |
| 5 | Playbook repo | Add `requirements.yml` with `name`, `src`, `scm`, `version`. |
| 6 | Playbook repo | `ansible-galaxy install -r requirements.yml --roles-path ./roles`. |
| 7 | Playbook repo | Optional: `ansible.cfg` → `roles_path = ./roles`. |
| 8 | Playbook repo | Playbook with `roles: [ wildfly_os_deps, ... ]`. |
| 9 | — | `ansible-playbook` with a correct inventory. |

---

## Practical notes

- **Private roles:** with HTTPS you usually need a token or credential helper; with SSH, a key in the workspace and `src: git+git@...`.
- **Dependency order:** if later `wildfly_install` depends on variables defined in another role, use `dependencies` in the consuming role’s `meta/main.yml` or keep the order in the playbook `roles:`.
- **CI/CD:** in a pipeline, always run `ansible-galaxy install -r requirements.yml` before `ansible-playbook`.
- **Same role for several playbooks:** several playbook repositories can point at the same `src` and `version` in their own `requirements.yml`.

---

## Expected result

- A **Git repository** contains a reusable Ansible role with standard structure.
- The **playbook project** declares that role in **`requirements.yml`**, installs it with **`ansible-galaxy`**, and the playbook invokes it by **name** without copying the role code into the playbook repo (unless you choose to version `roles/` after installing; the usual approach is not to version generated artifacts and to version `requirements.yml`).
