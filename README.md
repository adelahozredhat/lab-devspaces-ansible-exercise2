# lab-devspaces-ansible-exercise2

Curso práctico de **roles Ansible reutilizables**. **Este ejercicio está pensado para realizarse dentro de OpenShift Dev Spaces**.

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
- Trabajar **en Dev Spaces**, con `git`, `ansible`, `ansible-galaxy`, `yamllint` y `ansible-lint` (imagen del Devfile).
- Este clone ya es un repositorio Git con `origin` en Gitea: **no** crees un repo vacío ni ejecutes `git init` / `git remote add`.

---

## Parte A — Mover el rol a `lab-devspaces-ansible-exercise2`

### A.1 Este repositorio es el del rol

**No** hay que crear un repositorio vacío en la forja. Trabajas en **`lab-devspaces-ansible-exercise2`**, que ya está clonado y apunta a Gitea.

Convención: la **raíz de este repo es la raíz del rol** (no una subcarpeta `roles/wildfly_os_deps`). Así `ansible-galaxy install` instala el contenido con el `name` que declares en `requirements.yml`.

Árbol mínimo en **`lab-devspaces-ansible-exercise2/`**:

```text
lab-devspaces-ansible-exercise2/
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

(`README.md`, `devfile.yaml` y `.vscode/` ya vienen con el laboratorio; déjalos.)

### A.2 Mover el contenido desde el ejercicio 1

Desde el workspace, copia (o recorta) lo que tengas en exercise1 hacia la raíz de exercise2:

```bash
# Ajusta las rutas si tus proyectos no están al mismo nivel
cp -a ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps/tasks ./tasks
cp -a ../lab-devspaces-ansible-exercise1/roles/wildfly_os_deps/defaults ./defaults
# si ya tenías meta/ o vars/ en el ejercicio 1, cópialos igual
```

Si en el ejercicio 1 el rol solo existía como tareas dentro del playbook monolítico, crea los ficheros de A.3–A.6 a partir de ese bloque (dependencias OS: Java, `tar`, `gzip`).

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

Sustituye `author` por tu usuario de laboratorio (datos de acceso).

### A.7 Publicar los ficheros del rol

El remoto **ya existe**. Añade solo el código del rol (no este README de guía):

```bash
git add defaults vars meta tasks tests molecule
git commit -m "Add wildfly_os_deps role"
git push origin HEAD
```

Anota la **rama** que empujas (`master` en el clone del laboratorio, salvo que uses otra): la necesitarás en `version:` de `requirements.yml`.

---

## Parte B — Llamar el rol desde `lab-devspaces-ansible-exercise1`

Trabaja en el directorio del **playbook** del ejercicio 1 (`lab-devspaces-ansible-exercise1`).

### B.1 Quitar el rol local del ejercicio 1

Si aún existe `roles/wildfly_os_deps/` en exercise1, elimínalo (véase A.2). El resto de roles locales (`wildfly_account`, etc.) pueden quedarse en `roles/`.

Si versionas el playbook, no subas el `roles/wildfly_os_deps` generado por Galaxy: es un artefacto de `ansible-galaxy install`.

### B.2 Crear `requirements.yml` en la raíz de exercise1

**Sustituye** `<GITEA_HOST>` y `<GITEA_USER>` por los de **tus datos de acceso de laboratorio** (URL de Gitea y usuario, por ejemplo `lab-user-1`). Sustituye `master` si empujaste otra rama.

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
ansible-galaxy install -r requirements.yml --roles-path ./roles
```

Comprueba que existe `roles/wildfly_os_deps/tasks/main.yml`.

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
ansible-playbook -i inventory deploy-wildfly.yaml --check
ansible-playbook -i inventory deploy-wildfly.yaml
```

La ejecución real debe completar WildFly y `/sample/` como en el ejercicio 1; `wildfly_os_deps` solo cambia **de dónde** sale ese rol.

---

## Parte C — Tests del rol, yamllint y ansible-lint

Hazlo **en `lab-devspaces-ansible-exercise2`**, sobre el código del rol (no sobre la guía).

### C.1 Playbook de test (`tests/test.yml`)

```yaml
---
- name: Test del rol wildfly_os_deps
  hosts: servers
  become: true
  tasks:
    - name: Aplicar el rol bajo prueba
      ansible.builtin.include_role:
        name: "{{ playbook_dir }}/.."
```

Este play importa el rol que está en la **raíz de este repositorio**.

### C.2 Cómo ejecutarlo

Usa el **mismo inventario** del ejercicio 1 (IP de tus datos de laboratorio):

```bash
# desde lab-devspaces-ansible-exercise2
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

La imagen de Dev Spaces incluye **Molecule 25.5.0**. El driver se llama `default` (el nombre antiguo `delegated` ya no existe).

| Escenario | Máquina | create / destroy | Dónde |
| --------- | ------- | ---------------- | ----- |
| `default` | VM Fedora **nueva** en OpenShift (KubeVirt) | Crea la VM al inicio y **la destruye** al terminar | **Únicamente desde Dev Spaces** |
| `with_existin_machine` | Fedora **prearrancada** del laboratorio (inventario del ejercicio 1) | No crea ni borra esa VM | Dev Spaces, contra tu Fedora |

**Qué verifica este rol:** paquetes Java (`java-25-openjdk-devel`), `tar` y `gzip`. **No** compruebes el servicio `wildfly`, el puerto 8080 ni `/sample/`: eso lo hacen otros roles del ejercicio 1.

Si extraes **otro** rol (`wildfly_install`, `wildfly_systemd`, `wildfly_sample_app`, …), `tests/test.yml` y `verify.yml` tendrían que incluir **antes** los pasos previos que ese rol necesita (por ejemplo usuario, tarball y `standalone.xml` antes de systemd). Este ejemplo solo cubre `wildfly_os_deps`, que no depende de roles anteriores.

Crea los directorios:

```bash
mkdir -p molecule/default molecule/with_existin_machine
```

### D.1 Escenario `default` — VM de prueba en OpenShift (solo Dev Spaces)

Misma idea que la **sección 5.3.1 del ejercicio 1**: `create.yml` / `destroy.yml` / `molecule_vars.yml` dan de alta y eliminan una Fedora en KubeVirt. Copia esos tres ficheros desde exercise1 o reprodúcelos igual (driver `name: default`, credenciales `ocp_*` de **tus** datos de laboratorio, `namespace` del aula).

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

**Create / destroy:** mismos `molecule/default/create.yml`, `destroy.yml` y `molecule_vars.yml` que en el ejercicio 1 (login `oc`, VM KubeVirt, borrado al final). El escenario `default` se lanza **solo desde Dev Spaces**.

### D.2 Escenario `with_existin_machine` — VM del laboratorio

Copia `prepare.yml`, `converge.yml` y `verify.yml` del escenario `default`. `create.yml` / `destroy.yml` no deben crear ni apagar la Fedora del alumno (mismo patrón que el ejercicio 1: solo un `debug`).

En `molecule.yml` usa `driver.name: default` con `managed: false` y el `ansible_host` de **tus datos de laboratorio** (la IP del `inventory` de exercise1; `127.0.0.1:2222` es solo un ejemplo de túnel).

### D.3 Lanzar Molecule

Desde la raíz de **`lab-devspaces-ansible-exercise2`**:

```bash
# VM nueva en OpenShift; al terminar se destruye (solo Dev Spaces)
molecule test -s default

# Fedora del laboratorio; no se borra
molecule test -s with_existin_machine
```

Paso a paso (depuración), escenario `default`:

```bash
molecule create -s default
molecule prepare -s default
molecule converge -s default
molecule verify -s default
molecule destroy -s default
```

Tras `destroy` del escenario `default`, esa VM de prueba **ya no** debe existir en OpenShift. Tras `destroy` de `with_existin_machine`, la Fedora del alumno **sigue arrancada**.

---

## Resumen de pasos (checklist)

| Paso | Dónde | Acción |
|------|--------|--------|
| 1 | exercise2 | Estructura `tasks/`, `defaults/`, `vars/`, `meta/`, `tests/` en la raíz de **este** repo. |
| 2 | exercise1 → exercise2 | **Mover** el contenido de `roles/wildfly_os_deps` (o el bloque equivalente) a exercise2. |
| 3 | exercise1 | **Eliminar** `roles/wildfly_os_deps` local. |
| 4 | exercise2 | `git add` del rol, `commit` y `git push origin HEAD` (sin `git init` ni `remote add`). |
| 5 | exercise1 | `requirements.yml` con `src: https://<GITEA_HOST>/<GITEA_USER>/lab-devspaces-ansible-exercise2.git` (**sustituye** host, usuario y rama). |
| 6 | exercise1 | `ansible-galaxy install -r requirements.yml --roles-path ./roles`. |
| 7 | exercise1 | Playbook **completo**: `wildfly_os_deps` externo + el resto de roles **locales** del ejercicio 1. |
| 8 | exercise2 | `tests/test.yml` + `yamllint` + `ansible-lint`. |
| 9 | exercise2 | Molecule: `converge` = test del rol; `verify` = Java/tar/gzip; `default` create/destroy en OpenShift (solo Dev Spaces). |
| 10 | exercise1 | `ansible-playbook` completo (WildFly + `/sample/`) con el inventario correcto. |

---

## Notas prácticas

- **Sustituye siempre** host Gitea, usuario y rama; no dejes los marcadores `<GITEA_HOST>` / `<GITEA_USER>`.
- **Roles privados:** con HTTPS suele hacer falta token; en Dev Spaces el Gitea del laboratorio suele ser accesible con tu usuario.
- **Orden de dependencias:** si `wildfly_install` depende de otro rol, usa `dependencies` en `meta/main.yml` o el orden en `roles:` del playbook.
- **CI/CD:** `ansible-galaxy install -r requirements.yml` antes de `ansible-playbook`.

---

## Resultado esperado

- **`lab-devspaces-ansible-exercise2`** contiene el rol (tasks, defaults, vars, meta, tests), pasa yamllint/ansible-lint, el test del rol y Molecule (`default` crea y destruye la VM; `verify` solo comprueba este rol).
- **`lab-devspaces-ansible-exercise1`** ya **no** versiona `roles/wildfly_os_deps`; declara ese rol en `requirements.yml` y el playbook ejecuta la **instalación completa** (rol externo + roles locales del ejercicio 1).
