# lab-devspaces-ansible-exercise2

## ¿Qué es un rol en Ansible?

Un **rol** es una unidad reutilizable que agrupa tareas, datos, plantillas y metadatos bajo un **nombre** (`wildfly_os_deps`, `nginx`, etc.). El playbook solo declara qué roles aplicar y en qué orden; Ansible carga automáticamente los ficheros convencionales de cada rol (`tasks/main.yml`, `defaults/main.yml`, etc.). Así evitas playbooks enormes, puedes **versionar y compartir** un rol en Git (o Galaxy) y **reutilizarlo** en varios proyectos sin copiar y pegar YAML.

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
├── vars/
│   └── main.yml           # variables internas del rol (alta precedencia)
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
- name: recargar httpd
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

Variables con **precedencia mayor** que `defaults`; suelen ser constantes del rol o valores que no quieres que el usuario cambie sin editar el rol.

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

#### `README.md`

Documentación para humanos: qué hace el rol, variables disponibles y ejemplo de uso en un playbook. No lo interpreta Ansible, pero es esencial en repos compartidos.

---

En la práctica de este laboratorio bastará con **`tasks/`**, **`defaults/`** y **`meta/`** para el rol de ejemplo; **`handlers/`**, **`files/`**, **`templates/`** y **`vars/`** entran en juego cuando el rol crece o necesita plantillas y reinicios coordinados.

---

## En qué consiste este laboratorio

**Objetivo del ejercicio:** extraer un rol del playbook de **`lab-devspaces-ansible-exercise2`** a un **repositorio Git propio** y **consumirlo desde otro proyecto de playbook** mediante `ansible-galaxy` y un fichero **`requirements.yml`**.

Como referencia se usa el rol sugerido **`wildfly_os_deps`** (instalación de paquetes: Java, `tar`, `gzip`) de la tabla de la sección **4.1** del README de exercise2. Los mismos pasos sirven para `wildfly_account`, `wildfly_install`, `wildfly_bind`, `wildfly_systemd` o `wildfly_sample_app`, ajustando tareas, variables y dependencias entre roles.

### Objetivos concretos

1. Tener el código del rol en **un repositorio Git independiente**, con la estructura estándar de rol Ansible.
2. En el **proyecto del playbook** (por ejemplo el de exercise2, u otro), **no** duplicar ese árbol bajo `roles/` en el repositorio del playbook, sino **declarar el origen en `requirements.yml`** e instalar el rol con `ansible-galaxy`.
3. El playbook sigue referenciando el rol por **nombre** (`wildfly_os_deps`) como si estuviera en `roles/`, pero el contenido llega desde Git.

---

## Prerrequisitos

- Haber seguido **exercise2** hasta entender la sección **4** (roles sugeridos) o haber ya extraído al menos un rol localmente en `roles/<nombre_rol>/`.
- `git`, `ansible` y `ansible-galaxy` disponibles en el workspace (por ejemplo el contenedor del Devfile de este laboratorio).
- Un **remoto Git** donde publicar el rol: GitHub, GitLab u otro (URL HTTPS o SSH según política del laboratorio).

---

## Parte A — Crear el repositorio del rol (proyecto separado)

### A.1 Crear el repositorio vacío en la forja

En tu plataforma Git, crea un repositorio nuevo, por ejemplo `ansible-role-wildfly-os-deps`. **No** inicialices con README si vas a empujar un árbol local ya preparado (evita conflictos en el primer push).

### A.2 Clonar y estructura en la raíz del repo

Convención habitual para un repo **de un solo rol**: la **raíz del repositorio es la raíz del rol** (no una subcarpeta `roles/wildfly_os_deps` dentro del repo). Así `ansible-galaxy install -r requirements.yml` instala el contenido con el nombre que declares.

Árbol mínimo:

```text
ansible-role-wildfly-os-deps/
├── defaults/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
└── README.md          # opcional pero recomendable
```

### A.3 Contenido de `tasks/main.yml`

Migra solo las tareas que correspondan a este rol. Para `wildfly_os_deps`, equivalente al bloque de dependencias del playbook monolítico de exercise2:

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

Si el rol combinara más bloques, añade aquí el resto de tareas y usa nombres de variables coherentes con `defaults/main.yml`.

### A.4 Contenido de `defaults/main.yml`

Centraliza valores por defecto (aunque este rol concreto pueda no necesitar variables, es buena práctica dejar el fichero):

```yaml
---
# Valores por defecto del rol wildfly_os_deps (amplía si añades variables a las tareas)
```

Si parametrizas paquetes o versiones de Java, decláralas aquí y úsalas en `tasks/main.yml` con `{{ nombre_variable }}`.

### A.5 `meta/main.yml` (metadatos del rol)

Facilita documentación, compatibilidad y futuras publicaciones en Galaxy:

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

Ajusta `author`, `license` y `platforms` a lo que exija tu organización.

### A.6 Primer commit y push

```bash
cd ansible-role-wildfly-os-deps
git init
git add defaults meta tasks README.md
git commit -m "Initial import: wildfly_os_deps role"
git remote add origin <URL_DEL_REPOSITORIO_GIT>
git branch -M main
git push -u origin main
```

Para versiones estables, crea **tags** (`v1.0.0`) y usa el tag en `requirements.yml` (ver parte B).

---

## Parte B — Consumir el rol desde el proyecto del playbook

Trabaja en el directorio del **playbook** (por ejemplo `lab-devspaces-ansible-exercise2` o una copia solo con inventario y playbooks).

### B.1 Quitar el rol duplicado del repo del playbook (recomendado)

Si ya tenías `roles/wildfly_os_deps/` en el mismo repo que el playbook:

- Elimina esa carpeta del repositorio del playbook (o deja de versionarla) para que la única fuente de verdad sea Git del rol.
- Asegúrate de que `.gitignore` **no** ignore `roles/` si vas a versionar otros roles locales; si solo usas roles instalados por Galaxy, puedes ignorar `roles/wildfly_os_deps` tras instalar, o no versionar `roles/` y documentar que hay que ejecutar `ansible-galaxy install` tras clonar (estrategia habitual).

### B.2 Crear `requirements.yml` en la raíz del proyecto del playbook

Fichero `requirements.yml` (solo **roles** en este ejemplo):

```yaml
---
roles:
  - name: wildfly_os_deps
    src: git+<URL_DEL_REPOSITORIO_GIT>
    scm: git
    version: main
```

Sustituye:

- `<URL_DEL_REPOSITORIO_GIT>` por la URL clonable (HTTPS o SSH). Ejemplos:
  - `git+https://github.com/org/ansible-role-wildfly-os-deps.git`
  - `git+git@gitlab.example.com:ansible/ansible-role-wildfly-os-deps.git`
- `version` por la rama (`main`, `develop`) o por un **tag** (`v1.0.0`) para instalaciones reproducibles.

Formato alternativo equivalente (sin prefijo `git+` en algunos entornos):

```yaml
---
roles:
  - name: wildfly_os_deps
    src: https://github.com/org/ansible-role-wildfly-os-deps.git
    scm: git
    version: main
```

### B.3 Instalar roles en el proyecto del playbook

Desde el directorio donde está `requirements.yml`:

```bash
ansible-galaxy install -r requirements.yml
```

Por defecto los roles se instalan en `~/.ansible/roles` o en la ruta definida por `roles_path` en `ansible.cfg`. Para que queden **junto al playbook** en `./roles/` (típico en proyectos de laboratorio), usa:

```bash
ansible-galaxy install -r requirements.yml --roles-path ./roles
```

Comprueba que existe `roles/wildfly_os_deps/tasks/main.yml` (u otra ruta bajo `roles_path`).

### B.4 Configurar `ansible.cfg` (opcional pero claro)

Si usas `--roles-path ./roles` siempre, puedes fijarlo en `ansible.cfg` en la raíz del proyecto del playbook:

```ini
[defaults]
roles_path = ./roles
```

Así `ansible-playbook` resuelve el rol por nombre sin flags extra.

### B.5 Playbook que referencia el rol

El play no cambia respecto a tener el rol “a mano” en el repo: el nombre debe coincidir con `name` en `requirements.yml`.

```yaml
---
- name: Instalación de WildFly con rol externo (ejemplo parcial)
  hosts: servers
  become: true
  roles:
    - role: wildfly_os_deps
    # - role: wildfly_account
    # - role: wildfly_install
    # ... resto de roles locales o también en requirements.yml
```

Si los demás roles siguen solo en el repo del playbook, déjalos en `roles/` local; mezclar roles **Galaxy/Git** y roles **locales** es habitual.

### B.6 Verificación

```bash
ansible-playbook -i inventory deploy-wildfly.yaml --syntax-check
ansible-playbook -i inventory deploy-wildfly.yaml --check
```

(Ajusta el nombre del playbook si usas uno que solo incluye el rol de prueba.)

---

## Resumen de pasos (checklist)

| Paso | Dónde | Acción |
|------|--------|--------|
| 1 | Forja Git | Crear repo del rol (ej. `ansible-role-wildfly-os-deps`). |
| 2 | Repo del rol | Estructura `tasks/`, `defaults/`, `meta/` en la raíz. |
| 3 | Repo del rol | Migrar tareas y variables del bloque elegido de exercise2. |
| 4 | Repo del rol | `git commit` y `git push` (y tags si versionas). |
| 5 | Repo del playbook | Añadir `requirements.yml` con `name`, `src`, `scm`, `version`. |
| 6 | Repo del playbook | `ansible-galaxy install -r requirements.yml --roles-path ./roles`. |
| 7 | Repo del playbook | Opcional: `ansible.cfg` → `roles_path = ./roles`. |
| 8 | Repo del playbook | Playbook con `roles: [ wildfly_os_deps, ... ]`. |
| 9 | — | `ansible-playbook` con inventario correcto. |

---

## Notas prácticas

- **Roles privados:** con HTTPS suele hacer falta token o credencial helper; con SSH, clave en el workspace y `src: git+git@...`.
- **Orden de dependencias:** si más adelante `wildfly_install` depende de variables definidas en otro rol, usa `dependencies` en `meta/main.yml` del rol consumidor o mantén el orden en `roles:` del playbook.
- **CI/CD:** en pipeline, ejecuta siempre `ansible-galaxy install -r requirements.yml` antes del `ansible-playbook`.
- **Mismo rol para varios playbooks:** varios repositorios de playbook pueden apuntar al mismo `src` y `version` en su propio `requirements.yml`.

---

## Resultado esperado

- Un **repositorio Git** contiene un rol Ansible reutilizable con estructura estándar.
- El **proyecto del playbook** declara ese rol en **`requirements.yml`**, lo instala con **`ansible-galaxy`**, y el playbook lo invoca por **nombre** sin copiar el código del rol en el repo del playbook (salvo que elijas versionar `roles/` tras instalar; lo habitual es no versionar artefactos generados y sí versionar `requirements.yml`).
