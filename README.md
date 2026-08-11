# Taller Linux

Despliegue automatizado con Ansible de una aplicación web de cumpleaños en dos capas: servidor de aplicación con Apache y PHP sobre CentOS Stream 9, y servidor de base de datos MariaDB sobre Ubuntu Server.

La solución utiliza Ansible para automatizar la instalación, configuración y despliegue de todos los componentes necesarios.

## Arquitectura

| Servidor   | IP           | Sistema operativo | Función                                         |
| ---------- | ------------ | ----------------- | ----------------------------------------------- |
| `centos01` | `10.0.2.15`  | CentOS Stream 9   | Servidor web: Apache, PHP, PHP-FPM y aplicación |
| `ubuntu01` | `10.0.2.100` | Ubuntu Server     | Servidor de base de datos: MariaDB              |

La aplicación PHP se ejecuta en `centos01` y consulta los datos almacenados en MariaDB en `ubuntu01`.

Flujo de comunicación:

```text
Cliente
   |
   | HTTP :80
   v
centos01 - 10.0.2.15
Apache + PHP + PHP-FPM
   |
   | MariaDB :3306
   v
ubuntu01 - 10.0.2.100
MariaDB
   |
   v
Base de datos "cumples"
Tabla "cumpleanios"
```

## Contenido

| Carpeta        | Contenido                                                                |
| -------------- | ------------------------------------------------------------------------ |
| `inventory/`   | Inventario, grupos y variables de conexión                               |
| `playbooks/`   | Playbooks de base de datos, servidor web, hardening y playbook principal |
| `templates/`   | Templates Jinja2 para la aplicación PHP, SQL y configuración de MariaDB  |
| `vars/`        | Variables de la base de datos protegidas mediante Ansible Vault          |
| `collections/` | Archivo `requirements.yaml` con las collections requeridas               |
| `.git/`        | Historial y configuración del repositorio Git                            |
| `evidencia/`   | Salidas de ejecución y capturas utilizadas como evidencia del trabajo    |


## Archivos principales

```text
taller-linux-2026/
├── collections/
│   └── requirements.yaml
├── evidencia/
│   ├── appfuncionando.txt
│   ├── base-datos.txt
│   ├── hardening-ubuntu.txt
│   └── servidor-web.txt
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       └── linux.yaml
├── playbooks/
│   ├── site.yaml
│   ├── ubuntu01-database.yaml
│   ├── centos.yaml
│   ├── hardening.yaml
│   └── webserver.yaml
├── templates/
│   ├── 50-server.cnf.j2
│   ├── cumple.j2
│   ├── cumple.php.j2
│   ├── cumple.sql.j2
│   └── mariadb.cnf.j2
├── vars/
│   └── database.yaml
├── LICENSE
└── README.md

```

Los archivos `webserver.yaml`, `cumple.php.j2` y `mariadb.cnf.j2` corresponden a archivos auxiliares o versiones anteriores del desarrollo. La ejecución principal se realiza mediante `site.yaml`.

## Inventario

El inventario se encuentra en `inventory/hosts.ini`.

```ini
[centos]
centos01 ansible_host=10.0.2.15
centos02 ansible_host=10.0.2.3

[ubuntu]
ubuntu01 ansible_host=10.0.2.100
ubuntu02 ansible_host=10.0.2.101
```

Los servidores están organizados en los grupos `centos` y `ubuntu`.

El usuario utilizado para las conexiones SSH se define en:

```text
inventory/group_vars/linux.yaml
```

Actualmente:

```yaml
ansible_user: sysadmin
```

El grupo `centos` contiene dos hosts y el grupo `ubuntu` contiene dos hosts. La solución principal utiliza `centos01` como servidor de aplicación y `ubuntu01` como servidor de base de datos.

## Requisitos

### Nodo de control

Se utiliza un nodo de control con CentOS Stream 9 y Ansible instalado mediante `pipx`.

Instalamos `pipx`:

```bash
sudo dnf install pipx -y
```

Configuramos el PATH:

```bash
pipx ensurepath
```

Instalamos Ansible:

```bash
pipx install ansible-core
```

Cerramos y volvemos a abrir la terminal para que el PATH quede actualizado.

### Collections

Las collections requeridas se encuentran en:

```text
collections/requirements.yaml
```

Las instalamos con:

```bash
ansible-galaxy collection install -r collections/requirements.yaml
```

Collections utilizadas:

* `community.general`
* `community.mysql`
* `ansible.posix`

### Acceso SSH

El nodo de control debe poder conectarse mediante SSH a los servidores administrados.

Se utiliza una clave SSH, por ejemplo:

```bash
ssh-keygen -t ed25519
```

La clave pública debe copiarse a los hosts correspondientes:

```bash
ssh-copy-id sysadmin@10.0.2.15
ssh-copy-id sysadmin@10.0.2.3
ssh-copy-id sysadmin@10.0.2.100
ssh-copy-id sysadmin@10.0.2.101
```

Verificamos la conectividad desde Ansible:

```bash
ansible all -i inventory/hosts.ini -m ping
```

## Ansible Vault

Las variables de conexión de la base de datos se encuentran en:

```text
vars/database.yaml
```

Este archivo está cifrado mediante Ansible Vault y no debe contener información sensible en texto plano dentro del repositorio.

Las variables utilizadas son:

```text
DB_SERVER
DB_USER
DB_PASS
DB_DBASE
```

Para visualizar el archivo:

```bash
ansible-vault view vars/database.yaml
```

Para editarlo:

```bash
ansible-vault edit vars/database.yaml
```

Durante la ejecución del playbook se utiliza:

```text
--ask-vault-pass
```

### Verificar conectividad

Antes de ejecutar la automatización se puede verificar la conectividad con los hosts:

```bash
ansible all -i inventory/hosts.ini -m ping
```

## Ejecutar la solución completa

El playbook principal es:

```text
playbooks/site.yaml
```

Ejecutarlo mediante:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yaml \
  --ask-vault-pass \
  --ask-become-pass
```

El playbook principal importa los siguientes playbooks en orden:

```yaml
- name: Configurar servidor de base de datos
  ansible.builtin.import_playbook: ubuntu01-database.yaml

- name: Configurar servidor de aplicación
  ansible.builtin.import_playbook: centos.yaml
```

Por lo tanto, primero se configura MariaDB y posteriormente el servidor web.

### Verificar sintaxis

Antes de realizar el despliegue se puede comprobar la sintaxis:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yaml \
  --syntax-check \
  --ask-vault-pass
```

## Servidor Ubuntu — MariaDB

El playbook:

```text
playbooks/ubuntu01-database.yaml
```

se ejecuta sobre `ubuntu01`.

Realiza las siguientes tareas:

* Instala MariaDB Server.
* Inicia y habilita el servicio MariaDB mediante systemd.
* Instala `python3-pymysql`.
* Crea la base de datos definida mediante `DB_DBASE`.
* Genera mediante template el archivo SQL con el esquema y los datos iniciales.
* Comprueba si la tabla `cumpleanios` contiene datos.
* Importa el esquema y los datos iniciales cuando corresponde.
* Crea el usuario específico de la aplicación.
* Otorga permisos de lectura sobre la base de datos.
* Configura MariaDB para aceptar conexiones externas.
* Configura UFW.
* Permite el acceso al puerto TCP `3306` únicamente desde el servidor de aplicación.
* Reinicia MariaDB mediante un handler cuando se modifica su configuración.

La configuración de MariaDB se genera mediante:

```text
templates/50-server.cnf.j2
```

con:

```ini
[mysqld]
bind-address = 0.0.0.0
```

El acceso al puerto `3306` queda restringido mediante UFW a:

```text
10.0.2.15
```

que corresponde al servidor de aplicación.

## Base de datos

El esquema y los datos iniciales se generan mediante:

```text
templates/cumple.sql.j2
```

La base utilizada es:

```text
cumples
```

La tabla principal es:

```text
cumpleanios
```

y contiene:

```text
id
nombre
fecha
```

Los datos iniciales incluyen:

```text
Frodo Baggins   2005-01-14
Aragorn         2004-02-09
Arwen Undomiel  1994-12-09
```

La importación se realiza de forma condicional para evitar insertar nuevamente los datos en ejecuciones posteriores.

## Servidor CentOS — Apache y PHP

El playbook:

```text
playbooks/centos.yaml
```

se ejecuta sobre el grupo `centos`.

Realiza las siguientes tareas:

* Actualiza los paquetes del sistema.
* Instala Apache (`httpd`).
* Instala PHP.
* Instala PHP-FPM.
* Instala `php-mysqlnd` para permitir la conexión de PHP con MariaDB.
* Habilita e inicia Apache mediante systemd.
* Habilita e inicia PHP-FPM mediante systemd.
* Despliega la aplicación PHP mediante un template.
* Instala y habilita firewalld.
* Permite tráfico HTTP mediante firewalld.
* Permite tráfico HTTPS mediante firewalld.

La aplicación se genera desde:

```text
templates/cumple.j2
```

y se despliega como:

```text
/var/www/html/index.php
```

Las variables de conexión a MariaDB se obtienen desde `vars/database.yaml`, protegido mediante Ansible Vault.

## Aplicación web

La aplicación se encuentra disponible en el servidor:

```text
http://10.0.2.15/
```

La aplicación consulta la base de datos MariaDB y muestra la lista de cumpleaños almacenada en la tabla `cumpleanios`.

El resultado esperado incluye:

| Nombre         | Fecha      |
| -------------- | ---------- |
| Frodo Baggins  | 2005-01-14 |
| Aragorn        | 2004-02-09 |
| Arwen Undomiel | 1994-12-09 |

## Firewalls

### CentOS

Se utiliza `firewalld`.

Se habilita el servicio HTTP:

```text
http / TCP 80
```

También se habilita HTTPS:

```text
https / TCP 443
```

### Ubuntu

Se utiliza UFW.

La política de entrada se configura como denegada y se permite específicamente el acceso al puerto MariaDB:

```text
TCP 3306
```

únicamente desde:

```text
10.0.2.15
```

De esta forma, el servidor de base de datos no permite conexiones MariaDB desde otros hosts no autorizados.

## Hardening

El repositorio también contiene:

```text
playbooks/hardening.yaml
```

Este playbook realiza tareas iniciales de hardening sobre el grupo `ubuntu`:

* Actualiza los paquetes instalados.
* Elimina paquetes que quedaron huérfanos.
* Instala UFW.
* Restablece UFW a su configuración inicial.
* Permite conexiones salientes.
* Deniega conexiones entrantes por defecto.
* Permite conexiones SSH.
* Habilita UFW.
* Reinicia el servidor cuando corresponde.

El hardening es un playbook independiente y actualmente no es importado por `site.yaml`.

Si se ejecuta antes del playbook de base de datos, debe tenerse en cuenta que el playbook `ubuntu01-database.yaml` es el encargado de habilitar posteriormente el acceso necesario al puerto `3306`.

## Validaciones

### Validar servicios

En el servidor CentOS:

```bash
systemctl status httpd
systemctl status php-fpm
systemctl status firewalld
```

En el servidor Ubuntu:

```bash
systemctl status mariadb
systemctl status ufw
```

### Validar puertos

En el servidor web:

```bash
ss -lntp | grep ':80'
```

En el servidor de base de datos:

```bash
ss -lntp | grep ':3306'
```

### Validar la aplicación

Desde el nodo de control:

```bash
curl http://10.0.2.15/
```

También se puede acceder desde un navegador:

```text
http://10.0.2.15/
```

La aplicación debe mostrar la lista de cumpleaños obtenida desde MariaDB.

## Idempotencia

Una de las características requeridas es que una segunda ejecución no produzca cambios innecesarios.

Luego de una primera ejecución exitosa:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yaml \
  --ask-vault-pass \
  --ask-become-pass
```

se debe volver a ejecutar exactamente el mismo comando:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yaml \
  --ask-vault-pass \
  --ask-become-pass
```

En la segunda ejecución se espera que las tareas que ya se encuentran correctamente configuradas aparezcan como:

```text
ok
```

y que el `PLAY RECAP` muestre:

```text
changed=0
failed=0
unreachable=0
```

La creación de la tabla y los datos iniciales también se controla para evitar importar nuevamente información que ya existe.

## Evidencia

Se encontrarán los siguientes puntos:

1. Ejecución exitosa del playbook de hardening sobre los servidores Ubuntu.
2. Configuración de los servidores web CentOS mediante Apache, PHP-FPM, Firewalld y SELinux.
3. Configuración del servidor de base de datos Ubuntu mediante MariaDB.
4. Creación de la base de datos `cumples`, esquema y datos iniciales.
5. Configuración del acceso de MariaDB desde el servidor de aplicación.
6. Configuración de UFW y Fail2ban en los servidores Ubuntu.
7. Ejecuciones posteriores de los playbooks mostrando idempotencia.
8. `PLAY RECAP` de las ejecuciones, sin hosts inaccesibles ni tareas fallidas.

Las salidas de ejecución correspondientes se encuentran en la carpeta:

```text
evidencia/
```

## Uso de Inteligencia Artificial Generativa

Se utilizó inteligencia artificial generativa como apoyo y herramienta de consulta durante el desarrollo del trabajo, principalmente para la corrección y revisión de tareas de Ansible, utilización de módulos, configuración de servicios, resolución de errores y redacción de la documentación.

También se utilizó la documentación oficial de Ansible como referencia.

Todo el contenido utilizado en la solución fue revisado, adaptado y probado sobre los servidores reales del laboratorio antes de incorporarlo al repositorio.

## **Integrante**

**Fabián Ferreira**

Estudiante Nº 267484

Correo: [JF267484@fi365.ort.edu.uy](mailto:JF267484@fi365.ort.edu.uy)

---

## **Docente**

**Enrique Verdes**

