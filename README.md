# lab-devspaces-ansible-exercise2

Curso práctico de **roles Ansible reutilizables**. **Este ejercicio está pensado para realizarse dentro de OpenShift Dev Spaces**.

Versiones de la imagen (las mismas que el ejercicio 1; compruébalas en el workspace):

| Herramienta | Versión en Dev Spaces |
| ----------- | --------------------- |
| Python | **3.12** |
| ansible-core | **2.21.x** |
| yamllint | **1.38.x** |
| ansible-lint | **26.x** (p. ej. 26.6.0) |
| Molecule | **26.x** (p. ej. 26.6.0); driver **`default`** (no existe `delegated`) |

**Qué debes hacer:** **mover** el contenido del rol `wildfly_os_deps` que creaste en **`lab-devspaces-ansible-exercise1`** a **este** repositorio (`lab-devspaces-ansible-exercise2`) y **llamarlo desde el ejercicio 1** con `ansible-galaxy` y `requirements.yml`. No crees un repositorio vacío en la forja: usa el clone de exercise2, que ya tiene Git y `origin` en Gitea.

---

## ¿Qué es un rol en Ansible?

Un **rol** es una unidad reutilizable que agrupa tareas, datos, plantillas y metadatos bajo un **nombre** (`wildfly_os_deps`, `nginx`, etc.). El playbook solo declara qué roles aplicar y en qué orden; Ansible carga automáticamente los ficheros convencionales de cada rol (`tasks/main.yml`, `defaults/main.yml`, `vars/main.yml`, etc.). Así evitas playbooks enormes, puedes **versionar y compartir** un rol en Git (o Galaxy) y **reutilizarlo** en varios proyectos sin copiar y pegar YAML.

No es obligatorio usar todas las carpetas: muchos roles solo tienen `tasks/`, `defaults/` y `meta/`. Las que no existen se ignoran.

### Estructura típica de un rol (vista `tree`)

```text
nombre_del_rol/
├── defaults/
│   └── main.yml           # variables por defecto (baja precedencia)
├── files/
│   └── ejemplo.conf       # ficheros estáticos para copy/fetch
├── handlers/
│   └── main.yml           # acciones disparadas con notify (reinicios, reloads)
├── meta/
│   └── main.yml           # dependencias entre roles e información Galaxy
├── tasks/
│   └── main.yml           # tareas principales del rol
├── templates/
│   └── app.conf.j2        # plantillas Jinja2 para el módulo template
├── tests/
│   └── test.yml           # playbook de prueba del rol
├── vars/
│   └── main.yml           # constantes del rol (alta precedencia)
└── README.md              # documentación humana (opcional pero recomendable)
```

### Carpetas y ficheros: para qué sirven y un ejemplo mínimo

#### `tasks/main.yml`

Lista de **tareas** que Ansible ejecuta al aplicar el rol. Es el núcleo del rol.

```yaml
---
- name: Asegurar que un paquete está instalado
  ansible.builtin.dnf:
    name: httpd
    state: present
```

#### `handlers/main.yml`

**Handlers**: tareas que solo se ejecutan cuando una tarea anterior las **notifica** (`notify`), normalmente al final del play. Evitan reiniciar un servicio en cada corrida si no hubo cambios.

```yaml
---
- name: Recargar httpd
  ansible.builtin.service:
    name: httpd
    state: reloaded
```

#### `defaults/main.yml`

Variables con **menor precedencia**: valores por defecto que el playbook, el inventario o `vars` del play pueden sobrescribir sin tocar el código del rol.

```yaml
---
httpd_port: 80
httpd_package: httpd
```

#### `vars/main.yml`

Variables con **precedencia mayor** que `defaults`; son **constantes del rol** (valores que no quieres que el usuario cambie sin editar el rol).

```yaml
---
httpd_config_path: /etc/httpd/conf/httpd.conf
```

#### `files/`

Ficheros **estáticos** referenciados en tareas con rutas cortas: `ansible.builtin.copy: src: motd` busca `files/motd` dentro del rol.

Ejemplo: un fichero `files/motd` con una sola línea de texto que se copia a `/etc/motd` en el nodo.

#### `templates/`

Plantillas **Jinja2** (extensión `.j2`). El módulo `template` las expande en el destino sustituyendo `{{ variables }}`.

Ejemplo en `templates/virtualhost.conf.j2`:

```jinja2
Listen {{ httpd_port }}
```

#### `meta/main.yml`

**Dependencias** entre roles (Ansible puede instalar o ejecutar otros roles antes) y metadatos para documentación o **Ansible Galaxy**.

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

Playbooks que **ejercitan el rol** de forma aislada (sintaxis, `--check` o una corrida real). Ansible no los ejecuta solo: los lanzas tú con `ansible-playbook` (véase la parte C).

#### `README.md`

Documentación para humanos: qué hace el rol, variables disponibles y ejemplo de uso en un playbook. No lo interpreta Ansible.

En **este** repositorio el `README.md` es la **guía del laboratorio**. No lo sustituyas ni lo añadas al commit como si fuera el README del rol.

---

En la práctica de este laboratorio usarás **`tasks/`**, **`defaults/`**, **`vars/`**, **`meta/`** y **`tests/`**. **`handlers/`**, **`files/`** y **`templates/`** entran en juego cuando el rol crece o necesita plantillas y reinicios coordinados.

---

## En qué consiste este laboratorio

**Objetivo:** tomar el rol **`wildfly_os_deps`** (paquetes Java, `tar`, `gzip`) de la tabla **4.1 del README de `lab-devspaces-ansible-exercise1`**, **mover su contenido** a la raíz de **`lab-devspaces-ansible-exercise2`**, y **volver a invocarlo desde el ejercicio 1** (playbook + `requirements.yml` + `ansible-galaxy`).

Los mismos pasos sirven para `wildfly_account`, `wildfly_install`, `wildfly_bind`, `wildfly_systemd` o `wildfly_sample_app`, ajustando tareas, variables y dependencias.

### Objetivos concretos

1. El código del rol vive en **`lab-devspaces-ansible-exercise2`** (raíz del repo = raíz del rol).
2. En **`lab-devspaces-ansible-exercise1`** **no** queda una copia versionada de `roles/wildfly_os_deps/`: se **elimina** la carpeta local del rol y se declara el origen en `requirements.yml`.
3. El playbook del ejercicio 1 sigue usando el nombre `wildfly_os_deps`; el contenido llega desde Git (este repo) vía Galaxy.

---

## Prerrequisitos

- Haber seguido **`lab-devspaces-ansible-exercise1`** hasta la **sección 4** (roles sugeridos) y tener al menos `roles/wildfly_os_deps/` (u el bloque equivalente en el playbook) en ese proyecto.
- Trabajar **en Dev Spaces**, con las versiones de la tabla del inicio (`git`, `ansible`, `ansible-galaxy`, `yamllint`, `ansible-lint`, Molecule).
- Este clone ya es un repositorio Git con `origin` en Gitea: **no** crees un repo vacío ni ejecutes `git init` / `git remote add`.

---

## Parte A — Mover el rol a `lab-devspaces-ansible-exercise2`

### A.1 Este repositorio es el del rol

**No** hay que crear un repositorio vacío en la forja. Trabajas en **`lab-devspaces-ansible-exercise2`**, que ya está clonado y apunta a Gitea.

Convención: la **raíz de este repo es la raíz del rol** (no una subcarpeta `roles/wildfly_os_deps`). Así `ansible-galaxy install` instala el contenido con el `name` que declares en `requirements.yml`.

Árbol mínimo en **`lab-devspaces-ansible-exercise2/`**:

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
        └── …   # parte D
```

(`README.md`, `devfile.yaml` y `.vscode/` ya vienen con el laboratorio; déjalos. `tests/` y `molecule/` se crean en las partes C y D, no en el primer commit de A.7. Sobre `.galaxy-ignore`, véase A.7: **no** evita que un `ansible-galaxy install` por Git copie esos ficheros.)

### A.2 Mover el contenido desde el ejercicio 1

Desde el workspace, copia (o recorta) lo que tengas en exercise1 hacia la raíz de exercise2:

```bash
# Ajusta las rutas si tus proyectos no están al mismo nivel
mkdir -p tasks
cp -a ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps/tasks/. ./tasks/
# defaults/ puede no existir en el ejercicio 1 (p. ej. si las vars están en group_vars):
cp -a ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps/defaults/. ./defaults/ 2>/dev/null || true
# si ya tenías meta/ o vars/ en el ejercicio 1, cópialos igual
```

Si `defaults/` no estaba en exercise1, o el rol solo existía como tareas del playbook monolítico, **crea** los ficheros de A.3–A.6 a mano (dependencias OS: Java, `tar`, `gzip`). No asumas que el `cp` de `defaults` va a funcionar.

**Después**, en el ejercicio 1, **borra** la copia local para no duplicar la fuente de verdad:

```bash
rm -rf ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps
```

El playbook del ejercicio 1 seguirá llamando a `wildfly_os_deps` cuando lo instales con Galaxy (parte B).

### A.3 Contenido de `tasks/main.yml`

Usa las tareas que moviste. Si partes del bloque de dependencias del ejercicio 1:

```yaml
---
- name: Instalar dependencias (Java 17+ es requerido para WF 39)
  ansible.builtin.dnf:
    name: "{{ wf_os_packages }}"
    state: present
```

### A.4 Contenido de `defaults/main.yml`

Valores que el playbook **puede** sobrescribir (`group_vars`, `vars:` del play):

```yaml
---
wf_java_package: java-25-openjdk-devel
```

### A.5 Contenido de `vars/main.yml` (constantes del rol)

Aquí van constantes que **no** deberían cambiarse desde el playbook. En este rol, las utilidades de descompresión son fijas; el JDK sigue siendo sobreescribible vía `defaults`:

```yaml
---
wf_os_unpack_packages:
  - tar
  - gzip
wf_os_packages: "{{ [wf_java_package] + wf_os_unpack_packages }}"
```

Si tu rol no tiene constantes, deja el fichero con la cabecera `---` y un comentario; la carpeta `vars/` documenta la convención.

### A.6 `meta/main.yml` (metadatos del rol)

```yaml
---
galaxy_info:
  author: tu_usuario_de_laboratorio
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

Sustituye `author` por tu usuario de laboratorio (datos de acceso). `role_name` debe coincidir con el `name` de `requirements.yml` (parte B). `namespace` **no puede llevar guiones ni mayúsculas**: Galaxy y ansible-lint rechazan valores como `lab-user-2`. Usa algo como `labuser` o `labuser2`.

Si ansible-lint o Molecule siguen quejándose de `role-name` (el directorio del repo tiene guiones), añade un `.ansible-lint` en la raíz de este repo:

```yaml
---
skip_list:
  - role-name
```

### A.7 Publicar los ficheros del rol

El remoto **ya existe**. Añade solo el código del rol que **ya existe en este punto** (A.3–A.6). **No** hagas `git add tests molecule`: esas carpetas se crean en las partes C y D; si no existen, Git falla con `pathspec ... did not match any files`.

Crea `.galaxy-ignore` (útil si más adelante **empaquetas** el rol para Galaxy). **No** impide que `ansible-galaxy install` con `scm: git` clone el repo entero: en exercise1 verás este `README.md`, `README_EN.md`, `devfile.yaml` y `.vscode/` dentro de `roles/wildfly_os_deps/`. Eso es esperado con install-from-git. Tras instalar, puedes borrar a mano esos ficheros del rol instalado; **no** versionas esa carpeta (es un artefacto de Galaxy).

Añade también `.ansible/` (el symlink de la parte C) para no subirlo si algún día empaquetas el rol:

```text
README.md
README_EN.md
devfile.yaml
.vscode
.ansible
```

```bash
git add defaults vars meta tasks .galaxy-ignore .ansible-lint
git commit -m "Add wildfly_os_deps role"
git push origin HEAD
```

Tras las partes C y D, haz otro commit con `tests/`, `ansible.cfg` y `molecule/`.

Anota la **rama** que empujas (`master` en el clone del laboratorio, salvo que uses otra): la necesitarás en `version:` de `requirements.yml`.

---

## Parte B — Llamar el rol desde `lab-devspaces-ansible-exercise1`

Trabaja en el directorio del **playbook** del ejercicio 1 (`lab-devspaces-ansible-exercise1`).

### B.1 Quitar el rol local del ejercicio 1

Si aún existe `roles/wildfly_os_deps/` en exercise1, elimínalo (véase A.2). El resto de roles locales (`wildfly_account`, etc.) pueden quedarse en `roles/`.

Si versionas el playbook, no subas el `roles/wildfly_os_deps` generado por Galaxy: es un artefacto de `ansible-galaxy install`.

### B.2 `requirements.yml` en la raíz de exercise1

**Sustituye** `<GITEA_HOST>` y `<GITEA_USER>` por los de **tus datos de acceso de laboratorio** (URL de Gitea y usuario, por ejemplo `lab-user-1`). Sustituye `master` si empujaste otra rama.

Si el ejercicio 1 **ya tiene** un `requirements.yml` (p. ej. con `collections:` para Molecule), **no lo sustituyas**: añade el bloque `roles:` al fichero existente. Si lo reescribes solo con `roles:`, pierdes las collections.

```yaml
---
roles:
  - name: wildfly_os_deps
    src: https://<GITEA_HOST>/<GITEA_USER>/lab-devspaces-ansible-exercise2.git
    scm: git
    version: master
```

Ejemplo de `src` (los valores concretos los pone cada alumno):

```text
https://<GITEA_HOST>/<GITEA_USER>/lab-devspaces-ansible-exercise2.git
```

No copies una URL de ejemplo de un compañero: usa **tu** Gitea y **tu** usuario.

### B.3 Instalar el rol en el proyecto del playbook

Desde `lab-devspaces-ansible-exercise1`:

```bash
ansible-galaxy role install -r requirements.yml --roles-path ./roles
```

`--roles-path` hace que Galaxy **ignore** las `collections:` del mismo `requirements.yml` (verás un warning). Las collections se instalan aparte, si las necesitas:

```bash
ansible-galaxy collection install -r requirements.yml
```

Comprueba que existe `roles/wildfly_os_deps/tasks/main.yml`. Es normal que también aparezcan README/`devfile`/`.vscode` (install por Git; véase A.7).

### B.4 Configurar `ansible.cfg` (recomendado)

En la raíz de exercise1:

```ini
[defaults]
roles_path = ./roles
```

### B.5 Playbook completo (rol externo + roles locales)

En `deploy-wildfly.yaml` del ejercicio 1 **no comentes** el resto de roles. Solo `wildfly_os_deps` pasa a instalarse desde Git (este ejercicio); **`wildfly_account`**, **`wildfly_install`**, **`wildfly_bind`**, **`wildfly_systemd`** y **`wildfly_sample_app`** siguen siendo los que ya tenías en `roles/` del ejercicio 1. Así la corrida sigue siendo la **instalación completa de WildFly**, no un play parcial.

El `name` de Galaxy debe coincidir con `name` en `requirements.yml`:

```yaml
---
- name: Instalación de WildFly con roles
  hosts: servers
  become: true
  roles:
    - role: wildfly_os_deps      # externo: Galaxy / este repo (exercise2)
    - role: wildfly_account      # local: ejercicio 1
    - role: wildfly_install
    - role: wildfly_bind
    - role: wildfly_systemd
    - role: wildfly_sample_app
```

Si extraes **otro** rol a Git más adelante, el patrón es el mismo: ese rol sale de `roles/` local, entra en `requirements.yml`, y los demás se quedan en el playbook para que el orden y el resultado final no cambien.

### B.6 Verificación desde el ejercicio 1

Inventario: usa la IP de **tus datos de laboratorio** (véase el README del ejercicio 1).

```bash
ansible-playbook -i inventory deploy-wildfly.yaml --syntax-check
ansible-playbook -i inventory deploy-wildfly.yaml
```

`--syntax-check` solo valida YAML/play. La corrida **real** debe completar WildFly y `/sample/` como en el ejercicio 1; `wildfly_os_deps` solo cambia **de dónde** sale ese rol.

No uses `--check` sobre el playbook **completo** de WildFly: la tarea que empaqueta el WAR en localhost no escribe `/tmp/sample.war` en modo check y la copia siguiente falla (`Could not find or access '/tmp/sample.war'`). El `--check` del **test del rol** (parte C) sí es válido, porque solo instala RPM.

---

## Parte C — Tests del rol, yamllint y ansible-lint

Hazlo **en `lab-devspaces-ansible-exercise2`**, sobre el código del rol (no sobre la guía).

### C.1 Playbook de test (`tests/test.yml`)

El play debe llamar al rol **por su nombre** (`wildfly_os_deps`), no por una ruta (`{{ playbook_dir }}/..`). ansible-lint 26.x marca las rutas como `role-name[path]` y el `prepare` de Molecule fallaría.

Como la raíz de este repo **es** el rol (no hay subcarpeta `roles/wildfly_os_deps`), enlaza el directorio actual bajo ese nombre y declara `roles_path` en `ansible.cfg`:

```bash
mkdir -p .ansible/roles
ln -sfn "$(pwd)" .ansible/roles/wildfly_os_deps
```

`ansible.cfg` en la raíz de **este** repositorio:

```ini
[defaults]
roles_path = .ansible/roles
host_key_checking = False
```

`.ansible/` es local: no lo versiones (añádelo a `.gitignore` de este repo). `.galaxy-ignore` **no** evita que Git lo clone al instalar el rol; el symlink tampoco debe entrar en `git add`.

```yaml
---
- name: Test del rol wildfly_os_deps
  hosts: servers
  become: true
  roles:
    - role: wildfly_os_deps
```

### C.2 Cómo ejecutarlo

Usa el inventario del ejercicio 1 (IP de tus datos de laboratorio). La clave SSH de ese fichero suele ser `ssh_tests_connections/id_fedora_new`: esa ruta es **relativa al directorio de trabajo**. Si lanzas el playbook **desde exercise2**, SSH no encuentra la clave (`No such file or directory`).

Corrige el `inventory` del ejercicio 1 para anclar la clave a `inventory_dir` (sirve desde exercise1 y desde exercise2). En un inventario **INI** las llaves `{{ }}` **hay que citarlas**; si no, Ansible falla con `Expected key=value host variable assignment, got: inventory_dir` y `--syntax-check` puede salir **0** igual (inventario vacío, solo `localhost`):

```ini
[servers]
fedora-user1 ansible_host=<IP_de_tus_datos_de_laboratorio> ansible_user=user1 ansible_ssh_private_key_file="{{ inventory_dir }}/ssh_tests_connections/id_fedora_new"
```

Sustituye host, IP y usuario por **tus** datos. Comprueba que el inventario resuelve (`ansible -i … servers -m ping`) **antes** de fiarte del `--syntax-check`. Luego, desde `lab-devspaces-ansible-exercise2`:

```bash
ansible-playbook -i ../lab-devspaces-ansible-exercise1/inventory tests/test.yml --syntax-check
ansible-playbook -i ../lab-devspaces-ansible-exercise1/inventory tests/test.yml --check
# corrida real (instala Java/tar/gzip en la VM):
ansible-playbook -i ../lab-devspaces-ansible-exercise1/inventory tests/test.yml
```

Si tus carpetas de proyecto no son hermanas, ajusta la ruta del `-i inventory`.

### C.3 yamllint

Valida el YAML **del rol** (no hace falta lintar este README ni el `devfile.yaml` del laboratorio):

```bash
yamllint defaults vars meta tasks tests
```

Si prefieres `yamllint .`, añade un `.yamllint` que ignore `devfile.yaml`, este README y `.cache/`. Corrige avisos hasta código de salida `0`.

### C.4 ansible-lint

Analiza el rol completo (tareas, defaults, vars, meta y tests):

```bash
ansible-lint defaults vars meta tasks tests
```

Revisa **toda** la salida (no solo un aviso suelto) y corrige hasta código de salida `0`. Vuelve a lanzar C.3 y C.4 si cambias YAML.

---

## Parte D — Molecule: probar el playbook de test del rol

Molecule ejecuta `tests/test.yml` (parte C) contra una máquina de prueba: **create** → **prepare** (lint) → **converge** (el test del rol) → **verify** (solo lo que hace **este** rol) → **destroy**.

La imagen de Dev Spaces incluye **Molecule 26.x** (p. ej. 26.6.0) con ansible-core **2.21** y Python **3.12**. El driver se llama `default` (el nombre antiguo `delegated` ya no existe).

| Escenario | Máquina | create / destroy | Dónde |
| --------- | ------- | ---------------- | ----- |
| `default` | VM Fedora **nueva** en OpenShift (KubeVirt), `fedora-mol-<usuario>` | Crea la VM (cloud-init y clave SSH) y **la destruye** al terminar | **Únicamente desde Dev Spaces**, tras sustituir URL, usuario, contraseña y el namespace `virtualization-test-<usuario>` (D.1). |
| `with_existing_machine` | Fedora **prearrancada** del laboratorio (inventario del ejercicio 1) | No crea ni borra esa VM | Dev Spaces, contra tu Fedora. **Este es el escenario que debes completar** en el aula. |

**Qué verifica este rol:** paquetes Java (`java-25-openjdk-devel`), `tar` y `gzip`. **No** compruebes el servicio `wildfly`, el puerto 8080 ni `/sample/`: eso lo hacen otros roles del ejercicio 1.

Si extraes **otro** rol (`wildfly_install`, `wildfly_systemd`, `wildfly_sample_app`, …), `tests/test.yml` y `verify.yml` tendrían que incluir **antes** los pasos previos que ese rol necesita (por ejemplo usuario, tarball y `standalone.xml` antes de systemd). Este ejemplo solo cubre `wildfly_os_deps`, que no depende de roles anteriores.

Crea los directorios:

```bash
mkdir -p molecule/default molecule/with_existing_machine
```

### D.1 Escenario `default` — VM de prueba en OpenShift (solo Dev Spaces)

Misma VM que la **sección 5.3.1 del ejercicio 1**: KubeVirt, cloud-init con la clave del laboratorio, espera de la IP de la VMI y SSH como usuario `fedora`. Aquí el `converge` ejecuta `tests/test.yml` (solo `wildfly_os_deps`), no `deploy-wildfly.yaml`.

Sustituye los mismos marcadores que en el ejercicio 1:

| Marcador | Dónde | Valor que debes poner |
| -------- | ----- | --------------------- |
| `<usuario>` | Nombre de la plataforma en `molecule.yml` (los tres sitios) y la ruta `/tmp/molecule-default-<usuario>.ip` en `molecule.yml` y `create.yml` | Tu usuario de laboratorio, **el mismo** en todos esos sitios. Si eres `user2`, la VM se llama `fedora-mol-user2` y el fichero es `/tmp/molecule-default-user2.ip`. |
| `ocp_url` | `molecule_vars.yml` | URL de la API, con el formato `https://api.tu-cluster.com:6443` |
| `ocp_user` | `molecule_vars.yml` | `<usuario_de_laboratorio>` (el mismo usuario que `<usuario>`) |
| `ocp_pass` | `molecule_vars.yml` | `<password_de_laboratorio>` |
| `ocp_namespace` | `molecule_vars.yml` | `virtualization-test-<usuario_de_laboratorio>` |

El primer paso de `molecule test` es `destroy`, así que `molecule_vars.yml` tiene que estar relleno antes de cualquier comando de este escenario. Ese fichero lleva usuario y contraseña de OpenShift: **no** lo subas a Gitea.

Dos diferencias respecto al ejercicio 1, porque este repositorio **es** el rol y la clave SSH vive en el repo hermano:

- `roles_path` apunta a `${MOLECULE_PROJECT_DIRECTORY}/.ansible/roles` (el enlace de la parte C.1). Sin ese `ln -sfn`, `converge` no encuentra `wildfly_os_deps`.
- La clave es `../lab-devspaces-ansible-exercise1/ssh_tests_connections/id_fedora_new` (pública en cloud-init, privada en `ansible_ssh_private_key_file`).

La plataforma se llama `fedora-mol-<usuario>` y vive en `virtualization-test-<usuario>`, el mismo proyecto que la Fedora del inventario (`fedora-<usuario>`). `destroy` borra solo `fedora-mol-<usuario>`.

##### `molecule/default/molecule.yml`

```yaml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml
driver:
  name: default
platforms:
  - name: fedora-mol-<usuario>
provisioner:
  name: ansible
  config_options:
    defaults:
      roles_path: ${MOLECULE_PROJECT_DIRECTORY}/.ansible/roles
      collections_path: ~/.ansible/collections
      host_key_checking: false
      interpreter_python: auto_silent
  inventory:
    hosts:
      all:
        children:
          servers:
            hosts:
              fedora-mol-<usuario>: {}
    host_vars:
      fedora-mol-<usuario>:
        ansible_user: fedora
        ansible_host: "{{ lookup('ansible.builtin.file', '/tmp/molecule-default-<usuario>.ip', errors='ignore') | default('127.0.0.1', true) | trim }}"
        ansible_ssh_private_key_file: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') }}/../lab-devspaces-ansible-exercise1/ssh_tests_connections/id_fedora_new"
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

##### `molecule/default/molecule_vars.yml`

```yaml
---
ocp_url: "https://api.tu-cluster.com:6443"
ocp_user: "<usuario_de_laboratorio>"
ocp_pass: "<password_de_laboratorio>"
ocp_namespace: "virtualization-test-<usuario_de_laboratorio>"
```

El namespace sigue el formato `virtualization-test-<usuario_de_laboratorio>`. **No** subas este fichero a un remoto público.

##### `molecule/default/requirements.yml`

Molecule lo instala en el paso `dependency`, antes de `create`.

```yaml
---
collections:
  - name: kubevirt.core
  - name: kubernetes.core
  - name: community.general
```

##### `molecule/default/create.yml`

Entra en OpenShift, crea la VM con cloud-init y espera SSH. `lab_pubkey` apunta a la clave pública del ejercicio 1 (desde `molecule/default`, tres niveles arriba está el workspace).

```yaml
---
- name: Create VM in OpenShift
  hosts: localhost
  gather_facts: false
  vars_files:
    - molecule_vars.yml
  vars:
    lab_pubkey: >-
      {{ playbook_dir }}/../../../lab-devspaces-ansible-exercise1/ssh_tests_connections/id_fedora_new.pub
  tasks:
    - name: Log in to OpenShift
      ansible.builtin.command:
        cmd: >-
          oc login --insecure-skip-tls-verify=false
          --username {{ ocp_user }}
          --password {{ ocp_pass }}
          {{ ocp_url }}
      changed_when: false
      no_log: true

    - name: Get OpenShift API token
      ansible.builtin.command:
        cmd: oc whoami --show-token
      register: token
      changed_when: false

    - name: Create Fedora VM using KubeVirt
      kubevirt.core.kubevirt_vm:
        host: "{{ ocp_url }}"
        api_key: "{{ token.stdout }}"
        validate_certs: true
        state: present
        run_strategy: Always
        wait: true
        wait_timeout: 600
        namespace: "{{ ocp_namespace }}"
        name: "{{ item.name }}"
        spec:
          domain:
            resources:
              requests:
                memory: 4Gi
            devices:
              interfaces:
                - name: default
                  masquerade: {}
              disks:
                - name: containerdisk
                  disk:
                    bus: virtio
                - name: cloudinit
                  disk:
                    bus: virtio
          networks:
            - name: default
              pod: {}
          volumes:
            - name: containerdisk
              containerDisk:
                image: quay.io/containerdisks/fedora:latest
            - name: cloudinit
              cloudInitNoCloud:
                userData: |
                  #cloud-config
                  ssh_authorized_keys:
                    - {{ lookup('ansible.builtin.file', lab_pubkey) | trim }}
      loop: "{{ molecule_yml.platforms }}"

    - name: Wait until the VMI has an IP
      ansible.builtin.command:
        cmd: >-
          oc get vmi {{ molecule_yml.platforms[0].name }}
          -n {{ ocp_namespace }}
          -o jsonpath={.status.interfaces[0].ipAddress}
      register: vmi_ip
      changed_when: false
      retries: 36
      delay: 10
      until: vmi_ip.stdout is match('([0-9]{1,3}\.){3}[0-9]{1,3}')

    - name: Save the VMI IP for converge
      ansible.builtin.copy:
        dest: /tmp/molecule-default-<usuario>.ip
        content: "{{ vmi_ip.stdout | trim }}\n"
        mode: "0644"

    - name: Wait for SSH on the VMI
      ansible.builtin.wait_for:
        host: "{{ vmi_ip.stdout | trim }}"
        port: 22
        timeout: 300
```

La ruta de `/tmp/molecule-default-<usuario>.ip` tiene que coincidir con el `lookup` de `ansible_host`. La memoria es **4Gi**, la misma VM que en el ejercicio 1.

##### `molecule/default/destroy.yml`

```yaml
---
- name: Destroy VM in OpenShift
  hosts: localhost
  gather_facts: false
  vars_files:
    - molecule_vars.yml
  tasks:
    - name: Log in to OpenShift
      ansible.builtin.command:
        cmd: >-
          oc login --insecure-skip-tls-verify=false
          --username {{ ocp_user }}
          --password {{ ocp_pass }}
          {{ ocp_url }}
      changed_when: false
      no_log: true

    - name: Get OpenShift API token
      ansible.builtin.command:
        cmd: oc whoami --show-token
      register: token
      changed_when: false

    - name: Remove Fedora VM from OpenShift
      kubevirt.core.kubevirt_vm:
        host: "{{ ocp_url }}"
        api_key: "{{ token.stdout }}"
        validate_certs: true
        state: absent
        wait: true
        wait_timeout: 300
        namespace: "{{ ocp_namespace }}"
        name: "{{ item.name }}"
      loop: "{{ molecule_yml.platforms }}"
      ignore_errors: true

    - name: Clean up associated Service
      kubernetes.core.k8s:
        host: "{{ ocp_url }}"
        api_key: "{{ token.stdout }}"
        validate_certs: true
        state: absent
        namespace: "{{ ocp_namespace }}"
        kind: Service
        name: "svc-{{ molecule_yml.platforms[0].name }}"
      ignore_errors: true
```

`ignore_errors` deja continuar si la VM no llegó a crearse. `destroy` vuelve a hacer `oc login` porque Molecule lo ejecuta en un playbook aparte.

##### `molecule/default/prepare.yml`

Lint **del rol** (no de `deploy-wildfly.yaml`):

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

Ejecuta el playbook de test del rol (parte C), no el playbook completo de WildFly:

```yaml
---
- name: Converge
  ansible.builtin.import_playbook: ../../tests/test.yml
```

##### `molecule/default/verify.yml`

Solo aserciones de **este** rol:

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

`prepare.yml`, `converge.yml` y `verify.yml` son de **este** rol (lint del árbol del rol, `tests/test.yml`, paquetes Java). `create.yml` y `destroy.yml` son los de arriba. El escenario del aula sigue siendo `with_existing_machine` (D.2). El `default` se lanza **solo desde Dev Spaces** cuando ya sustituiste `<usuario>`, `ocp_url`, `ocp_user`, `ocp_pass` y `ocp_namespace`.

### D.2 Escenario `with_existing_machine` — VM del laboratorio

Copia `prepare.yml`, `converge.yml` y `verify.yml` del escenario `default`. `create.yml` / `destroy.yml` no deben crear ni apagar la Fedora del alumno (mismo patrón que el ejercicio 1: solo un `debug`).

En `molecule.yml` usa `driver.name: default` con `managed: false`. El `ansible_host` es la IP de **tus datos de laboratorio** (la misma del `inventory` de exercise1) y el puerto **22**. En este aula Dev Spaces llega a la Fedora **directamente**; `127.0.0.1:2222` solo aplica si el formador te ha pedido un **túnel SSH**. Si los copias tal cual, Molecule no conecta.

La clave SSH debe ser alcanzable **desde exercise2**, por ejemplo:

```yaml
ansible_ssh_private_key_file: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') }}/../lab-devspaces-ansible-exercise1/ssh_tests_connections/id_fedora_new"
```

### D.3 Lanzar Molecule

Desde la raíz de **`lab-devspaces-ansible-exercise2`**:

```bash
# Fedora del laboratorio; no se borra (escenario del aula)
molecule test -s with_existing_machine

# VM nueva en OpenShift; al terminar se destruye (solo Dev Spaces, con los marcadores sustituidos)
molecule test -s default
```

Paso a paso (depuración), escenario `with_existing_machine`:

```bash
molecule create -s with_existing_machine
molecule prepare -s with_existing_machine
molecule converge -s with_existing_machine
molecule verify -s with_existing_machine
molecule destroy -s with_existing_machine
```

Tras `destroy` del escenario `default`, la VM `fedora-mol-<usuario>` **ya no** debe existir en `virtualization-test-<usuario_de_laboratorio>`. La Fedora del inventario sigue en ese proyecto. Tras `destroy` de `with_existing_machine`, la Fedora del alumno **sigue arrancada**.

---

## Resumen de pasos (checklist)

| Paso | Dónde | Acción |
|------|--------|--------|
| 1 | exercise2 | Estructura `tasks/`, `defaults/`, `vars/`, `meta/`, `tests/` en la raíz de **este** repo. |
| 2 | exercise1 → exercise2 | **Mover** el contenido de `roles/wildfly_os_deps` (o el bloque equivalente) a exercise2. |
| 3 | exercise1 | **Eliminar** `roles/wildfly_os_deps` local. |
| 4 | exercise2 | `.galaxy-ignore` + `git add` solo de lo que exista (`defaults vars meta tasks .galaxy-ignore .ansible-lint`); `commit` y `git push origin HEAD`. `tests/` y `molecule/` en un commit posterior. No subas `.ansible/`. |
| 5 | exercise1 | Añade `roles:` a `requirements.yml` (no borres `collections:` si ya estaban) con `src` de **tu** Gitea. |
| 6 | exercise1 | `ansible-galaxy role install -r requirements.yml --roles-path ./roles` (y `collection install` aparte si aplica). |
| 7 | exercise1 | Playbook **completo**: `wildfly_os_deps` externo + el resto de roles **locales** del ejercicio 1. |
| 8 | exercise2 | `ansible.cfg` + enlace `.ansible/roles/wildfly_os_deps`, `tests/test.yml` (rol por **nombre**), inventario INI con `inventory_dir` **entre comillas**, `yamllint` + `ansible-lint`. |
| 9 | exercise2 | Molecule `with_existing_machine`: `converge` = test del rol; `verify` = Java/tar/gzip. `default` crea `fedora-mol-<usuario>` en `virtualization-test-<usuario_de_laboratorio>` tras sustituir los marcadores de D.1. |
| 10 | exercise1 | `ansible-playbook` **real** (WildFly + `/sample/`); no uses `--check` del playbook completo. |

---

## Notas prácticas

- **Sustituye siempre** host Gitea, usuario y rama; no dejes los marcadores `<GITEA_HOST>` / `<GITEA_USER>`.
- **Roles privados:** con HTTPS suele hacer falta token; en el Gitea del laboratorio el `ansible-galaxy install` por Git suele funcionar sin token extra.
- **Orden de dependencias:** si `wildfly_install` depende de otro rol, usa `dependencies` en `meta/main.yml` o el orden en `roles:` del playbook.
- **CI/CD:** `ansible-galaxy role install -r requirements.yml --roles-path ./roles` (y `collection install` si el fichero también lista collections) antes de `ansible-playbook`.
- **Galaxy / Git:** `.galaxy-ignore` no se aplica al clone; el rol instalado incluirá README, `devfile.yaml` y `.vscode/`. No versiones `roles/wildfly_os_deps/` en exercise1.
- **Inventario desde exercise2:** usa `ansible_ssh_private_key_file="{{ inventory_dir }}/ssh_tests_connections/id_fedora_new"` (con **comillas**) en el `inventory` INI del ejercicio 1; una ruta relativa al cwd no resuelve la clave, y sin comillas el inventario ni siquiera parsea.

---

## Resultado esperado

- **`lab-devspaces-ansible-exercise2`** contiene el rol (tasks, defaults, vars, meta, tests), pasa yamllint/ansible-lint, el test del rol y Molecule `with_existing_machine` (`verify` solo comprueba este rol). El escenario `default` crea `fedora-mol-<usuario>` en `virtualization-test-<usuario_de_laboratorio>` cuando sustituyes `<usuario>`, `ocp_url`, `ocp_user`, `ocp_pass` y `ocp_namespace`; al terminar, Molecule borra esa VM.
- **`lab-devspaces-ansible-exercise1`** ya **no** versiona `roles/wildfly_os_deps`; declara ese rol en `requirements.yml` y el playbook ejecuta la **instalación completa** (rol externo + roles locales del ejercicio 1).
