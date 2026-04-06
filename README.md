# lab-devspaces-ansible-exercise2

Curso práctico de desarrollo de playbooks Ansible. El fichero de referencia `deploy-wildfly.yaml` muestra el resultado objetivo; este documento sirve como guía para construirlo, refactorizarlo y validarlo con herramientas de calidad.

---

## Preparación del entorno

```bash
chmod 600 ssh_tests_connections/id_fedora_new
ansible-galaxy collection install community.general
```

**Resultado final del bloque (preparación):** clave SSH con permisos restrictivos para que SSH y Ansible la acepten; colección `community.general` disponible en el entorno de Ansible del controlador para módulos adicionales si el playbook las usa.

---

## 1. Contexto: qué hace `deploy-wildfly.yaml`

El playbook automatiza una instalación de **WildFly 39** alineada con la documentación oficial del producto. En una sola corrida sobre el grupo de inventario `servers` (con privilegios elevados):

1. **Define variables** de versión, rutas de instalación, usuario y grupo del servicio, y la URL de descarga del tarball oficial.
2. **Prepara el sistema operativo** instalando Java 21 (requisito para WildFly 39) y utilidades necesarias para descomprimir el paquete.
3. **Crea identidad de servicio**: grupo y usuario de sistema dedicados, con home en el directorio de instalación y shell restrictivo.
4. **Descarga e instala** el WildFly desde GitHub en `/opt`, evitando repetir la extracción si ya existe el indicador de instalación previsto.
5. **Normaliza la ruta de trabajo** mediante un enlace simbólico desde `/opt/wildfly-<versión>` hacia `/opt/wildfly`, de modo que scripts y servicios apunten a una ruta estable.
6. **Ajusta la configuración** `standalone.xml` para que la interfaz pública escuche en `0.0.0.0` en lugar de solo `127.0.0.1`, permitiendo acceso de red al servidor de aplicaciones.
7. **Integra systemd**: copia `launch.sh`, la unidad `wildfly.service` y `wildfly.conf` desde los ejemplos incluidos en la distribución, crea `/etc/wildfly` y arranca y habilita el servicio con recarga del demonio systemd.
8. **Despliega una aplicación de ejemplo** empaquetando un `index.html` local como WAR en el controlador, copiándolo al directorio `standalone/deployments` del WildFly para comprobar el pipeline de despliegue.

En el fichero de referencia hay tareas de **firewalld** comentadas; puedes activarlas en tu versión si el entorno lo requiere.

**Resultado final del bloque (§1 — visión global):** al ejecutar el playbook de referencia sobre `servers`, el nodo queda con WildFly instalado bajo `/opt`, servicio `wildfly` gestionado por systemd, escucha en red (interfaz pública `0.0.0.0`), y una aplicación de ejemplo accesible vía HTTP en el contexto `/sample/` (puerto 8080 por defecto), salvo que el cortafuegos impida el acceso externo.

---

## 2. Guía paso a paso (primer playbook monolítico)

Objetivo: obtener un playbook equivalente al de referencia, entendiendo el **orden lógico** y el **propósito** de cada bloque. No se repite aquí la documentación de cada módulo de Ansible; consulta la documentación oficial del módulo cuando lo uses por primera vez.

Tras cada paso aparece el **YAML concreto** tomado de `deploy-wildfly.yaml`. Al cierre de cada **bloque lógico** se resume el **estado resultante** en el nodo (ficheros, servicios o ausencia de cambios en disco).

### Paso 1 — Cabecera del play

Define un nombre descriptivo del play, el patrón de hosts (`servers` u otro grupo de tu inventario) y `become: true` si necesitas instalar paquetes y escribir bajo `/opt` y `/etc`.

```yaml
- name: Instalación de WildFly 39 siguiendo la Guía Oficial
  hosts: servers
  become: true
```

### Paso 2 — Variables del play (`vars`)

Declara al menos: versión de WildFly, directorio de instalación lógico (`/opt/wildfly`), usuario y grupo del servicio, y la URL de descarga del `.tar.gz` (puedes interpolar la versión en la cadena).

```yaml
  vars:
    wf_version: "39.0.1.Final"
    wf_install_dir: "/opt/wildfly"
    wf_user: "wildfly"
    wf_group: "wildfly"
    wf_url: "https://github.com/wildfly/wildfly/releases/download/{{ wf_version }}/wildfly-{{ wf_version }}.tar.gz"
```

**Resultado en el nodo (ficheros / estado):** los pasos 1 y 2 no escriben nada en el objetivo; solo fijan el alcance del play y las variables en memoria durante la ejecución.

**Resultado final del bloque (pasos 1–2 — cabecera y variables):** inventario y play listos para ejecutar tareas; ningún cambio persistente en el host hasta la primera tarea bajo `tasks:`.

### Paso 3 — Dependencias del sistema

Primera tarea: instalar en el nodo objetivo el JDK adecuado (Java 21 para WildFly 39), más herramientas imprescindibles para manejar el archivo comprimido (por ejemplo `tar` y `gzip`).

```yaml
  tasks:
    - name: Instalar dependencias (Java 17+ es requerido para WF 39)
      ansible.builtin.dnf:
        name:
          - java-21-openjdk-devel
          - tar
          - gzip
        state: present
```

**Resultado en el nodo:** paquetes RPM instalados (`java-21-openjdk-devel`, `tar`, `gzip`); binarios disponibles en el PATH del sistema (`java`, `tar`, `gzip`).

**Resultado final del bloque (paso 3 — dependencias OS):** sistema preparado para descargar y desempaquetar WildFly y para ejecutar la JVM requerida.

### Paso 4 — Grupo de sistema para WildFly

Crea el grupo del servicio como grupo de sistema, presente en el nodo.

```yaml
    - name: Crear grupo de sistema para WildFly
      ansible.builtin.group:
        name: "{{ wf_group }}"
        system: true
        state: present
```

**Resultado final del bloque (paso 4 — solo grupo):** en `/etc/group` existe la línea del grupo de sistema `wildfly` (GID asignado por el sistema).

### Paso 5 — Usuario de sistema para WildFly

Crea el usuario del servicio asociado a ese grupo, con home en el directorio de instalación, shell de no login y marcado como cuenta de sistema.

```yaml
    - name: Crear usuario de sistema para WildFly
      ansible.builtin.user:
        name: "{{ wf_user }}"
        group: "{{ wf_group }}"
        home: "{{ wf_install_dir }}"
        shell: /sbin/nologin
        system: true
        state: present
```

**Resultado en el nodo:** entrada en `/etc/group` para `wildfly`; entrada en `/etc/passwd` (y shadow) para `wildfly` con GID del grupo anterior, home declarada como `/opt/wildfly` y shell `/sbin/nologin`. El directorio home puede aún no existir como árbol completo hasta que se cree la instalación bajo `/opt`.

**Resultado final del bloque (pasos 4–5 — identidad del servicio):** cuenta de sistema lista para ejecutar WildFly: grupo y usuario `wildfly` creados; aún no hay árbol de producto bajo `/opt/wildfly` hasta completar la instalación.

### Paso 6 — Descarga e instalación del producto

Descarga el archivo desde la URL y descomprímelo bajo `/opt`, asignando propietario y grupo al usuario WildFly. Usa un mecanismo idempotente (por ejemplo comprobación de que ya existe un fichero clave de la instalación) para no repetir la extracción innecesariamente.

```yaml
    - name: Descargar y extraer WildFly 39
      ansible.builtin.unarchive:
        src: "{{ wf_url }}"
        dest: "/opt"
        remote_src: yes
        owner: "{{ wf_user }}"
        group: "{{ wf_group }}"
        creates: "{{ wf_install_dir }}/bin/standalone.sh"
```

**Resultado final del bloque (paso 6 — extracción):** existe `/opt/wildfly-{{ wf_version }}/` (p. ej. `/opt/wildfly-39.0.1.Final/`) con el contenido del tarball, UID/GID `wildfly`. La ruta canónica `/opt/wildfly` puede no existir aún si esta es la primera instalación y el enlace se crea en pasos posteriores.

### Paso 7 — Limpieza del enlace o directorio destino (si aplica)

Si tu diseño reutiliza siempre la misma ruta canónica (`/opt/wildfly`), incluye una tarea condicionada que elimine ese destino cuando no estés en modo check, de forma coherente con cómo quieres gestionar actualizaciones o reinstalaciones.

```yaml
    - name: Eliminar directorio destino si ya existe y no es link (Limpieza)
      ansible.builtin.file:
        path: "{{ wf_install_dir }}"
        state: absent
      when: not ansible_check_mode
```

**Resultado final del bloque (paso 7 — limpieza del destino):** si existía `/opt/wildfly` (enlace o directorio), ha sido eliminado; el directorio versionado `/opt/wildfly-{{ wf_version }}/` permanece. En modo `--check` la tarea no aplica el cambio (`when: not ansible_check_mode`).

### Paso 8 — Enlace simbólico a la versión concreta

Crea el enlace desde el directorio versionado bajo `/opt` hacia la ruta estable de trabajo, con el mismo propietario y grupo que el servicio.

```yaml
    - name: Crear enlace simbólico a la versión instalada
      ansible.builtin.file:
        src: "/opt/wildfly-{{ wf_version }}"
        dest: "{{ wf_install_dir }}"
        state: link
        owner: "{{ wf_user }}"
        group: "{{ wf_group }}"
```

**Resultado en el nodo:** directorio versionado `/opt/wildfly-39.0.1.Final/` con el árbol completo del servidor (por ejemplo `bin/standalone.sh`, `standalone/configuration/standalone.xml`, `docs/contrib/scripts/systemd/`, etc.), propietario `wildfly`. Enlace simbólico `/opt/wildfly` → `/opt/wildfly-39.0.1.Final`. Tras el paso 7, si existía un enlace o ruta previa en `/opt/wildfly`, se ha eliminado antes de recrear el enlace.

**Resultado final del bloque (pasos 6–8 — instalación y ruta estable):** producto WildFly en `/opt/wildfly-<versión>/` y accesible también como `/opt/wildfly` (symlink); listo para configuración y servicio.

### Paso 9 — Escucha en todas las interfaces

Modifica la línea correspondiente en `standalone.xml` del WildFly ya instalado (vía la ruta del enlace) para que la interfaz `public` use `0.0.0.0` por defecto en lugar de `127.0.0.1`. Asegúrate de que la expresión regular y la línea sustituta coinciden con el formato XML real del fichero.

```yaml
    - name: Configurar WildFly para que escuche en todas las IPs (0.0.0.0)
      ansible.builtin.lineinfile:
        path: "{{ wf_install_dir }}/standalone/configuration/standalone.xml"
        regexp: '<interface name="public">(\s*)<inet-address value="\$ \{jboss\.bind\.address:127\.0\.0\.1\}"/>'
        line: '        <interface name="public"><inet-address value="${jboss.bind.address:0.0.0.0}"/></interface>'
        backrefs: yes
```

**Resultado en el nodo:** en `/opt/wildfly/standalone/configuration/standalone.xml` (vía el enlace), el fragmento de la interfaz `public` queda con `inet-address` por defecto `0.0.0.0` en lugar de `127.0.0.1`, de modo que el servidor pueda escuchar en todas las interfaces de red.

**Resultado final del bloque (paso 9 — red / bind):** configuración persistente en `standalone.xml` para exponer la interfaz `public` en todas las IPs; efecto pleno tras el siguiente arranque del proceso WildFly (o reinicio del servicio).

### Paso 10 — Script de arranque para systemd

Copia desde la documentación incluida en la instalación el script `launch.sh` al árbol `bin` del WildFly, conservando permisos de ejecución.

```yaml
    - name: Instalar el archivo de servicio Systemd (Siguiendo la guía de WF)
      ansible.builtin.copy:
        src: "{{ wf_install_dir }}/docs/contrib/scripts/systemd/launch.sh"
        dest: "{{ wf_install_dir }}/bin/launch.sh"
        remote_src: yes
        mode: '0755'
```

### Paso 11 — Unidad systemd

Copia el fichero de unidad `wildfly.service` al directorio de unidades de systemd del sistema.

```yaml
    - name: Instalar el archivo de servicio Systemd (Siguiendo la guía de WF)
      ansible.builtin.copy:
        src: "{{ wf_install_dir }}/docs/contrib/scripts/systemd/wildfly.service"
        dest: "/etc/systemd/system/wildfly.service"
        remote_src: yes
```

### Paso 12 — Directorio y fichero de configuración del servicio

Crea `/etc/wildfly` y copia allí el `wildfly.conf` de ejemplo desde la instalación.

```yaml
    - name: Crear directorio de configuración para el servicio
      ansible.builtin.file:
        path: /etc/wildfly
        state: directory

    - name: Copiar configuración por defecto del servicio
      ansible.builtin.copy:
        src: "{{ wf_install_dir }}/docs/contrib/scripts/systemd/wildfly.conf"
        dest: "/etc/wildfly/wildfly.conf"
        remote_src: yes
```

**Resultado final del bloque (pasos 10–12 — ficheros de servicio en disco):** `/opt/wildfly/bin/launch.sh` con modo ejecutable; `/etc/systemd/system/wildfly.service`; directorio `/etc/wildfly/` con `wildfly.conf`. El demonio **aún puede no estar** activo hasta la tarea del paso 13.

### Paso 13 — Arranque y habilitación del servicio

Recarga la configuración de systemd, asegura que el servicio `wildfly` esté iniciado y habilitado al arranque.

```yaml
    - name: Recargar systemd y arrancar WildFly
      ansible.builtin.systemd:
        name: wildfly
        state: started
        enabled: true
        daemon_reload: true
```

**Resultado en el nodo:** `/opt/wildfly/bin/launch.sh` ejecutable; `/etc/systemd/system/wildfly.service`; directorio `/etc/wildfly/` con `wildfly.conf`. Tras la tarea systemd: unidad recargada, servicio `wildfly` activo (`systemctl is-active wildfly`) y habilitado al arranque; proceso escuchando (entre otros) en el puerto de aplicaciones HTTP configurado por defecto (8080).

**Resultado final del bloque (pasos 10–13 — integración systemd y runtime):** unidad instalada, configuración en `/etc/wildfly`, servicio `wildfly` **running** y **enabled**, con JVM escuchando en 8080 (y demás puertos por defecto de `standalone`).

### Paso 14 — (Opcional) Firewall

Si tu laboratorio usa `firewalld`, descomenta o añade la apertura del puerto HTTP del servidor de aplicaciones (por ejemplo 8080/tcp) de forma permanente e inmediata. Requiere la colección adecuada en el proyecto.

```yaml
    # - name: Abrir puerto 8080 en firewalld
    #   ansible.posix.firewalld:
    #     port: 8080/tcp
    #     permanent: true
    #     state: enabled
    #     immediate: true
```

**Resultado en el nodo (si descomentas y ejecutas):** regla permanente y efectiva en `firewalld` para `8080/tcp`; acceso externo al puerto según la política de red del host.

**Resultado final del bloque (paso 14 — firewall, opcional):** si la tarea está activa, `firewall-cmd --list-ports` (o equivalente) muestra `8080/tcp` en runtime y en configuración permanente.

### Paso 15 — Aplicación de ejemplo

Encapsula en un flujo claro: en el controlador, genera un WAR a partir de `index.html` (empaquetado como archivo zip con extensión `.war`); copia ese artefacto al directorio `standalone/deployments` del WildFly con el propietario correcto. Si el WAR se construye en localhost, usa delegación y desactiva `become` en esa parte para no exigir privilegios en la máquina desde la que ejecutas Ansible.

```yaml
    - name: Crear y desplegar la aplicación de ejemplo
      block:
        - name: Empaquetar WAR localmente
          ansible.builtin.archive:
            path: index.html
            dest: /tmp/sample.war
            format: zip
          delegate_to: localhost
          become: false

        - name: Copiar WAR al directorio de despliegue
          ansible.builtin.copy:
            src: /tmp/sample.war
            dest: "{{ wf_install_dir }}/standalone/deployments/sample.war"
            owner: "{{ wf_user }}"
            group: "{{ wf_group }}"
```

**Resultado en el nodo:** en la máquina de control: fichero `/tmp/sample.war` (ZIP con `index.html` dentro). En el objetivo: `/opt/wildfly/standalone/deployments/sample.war` con propietario `wildfly`; WildFly puede generar `sample.war.deployed` (y otros marcadores) al completar el despliegue. Respuesta HTTP esperada para comprobar el contexto, por ejemplo `curl http://<host>:8080/sample/`.

**Resultado final del bloque (paso 15 — aplicación de ejemplo):** artefacto `sample.war` desplegado; contexto `/sample/` sirve el contenido de `index.html`; comprobación típica `curl -sSf http://localhost:8080/sample/` desde el servidor o contra la IP del nodo.

Ejecuta el playbook y valida con `curl` o navegador contra la URL de la aplicación de ejemplo.

**Resultado final del bloque (§2 — playbook monolítico completo):** acumulado de todos los sub-bloques anteriores: RPM de Java y utilidades; usuario/grupo `wildfly`; `/opt/wildfly` → versión instalada; `standalone.xml` con bind `0.0.0.0`; systemd + servicio activo; opcionalmente firewalld; WAR de ejemplo y HTTP 200 en `/sample/`.

---

## 3. Segunda parte: refactorización con `tags` y `block`

Partiendo del playbook monolítico, mejora la operabilidad sin cambiar el comportamiento funcional.

### 3.1 Etiquetas (`tags`)

Asigna etiquetas coherentes a grupos de tareas para poder ejecutar solo fragmentos del play, por ejemplo:

- `deps` — paquetes del sistema.
- `wildfly_user` — grupo y usuario.
- `wildfly_install` — descarga, limpieza de destino y enlace simbólico.
- `wildfly_config` — modificación de `standalone.xml`.
- `wildfly_service` — ficheros systemd, `/etc/wildfly`, arranque del servicio.
- `app` — empaquetado y copia del WAR de ejemplo.
- `firewall` — reglas de cortafuegos, si las tienes.

Ejemplo de uso:

```bash
ansible-playbook -i inventory deploy-wildfly.yaml --tags wildfly_service
ansible-playbook -i inventory deploy-wildfly.yaml --skip-tags app
```

Ejemplo concreto (fragmento sobre tareas ya existentes):

```yaml
    - name: Instalar dependencias (Java 17+ es requerido para WF 39)
      ansible.builtin.dnf:
        name:
          - java-21-openjdk-devel
          - tar
          - gzip
        state: present
      tags: [deps]

    - name: Crear grupo de sistema para WildFly
      ansible.builtin.group:
        name: "{{ wf_group }}"
        system: true
        state: present
      tags: [wildfly_user]
```

**Resultado en el nodo:** idéntico al playbook sin etiquetas; las `tags` solo cambian **qué tareas se ejecutan** al invocar `--tags` / `--skip-tags`, no el estado final del sistema cuando corres el play completo.

### 3.2 Bloques (`block`)

Agrupa tareas relacionadas en `block` para:

- **Legibilidad**: un bloque “instalación binaria”, otro “integración systemd”, otro “aplicación de ejemplo”.
- **Rescate opcional** (`rescue`) o **siempre** (`always`): por ejemplo, si falla la copia del servicio, registrar un mensaje o intentar un diagnóstico; en `always`, tareas de limpieza o de registro de estado.

No es obligatorio añadir `rescue`/`always` si solo buscas agrupación visual; basta con `block` y comentarios en el playbook.

Ejemplo concreto (mismo bloque que el paso 15, ya presente en el playbook de referencia):

```yaml
    - name: Crear y desplegar la aplicación de ejemplo
      block:
        - name: Empaquetar WAR localmente
          ansible.builtin.archive:
            path: index.html
            dest: /tmp/sample.war
            format: zip
          delegate_to: localhost
          become: false

        - name: Copiar WAR al directorio de despliegue
          ansible.builtin.copy:
            src: /tmp/sample.war
            dest: "{{ wf_install_dir }}/standalone/deployments/sample.war"
            owner: "{{ wf_user }}"
            group: "{{ wf_group }}"
```

**Resultado en el nodo:** igual que sin `block`: `/tmp/sample.war` en el controlador y `sample.war` bajo `standalone/deployments/` en el objetivo. El `block` agrupa el flujo para lectura y para poder añadir `rescue` / `always` sin alterar el resultado en disco.

**Resultado final del bloque (§3 — tags y block):** el **estado del nodo objetivo** tras un `ansible-playbook` completo sin `--tags` coincide con el del monolito (§2). En el **repositorio**, el playbook es más operable: puedes limitar ejecución por etiquetas y estructurar tareas en `block` sin cambiar el resultado en disco salvo que omitas tareas con `--skip-tags`.

---

## 4. Tercera parte: roles, variables y handlers

Extrae los bloques anteriores a **roles** dentro del mismo proyecto de playbook (por ejemplo directorio `roles/` junto al playbook). El playbook principal queda corto: lista de roles y, si hace falta, `vars` o `vars_files` que sobrescriban los valores por defecto de cada rol.

### 4.1 Esquema sugerido de roles

| Rol | Contenido típico |
|-----|-------------------|
| `wildfly_os_deps` | Instalación de paquetes (Java, tar, gzip). |
| `wildfly_account` | Grupo y usuario de sistema. |
| `wildfly_install` | Descarga, extracción, limpieza condicional del destino, enlace simbólico. |
| `wildfly_bind` | Ajuste de `standalone.xml` para `0.0.0.0`. |
| `wildfly_systemd` | `launch.sh`, `wildfly.service`, `/etc/wildfly`, arranque y habilitación. |
| `wildfly_sample_app` | Bloque de empaquetado local y copia del WAR. |

Puedes fusionar roles si prefieres menos granularidad; lo importante es que cada rol tenga una responsabilidad clara.

**Resultado en el proyecto (ficheros):** árbol orientativo tras crear los roles (los nombres de fichero son convención Ansible):

```text
roles/
  wildfly_os_deps/tasks/main.yml
  wildfly_account/tasks/main.yml
  wildfly_install/tasks/main.yml
  wildfly_bind/tasks/main.yml
  wildfly_systemd/tasks/main.yml
  wildfly_sample_app/tasks/main.yml
deploy-wildfly.yaml   # o site.yml que importa roles
```

### 4.2 Variables

Define en cada rol un `defaults/main.yml` con los valores por defecto (versión, rutas, nombres de usuario/grupo, URL). El playbook puede pasar variables con `vars:` o con un fichero `group_vars/servers.yml` para no hardcodear secretos o IPs.

Ejemplo concreto (`roles/wildfly_install/defaults/main.yml`):

```yaml
wf_version: "39.0.1.Final"
wf_install_dir: "/opt/wildfly"
wf_user: "wildfly"
wf_group: "wildfly"
wf_url: "https://github.com/wildfly/wildfly/releases/download/{{ wf_version }}/wildfly-{{ wf_version }}.tar.gz"
```

**Resultado en el nodo:** igual que en la guía monolítica; las variables solo cambian **dónde** se definen (rol vs play). Si el playbook o `group_vars` sobrescriben `wf_version`, el árbol bajo `/opt` usará la nueva versión en rutas y URL.

### 4.3 Handlers

Sustituye o complementa tareas que hoy mezclan “cambiar fichero” y “reiniciar servicio” con el patrón **`notify`**:

- Handler **`recargar systemd`**: ejecuta recarga del demonio cuando cambian unidades bajo `/etc/systemd/system`.
- Handler **`reiniciar wildfly`**: reinicia el servicio cuando cambian `wildfly.conf`, `launch.sh` o ficheros bajo `standalone/configuration` que requieran reinicio.

En las tareas `copy` o `template` que modifiquen esos ficheros, añade `notify` con el nombre del handler. Mantén coherencia: si una tarea ya fuerza `daemon_reload` y `state: started`, al introducir handlers revisa que no dupliques reinicios innecesarios en la primera ejecución.

Ejemplo concreto (`roles/wildfly_systemd/handlers/main.yml`):

```yaml
---
- name: recargar systemd
  ansible.builtin.systemd:
    daemon_reload: true

- name: reiniciar wildfly
  ansible.builtin.service:
    name: wildfly
    state: restarted
```

Tarea que notifica (fragmento):

```yaml
    - name: Copiar unidad wildfly.service
      ansible.builtin.copy:
        src: "{{ wf_install_dir }}/docs/contrib/scripts/systemd/wildfly.service"
        dest: "/etc/systemd/system/wildfly.service"
        remote_src: yes
      notify:
        - recargar systemd
        - reiniciar wildfly
```

**Resultado en el nodo:** tras un cambio que dispare los handlers, systemd recarga unidades y el servicio `wildfly` se reinicia; en disco persisten los mismos ficheros que en el playbook monolítico (`/etc/systemd/system/wildfly.service`, etc.), con el orden de aplicación gobernado por Ansible al final del play (handlers).

### 4.4 Playbook que llama a los roles

Ejemplo de estructura (ilustrativo):

```yaml
---
- name: Instalación de WildFly con roles
  hosts: servers
  become: true
  roles:
    - role: wildfly_os_deps
    - role: wildfly_account
    - role: wildfly_install
    - role: wildfly_bind
    - role: wildfly_systemd
    - role: wildfly_sample_app
      tags: [app]
```

Las etiquetas puedes definirlas a nivel de rol en el playbook o dentro de cada rol en las tareas.

**Resultado en el nodo:** equivalente a ejecutar el playbook monolítico completo; la diferencia está en el **layout del repositorio** (roles reutilizables) y en la posibilidad de sobrescribir variables por entorno sin editar las tareas.

**Resultado final del bloque (§4 — roles):** en disco del objetivo, el mismo resultado que **Resultado final del bloque (§2)** si los roles replican las mismas tareas y variables. En el **proyecto**: directorio `roles/` con `tasks/main.yml`, `defaults/main.yml` y, si aplica, `handlers/main.yml`; playbook corto que solo ordena roles y opcionalmente `group_vars` / `vars`.

---

## 5. Calidad: yamllint, ansible-lint y Molecule

### 5.1 yamllint

Comprueba sintaxis y estilo YAML (indentación, líneas largas, documentos múltiples).

Instalación (ejemplo con pip):

```bash
pip install yamllint
```

Ejecución típica sobre el proyecto:

```bash
yamllint .
```

Opcional: añade un fichero `.yamllint` en la raíz para relajar o endurecer reglas según el estándar del curso.

**Resultado final del bloque (5.1 — yamllint):** salida sin errores (código de salida `0`) o lista de ficheros/líneas a corregir según tu `.yamllint`; el YAML del proyecto cumple las reglas de estilo acordadas en el curso.

### 5.2 ansible-lint

Analiza playbooks y roles contra buenas prácticas de Ansible (nombres, `fqcn`, uso de `become`, idempotencia, etc.).

Instalación:

```bash
pip install ansible-lint
```

Ejecución:

```bash
ansible-lint deploy-wildfly.yaml
# o, si ya tienes roles:
ansible-lint .
```

Corrige los avisos que el formador marque como obligatorios para aprobar el ejercicio.

**Resultado final del bloque (5.2 — ansible-lint):** ejecución con código de salida `0` (o solo avisos aceptados por el formador); playbooks y roles pasan las reglas de calidad configuradas (FQCN, nombres de tareas, idempotencia, etc.).

### 5.3 Molecule (prueba del playbook)

Molecule ejecuta el playbook contra un entorno efímero (contenedor, VM o delegado a un proveedor) y un playbook de verificación.

En este repositorio existe un escenario bajo `molecule/default/` con driver **delegated** y pasos `create`, `converge`, `verify` y `destroy`. Para alinearlo con **este** ejercicio de WildFly debes, como mínimo:

1. **`converge.yml`**: que importe o incluya tu playbook real (por ejemplo `deploy-wildfly.yaml` o el playbook basado en roles), no un nombre placeholder.
2. **Inventario / hosts**: el grupo o nombre de host del playbook debe coincidir con el definido en `molecule.yml` y en `host_vars` (por ejemplo si el play usa `hosts: servers`, el inventario de Molecule debe declarar ese grupo con el hostname de la plataforma de prueba).
3. **`verify.yml`**: comprobar el servicio **`wildfly`** (no otro nombre de unidad), esperar al puerto **8080** y validar la URL de la aplicación de ejemplo (**`/sample/`** o la ruta que corresponda a tu WAR), con reintentos razonables.

Instalación típica:

```bash
pip install molecule molecule-plugins ansible
```

Ejecución del escenario por defecto:

```bash
cd /ruta/al/proyecto
molecule test
```

Secuencia manual (depuración):

```bash
molecule create
molecule converge
molecule verify
molecule destroy
```

Ajusta `create.yml` y las credenciales en variables de grupo si tu entorno de laboratorio usa OpenShift/KubeVirt u otro backend; sin un nodo alcanzable por SSH, `converge` no podrá aplicar el playbook.

**Resultado final del bloque (5.3 — Molecule):** `molecule test` completa la secuencia `destroy` → `create` → `converge` → `verify` → `destroy` con código de salida `0`; el playbook se aplica en el entorno de prueba y `verify.yml` confirma servicio, puerto y URL de la aplicación. Si falla un paso, la salida indica qué escenario (`converge` / `verify`) corregir.

**Resultado final del bloque (§5 — calidad):** pipeline local repetible: YAML válido y consistente (yamllint), buenas prácticas Ansible (ansible-lint) y prueba de extremo a extremo (Molecule) alineada con WildFly y `/sample/`.

---

## Resumen

1. Construye el playbook monolítico siguiendo los pasos de la sección 2.  
2. Refactoriza con `tags` y `block` (sección 3).  
3. Extrae a roles, centraliza variables y añade handlers (sección 4).  
4. Valida con yamllint, ansible-lint y Molecule (sección 5).

El fichero `deploy-wildfly.yaml` del repositorio es la referencia de resultado; la práctica consiste en reproducirlo, mejorarlo estructuralmente y demostrar calidad con las herramientas anteriores.

---

## Comandos auxiliares

Ajusta el inventario (`inventory`) con la IP o nombre de host real del nodo y el usuario SSH correspondiente.

```bash
ansible-playbook -i inventory deploy-wildfly.yaml
```

Comprobaciones manuales tras el despliegue (ejemplo):

```bash
ssh user1@10.234.2.26 -i ssh_tests_connections/id_fedora_new
curl localhost:8080/sample/
```

**Resultado final del bloque (comandos auxiliares):** playbook ejecutado contra el inventario ajustado; sesión SSH al nodo posible con la clave indicada; `curl` devuelve el HTML de la app de ejemplo (código HTTP 200) si WildFly y el despliegue `/sample/` están correctos.
