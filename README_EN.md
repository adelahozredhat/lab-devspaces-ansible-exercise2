# lab-devspaces-ansible-exercise2

Hands-on course on **reusable Ansible roles**. **This exercise is designed to be completed inside OpenShift Dev Spaces**.

Image versions (the same as exercise 1; check them in the workspace):

| Tool | Version in Dev Spaces |
| ---- | --------------------- |
| Python | **3.12** |
| ansible-core | **2.21.x** |
| yamllint | **1.38.x** |
| ansible-lint | **26.x** (e.g. 26.6.0) |
| Molecule | **26.x** (e.g. 26.6.0); driver **`default`** (`delegated` does not exist) |

**What you must do:** **move** the `wildfly_os_deps` role content you created in **`lab-devspaces-ansible-exercise1`** into **this** repository (`lab-devspaces-ansible-exercise2`) and **call it from exercise 1** with `ansible-galaxy` and `requirements.yml`. Do not create an empty repository on the forge: use the exercise2 clone, which already has Git and `origin` on Gitea.

---

## What is an Ansible role?

A **role** is a reusable unit that groups tasks, data, templates, and metadata under a **name** (`wildfly_os_deps`, `nginx`, etc.). The playbook only declares which roles to apply and in which order; Ansible automatically loads each role’s conventional files (`tasks/main.yml`, `defaults/main.yml`, `vars/main.yml`, etc.). That way you avoid huge playbooks, you can **version and share** a role in Git (or Galaxy), and **reuse it** across several projects without copy-pasting YAML.

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
├── tests/
│   └── test.yml           # role test playbook
├── vars/
│   └── main.yml           # role constants (high precedence)
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
- name: Recargar httpd
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

Variables with **higher precedence** than `defaults`; they are **role constants** (values you do not want the user to change without editing the role).

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

#### `tests/`

Playbooks that **exercise the role** in isolation (syntax, `--check`, or a real run). Ansible does not run them by itself: you launch them with `ansible-playbook` (see part C).

#### `README.md`

Human documentation: what the role does, available variables, and a playbook usage example. Ansible does not interpret it.

In **this** repository `README.md` is the **lab guide**. Do not replace it or commit it as if it were the role README.

---

In this lab you will use **`tasks/`**, **`defaults/`**, **`vars/`**, **`meta/`**, and **`tests/`**. **`handlers/`**, **`files/`**, and **`templates/`** come into play when the role grows or needs templates and coordinated restarts.

---

## What this lab consists of

**Goal:** take the **`wildfly_os_deps`** role (Java, `tar`, `gzip` packages) from table **4.1 in the `lab-devspaces-ansible-exercise1` README**, **move its content** to the root of **`lab-devspaces-ansible-exercise2`**, and **invoke it again from exercise 1** (playbook + `requirements.yml` + `ansible-galaxy`).

The same steps work for `wildfly_account`, `wildfly_install`, `wildfly_bind`, `wildfly_systemd`, or `wildfly_sample_app`, adjusting tasks, variables, and dependencies.

### Concrete objectives

1. The role code lives in **`lab-devspaces-ansible-exercise2`** (repo root = role root).
2. In **`lab-devspaces-ansible-exercise1`** there is **no** versioned copy of `roles/wildfly_os_deps/`: **delete** the local role folder and declare the source in `requirements.yml`.
3. The exercise 1 playbook still uses the name `wildfly_os_deps`; the content comes from Git (this repo) via Galaxy.

---

## Prerequisites

- Have followed **`lab-devspaces-ansible-exercise1`** through **section 4** (suggested roles) and have at least `roles/wildfly_os_deps/` (or the equivalent block in the playbook) in that project.
- Work **in Dev Spaces**, with the versions in the table at the start (`git`, `ansible`, `ansible-galaxy`, `yamllint`, `ansible-lint`, Molecule).
- This clone is already a Git repository with `origin` on Gitea: **do not** create an empty repo or run `git init` / `git remote add`.

---

## Part A — Move the role into `lab-devspaces-ansible-exercise2`

### A.1 This repository is the role repository

**Do not** create an empty repository on the forge. You work in **`lab-devspaces-ansible-exercise2`**, which is already cloned and points at Gitea.

Convention: **this repo root is the role root** (not a `roles/wildfly_os_deps` subfolder). That way `ansible-galaxy install` installs the content under the `name` you declare in `requirements.yml`.

Minimum tree in **`lab-devspaces-ansible-exercise2/`**:

```text
lab-devspaces-ansible-exercise2/
├── .galaxy-ignore
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── tests/
│   └── test.yml
└── molecule/
    └── default/
        └── …   # part D
```

(`README.md`, `devfile.yaml`, and `.vscode/` already ship with the lab; leave them. Exclude them from Galaxy with `.galaxy-ignore`; see A.7. `tests/` and `molecule/` are created in parts C and D, not in the first A.7 commit.)

### A.2 Move the content from exercise 1

From the workspace, copy (or cut) what you have in exercise1 into the exercise2 root:

```bash
# Adjust the paths if your projects are not siblings
cp -a ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps/tasks ./tasks
cp -a ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps/defaults ./defaults
# if you already had meta/ or vars/ in exercise 1, copy them too
```

If in exercise 1 the role only existed as tasks inside the monolithic playbook, create the A.3–A.6 files from that block (OS dependencies: Java, `tar`, `gzip`).

**Then**, in exercise 1, **delete** the local copy so there is a single source of truth:

```bash
rm -rf ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps
```

The exercise 1 playbook will still call `wildfly_os_deps` after you install it with Galaxy (part B).

### A.3 Contents of `tasks/main.yml`

Use the tasks you moved. If you start from the exercise 1 dependencies block:

```yaml
---
- name: Instalar dependencias (Java 17+ es requerido para WF 39)
  ansible.builtin.dnf:
    name: "{{ wf_os_packages }}"
    state: present
```

### A.4 Contents of `defaults/main.yml`

Values the playbook **may** override (`group_vars`, play `vars:`):

```yaml
---
wf_java_package: java-25-openjdk-devel
```

### A.5 Contents of `vars/main.yml` (role constants)

Constants that **should not** be changed from the playbook. In this role, the unpack utilities are fixed; the JDK remains overridable via `defaults`:

```yaml
---
wf_os_unpack_packages:
  - tar
  - gzip
wf_os_packages: "{{ [wf_java_package] + wf_os_unpack_packages }}"
```

If your role has no constants, leave the file with a `---` header and a comment; the `vars/` folder documents the convention.

### A.6 `meta/main.yml` (role metadata)

```yaml
---
galaxy_info:
  author: your_lab_user
  role_name: wildfly_os_deps
  namespace: labuser
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

Replace `author` with your lab user (access data). `role_name` must match the `name` in `requirements.yml` (part B). `namespace` **cannot contain hyphens or uppercase letters**: Galaxy and ansible-lint reject values such as `lab-user-2`. Use something like `labuser` or `labuser2`.

If ansible-lint or Molecule still complain about `role-name` (the repo directory name has hyphens), add a `.ansible-lint` at the root of this repo:

```yaml
---
skip_list:
  - role-name
```

### A.7 Publish the role files

The remote **already exists**. Add only the role code that **already exists at this point** (A.3–A.6). **Do not** `git add tests molecule`: those folders are created in parts C and D; if they are missing, Git fails with `pathspec ... did not match any files`.

Create `.galaxy-ignore` so `ansible-galaxy install` **does not** copy the lab guide, `devfile`, or `.vscode` into `roles/wildfly_os_deps/` in exercise 1:

```text
README.md
README_EN.md
devfile.yaml
.vscode
.ansible
```

```bash
git add defaults vars meta tasks .galaxy-ignore
git commit -m "Add wildfly_os_deps role"
git push origin HEAD
```

After parts C and D, make another commit with `tests/`, `ansible.cfg`, and `molecule/`.

Note the **branch** you push (`master` on the lab clone unless you use another): you will need it as `version:` in `requirements.yml`.

---

## Part B — Call the role from `lab-devspaces-ansible-exercise1`

Work in the exercise 1 **playbook** directory (`lab-devspaces-ansible-exercise1`).

### B.1 Remove the local role from exercise 1

If `roles/wildfly_os_deps/` still exists in exercise1, delete it (see A.2). Other local roles (`wildfly_account`, and so on) can stay under `roles/`.

If you version the playbook, do not commit the `roles/wildfly_os_deps` folder produced by Galaxy: it is an `ansible-galaxy install` artifact.

### B.2 Create `requirements.yml` at the exercise1 root

**Replace** `<GITEA_HOST>` and `<GITEA_USER>` with values from **your lab access data** (Gitea URL and user, for example `lab-user-1`). Replace `master` if you pushed a different branch.

```yaml
---
roles:
  - name: wildfly_os_deps
    src: https://<GITEA_HOST>/<GITEA_USER>/lab-devspaces-ansible-exercise2.git
    scm: git
    version: master
```

Example `src` (each student fills in their own values):

```text
https://<GITEA_HOST>/<GITEA_USER>/lab-devspaces-ansible-exercise2.git
```

Do not copy a classmate’s URL: use **your** Gitea host and **your** user.

### B.3 Install the role in the playbook project

From `lab-devspaces-ansible-exercise1`:

```bash
ansible-galaxy install -r requirements.yml --roles-path ./roles
```

Check that `roles/wildfly_os_deps/tasks/main.yml` exists.

### B.4 Configure `ansible.cfg` (recommended)

At the exercise1 root:

```ini
[defaults]
roles_path = ./roles
```

### B.5 Complete playbook (external role + local roles)

In exercise 1 `deploy-wildfly.yaml` **do not comment out** the other roles. Only `wildfly_os_deps` is installed from Git (this exercise); **`wildfly_account`**, **`wildfly_install`**, **`wildfly_bind`**, **`wildfly_systemd`**, and **`wildfly_sample_app`** stay as the ones you already had under `roles/` in exercise 1. The run remains a **full WildFly installation**, not a partial play.

The Galaxy `name` must match `name` in `requirements.yml`:

```yaml
---
- name: Instalación de WildFly con roles
  hosts: servers
  become: true
  roles:
    - role: wildfly_os_deps      # external: Galaxy / this repo (exercise2)
    - role: wildfly_account      # local: exercise 1
    - role: wildfly_install
    - role: wildfly_bind
    - role: wildfly_systemd
    - role: wildfly_sample_app
```

If you later extract **another** role to Git, the pattern is the same: that role leaves local `roles/`, enters `requirements.yml`, and the others stay in the playbook so the order and the final result do not change.

### B.6 Verification from exercise 1

Inventory: use the IP from **your lab access data** (see the exercise 1 README).

```bash
ansible-playbook -i inventory deploy-wildfly.yaml --syntax-check
ansible-playbook -i inventory deploy-wildfly.yaml --check
ansible-playbook -i inventory deploy-wildfly.yaml
```

The real run must complete WildFly and `/sample/` as in exercise 1; `wildfly_os_deps` only changes **where** that role comes from.

---

## Part C — Role tests, yamllint, and ansible-lint

Do this **in `lab-devspaces-ansible-exercise2`**, against the role code (not the guide).

### C.1 Test playbook (`tests/test.yml`)

The play must call the role **by name** (`wildfly_os_deps`), not by a path (`{{ playbook_dir }}/..`). ansible-lint 26.x flags paths as `role-name[path]` and Molecule `prepare` would fail.

Because this repo **root is** the role (there is no `roles/wildfly_os_deps` subfolder), symlink the current directory under that name and set `roles_path` in `ansible.cfg`:

```bash
mkdir -p .ansible/roles
ln -sfn "$(pwd)" .ansible/roles/wildfly_os_deps
```

`ansible.cfg` at the root of **this** repository:

```ini
[defaults]
roles_path = .ansible/roles
host_key_checking = False
```

`.ansible/` is local (it is in `.galaxy-ignore`); you do not need to version it.

```yaml
---
- name: Test del rol wildfly_os_deps
  hosts: servers
  become: true
  roles:
    - role: wildfly_os_deps
```

### C.2 How to run it

Use the exercise 1 inventory (IP from your lab access data). The SSH key in that file is usually `ssh_tests_connections/id_fedora_new`: that path is **relative to the working directory**. If you run the playbook **from exercise2**, SSH cannot find the key (`No such file or directory`).

Fix the exercise 1 `inventory` so the key is anchored to `inventory_dir` (works from exercise1 and from exercise2):

```ini
[servers]
fedora-user1 ansible_host=<IP_from_your_lab_access_data> ansible_user=user1 ansible_ssh_private_key_file={{ inventory_dir }}/ssh_tests_connections/id_fedora_new
```

Replace host, IP, and user with **your** data. Then, from `lab-devspaces-ansible-exercise2`:

```bash
ansible-playbook -i ../lab-devspaces-ansible-exercise1/inventory tests/test.yml --syntax-check
ansible-playbook -i ../lab-devspaces-ansible-exercise1/inventory tests/test.yml --check
# real run (installs Java/tar/gzip on the VM):
ansible-playbook -i ../lab-devspaces-ansible-exercise1/inventory tests/test.yml
```

If your project folders are not siblings, adjust the `-i inventory` path.

### C.3 yamllint

Validate the **role** YAML (you do not need to lint this README or the lab `devfile.yaml`):

```bash
yamllint defaults vars meta tasks tests
```

If you prefer `yamllint .`, add a `.yamllint` that ignores `devfile.yaml`, this README, and `.cache/`. Fix warnings until exit code `0`.

### C.4 ansible-lint

Analyze the full role (tasks, defaults, vars, meta, and tests):

```bash
ansible-lint defaults vars meta tasks tests
```

Review **the full output** (not a single warning) and fix until exit code `0`. Re-run C.3 and C.4 if you change YAML.

---

## Part D — Molecule: test the role test playbook

Molecule runs `tests/test.yml` (part C) against a test machine: **create** → **prepare** (lint) → **converge** (the role test) → **verify** (only what **this** role does) → **destroy**.

The Dev Spaces image includes **Molecule 26.x** (e.g. 26.6.0) with ansible-core **2.21** and Python **3.12**. The driver is named `default` (the old name `delegated` no longer exists).

| Scenario | Machine | create / destroy | Where |
| -------- | ------- | ---------------- | ----- |
| `default` | **New** Fedora VM on OpenShift (KubeVirt) | Creates the VM at the start and **destroys** it at the end | **Only from Dev Spaces** |
| `with_existing_machine` | **Pre-started** lab Fedora (exercise 1 inventory) | Does not create or delete that VM | Dev Spaces, against your Fedora |

**What this role verifies:** Java (`java-25-openjdk-devel`), `tar`, and `gzip` packages. **Do not** check the `wildfly` service, port 8080, or `/sample/`: those belong to other exercise 1 roles.

If you extract **another** role (`wildfly_install`, `wildfly_systemd`, `wildfly_sample_app`, …), `tests/test.yml` and `verify.yml` must **first** include the prerequisite steps that role needs (for example user, tarball, and `standalone.xml` before systemd). This example only covers `wildfly_os_deps`, which does not depend on earlier roles.

Create the directories:

```bash
mkdir -p molecule/default molecule/with_existing_machine
```

### D.1 Scenario `default` — test VM on OpenShift (Dev Spaces only)

Same idea as **section 5.3.1 of exercise 1**: `create.yml` / `destroy.yml` / `molecule_vars.yml` provision and remove a Fedora VM on KubeVirt. Copy those three files from exercise1 or reproduce them the same way (driver `name: default`, `ocp_*` credentials from **your** lab access data, classroom `namespace`).

##### `molecule/default/molecule.yml`

```yaml
---
dependency:
  name: galaxy
driver:
  name: default
platforms:
  - name: fedora-chocolate-smelt-74
provisioner:
  name: ansible
  inventory:
    hosts:
      all:
        children:
          servers:
            hosts:
              fedora-chocolate-smelt-74: {}
    host_vars:
      fedora-chocolate-smelt-74:
        ansible_user: fedora
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
verifier:
  name: ansible
scenario:
  test_sequence:
    - destroy
    - create
    - prepare
    - converge
    - verify
    - destroy
```

##### `molecule/default/prepare.yml`

Lint **the role** (not `deploy-wildfly.yaml`):

```yaml
---
- name: Lint YAML and Ansible before converge
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    project_dir: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') }}"
  tasks:
    - name: Run yamllint on the role
      ansible.builtin.command:
        cmd: yamllint defaults vars meta tasks tests
        chdir: "{{ project_dir }}"
      changed_when: false

    - name: Run ansible-lint on the role
      ansible.builtin.command:
        cmd: ansible-lint defaults vars meta tasks tests
        chdir: "{{ project_dir }}"
      changed_when: false
```

##### `molecule/default/converge.yml`

Runs the role test playbook (part C), not the full WildFly playbook:

```yaml
---
- name: Converge
  ansible.builtin.import_playbook: ../../tests/test.yml
```

##### `molecule/default/verify.yml`

Assertions for **this** role only:

```yaml
---
- name: Verificar paquetes instalados por wildfly_os_deps
  hosts: servers
  become: true
  gather_facts: false
  vars:
    wf_java_package: java-25-openjdk-devel
  tasks:
    - name: Comprobar que Java, tar y gzip están instalados
      ansible.builtin.dnf:
        name:
          - "{{ wf_java_package }}"
          - tar
          - gzip
        state: present
      check_mode: true
      register: pkg_status
      failed_when: pkg_status.changed

    - name: Comprobar que java está en el PATH
      ansible.builtin.command:
        cmd: java -version
      changed_when: false
```

**Create / destroy:** the same `molecule/default/create.yml`, `destroy.yml`, and `molecule_vars.yml` as in exercise 1 (`oc` login, KubeVirt VM, delete at the end). The `default` scenario is run **only from Dev Spaces**.

### D.2 Scenario `with_existing_machine` — lab VM

Copy `prepare.yml`, `converge.yml`, and `verify.yml` from the `default` scenario. `create.yml` / `destroy.yml` must not create or power off the student’s Fedora (same pattern as exercise 1: a `debug` message only).

In `molecule.yml` use `driver.name: default` with `managed: false` and `ansible_host` from **your lab access data** (the IP in the exercise1 `inventory`; `127.0.0.1:2222` is only a tunnel example). The SSH key must be reachable **from exercise2**: point it at `../lab-devspaces-ansible-exercise1/ssh_tests_connections/id_fedora_new` (or use the same absolute/`inventory_dir` path as in C.2).

### D.3 Run Molecule

From the **`lab-devspaces-ansible-exercise2`** root:

```bash
# new VM on OpenShift; destroyed at the end (Dev Spaces only)
molecule test -s default

# lab Fedora; not deleted
molecule test -s with_existing_machine
```

Step by step (debugging), `default` scenario:

```bash
molecule create -s default
molecule prepare -s default
molecule converge -s default
molecule verify -s default
molecule destroy -s default
```

After `destroy` on `default`, that test VM **must no longer** exist in OpenShift. After `destroy` on `with_existing_machine`, the student’s Fedora **stays running**.

---

## Step summary (checklist)

| Step | Where | Action |
|------|--------|--------|
| 1 | exercise2 | `tasks/`, `defaults/`, `vars/`, `meta/`, `tests/` at the root of **this** repo. |
| 2 | exercise1 → exercise2 | **Move** `roles/wildfly_os_deps` content (or the equivalent block) into exercise2. |
| 3 | exercise1 | **Delete** local `roles/wildfly_os_deps`. |
| 4 | exercise2 | `.galaxy-ignore` + `git add` only what exists (`defaults vars meta tasks .galaxy-ignore`); `commit` and `git push origin HEAD`. `tests/` and `molecule/` in a later commit. |
| 5 | exercise1 | `requirements.yml` with `src: https://<GITEA_HOST>/<GITEA_USER>/lab-devspaces-ansible-exercise2.git` (**replace** host, user, and branch). |
| 6 | exercise1 | `ansible-galaxy install -r requirements.yml --roles-path ./roles`. |
| 7 | exercise1 | **Complete** playbook: external `wildfly_os_deps` + remaining **local** exercise 1 roles. |
| 8 | exercise2 | `ansible.cfg` + `.ansible/roles/wildfly_os_deps` symlink, `tests/test.yml` (role by **name**), inventory with `inventory_dir`, `yamllint` + `ansible-lint`. |
| 9 | exercise2 | Molecule: `converge` = role test; `verify` = Java/tar/gzip; `default` create/destroy on OpenShift (Dev Spaces only). |
| 10 | exercise1 | Full `ansible-playbook` (WildFly + `/sample/`) with the correct inventory. |

---

## Practical notes

- **Always replace** Gitea host, user, and branch; do not leave `<GITEA_HOST>` / `<GITEA_USER>` placeholders.
- **Private roles:** HTTPS often needs a token; in Dev Spaces the lab Gitea is usually reachable with your user.
- **Dependency order:** if `wildfly_install` depends on another role, use `dependencies` in `meta/main.yml` or the order in the playbook `roles:`.
- **CI/CD:** `ansible-galaxy install -r requirements.yml` before `ansible-playbook`.
- **Galaxy:** without `.galaxy-ignore`, the role installed in exercise1 includes this README, `devfile.yaml`, and `.vscode/`.
- **Inventory from exercise2:** use `{{ inventory_dir }}/ssh_tests_connections/id_fedora_new` in the exercise 1 `inventory`; a cwd-relative path will not resolve the key.

---

## Expected result

- **`lab-devspaces-ansible-exercise2`** contains the role (tasks, defaults, vars, meta, tests), passes yamllint/ansible-lint, the role test, and Molecule (`default` creates and destroys the VM; `verify` checks only this role).
- **`lab-devspaces-ansible-exercise1`** no longer versions `roles/wildfly_os_deps`; it declares that role in `requirements.yml` and the playbook runs the **full installation** (external role + local exercise 1 roles).
