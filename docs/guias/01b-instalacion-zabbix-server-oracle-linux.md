# Instalación de Zabbix Server 7.4 en Oracle Linux 8

> Estado: **procedimiento documentado con base en las guías oficiales; pendiente de ejecutarse y validarse en el laboratorio**.

Esta guía describe una alternativa a la instalación de Zabbix Server con Docker Desktop en Windows.

La arquitectura documentada en este archivo utiliza:

```text
Oracle Linux 8.10
Zabbix Server 7.4
Zabbix Frontend
Zabbix Agent 2
MySQL Community Server 8.4 LTS
Nginx
PHP-FPM
```

Los paquetes oficiales de Zabbix 7.4 están disponibles para Oracle Linux 8. Zabbix admite MySQL 8.0.30 o superior, incluido MySQL 8.4, y admite Nginx 1.20 o superior.

---

# 1. Cuándo utilizar esta instalación

Utilizar esta opción cuando se quiera:

- Ejecutar Zabbix Server directamente sobre Linux.
- Evitar la dependencia de Docker Desktop y Windows.
- Preparar una arquitectura más cercana a producción.
- Administrar Zabbix mediante servicios `systemd`.
- Mantener el servidor, la interfaz y la base de datos en una misma máquina durante una prueba controlada.

Para instalaciones grandes, evaluar colocar la base de datos en un servidor separado.

---

# 2. Diferencia frente a la instalación con Docker

| Instalación | Característica |
|---|---|
| Windows con Docker | Los componentes se ejecutan dentro de contenedores |
| Oracle Linux con paquetes | Los componentes se instalan como servicios del sistema operativo |

En Oracle Linux se administrarán servicios como:

```text
mysqld
zabbix-server
zabbix-agent2
nginx
php-fpm
```

---

# 3. Requisitos previos

## 3.1. Equipo

Esta guía está preparada para:

```text
Oracle Linux 8.10 de 64 bits
```

## 3.2. Usuario

Ejecutar los comandos con:

```text
root
```

O con un usuario que tenga permisos de `sudo`.

Cuando se utilice `sudo`, agregarlo al inicio de los comandos administrativos.

## 3.3. Recursos mínimos para una prueba pequeña

Como punto inicial para un laboratorio:

```text
CPU: 2 vCPU
Memoria: 8 GB
Disco: 40 GB o más
```

El almacenamiento requerido depende principalmente de:

- Cantidad de equipos.
- Cantidad de métricas.
- Frecuencia de recopilación.
- Días de históricos.
- Días de tendencias.

## 3.4. Datos que deben definirse antes de comenzar

Preparar:

```text
<HOSTNAME_ZABBIX_SERVER>
<IP_ZABBIX_SERVER>
<CONTRASENA_ROOT_MYSQL>
<CONTRASENA_BD_ZABBIX>
```

No colocar las contraseñas reales en este repositorio.

---

# 4. Verificar el sistema operativo

Ejecutar en Oracle Linux:

```bash
cat /etc/os-release
uname -m
hostnamectl
```

Resultados esperados:

```text
Oracle Linux 8
x86_64
```

Confirmar la hora:

```bash
timedatectl
```

La sincronización de tiempo es importante para que eventos, gráficas y alertas coincidan.

Cuando la zona horaria sea Ciudad de México, puede configurarse con:

```bash
sudo timedatectl set-timezone America/Mexico_City
```

Validar nuevamente:

```bash
timedatectl
```

---

# 5. Actualizar paquetes base

Ejecutar:

```bash
sudo dnf clean all
sudo dnf makecache
sudo dnf update -y
```

Reiniciar cuando se haya actualizado el kernel o componentes fundamentales:

```bash
sudo reboot
```

Después de reconectarse, verificar:

```bash
uptime
```

---

# 6. Verificar que no exista otra instalación

Antes de instalar, revisar paquetes y servicios existentes:

```bash
rpm -qa | grep -Ei 'zabbix|mysql|mariadb|nginx|php'
```

Revisar servicios:

```bash
systemctl list-unit-files |
grep -Ei 'zabbix|mysql|mariadb|nginx|php-fpm'
```

Si ya existe una base de datos o instalación de Zabbix, detener el procedimiento. Primero debe documentarse si se realizará una actualización, migración o instalación nueva.

Esta guía asume un servidor limpio.

---

# 7. Instalar MySQL Community Server 8.4

Zabbix no instala automáticamente el motor de base de datos. Debe instalarse y quedar funcionando antes de crear la base de datos de Zabbix.

## 7.1. Descargar el repositorio oficial de MySQL

El archivo vigente al documentar esta guía para Oracle Linux 8 es:

```text
mysql84-community-release-el8-3.noarch.rpm
```

Descargarlo desde el sitio oficial de MySQL Yum Repository.

Cuando el archivo se haya descargado en `/tmp`, comprobarlo:

```bash
ls -l /tmp/mysql84-community-release-el8-3.noarch.rpm
```

## 7.2. Instalar el archivo del repositorio

Ejecutar:

```bash
sudo dnf install -y /tmp/mysql84-community-release-el8-3.noarch.rpm
```

Verificar los repositorios habilitados:

```bash
sudo dnf repolist enabled | grep -i mysql
```

Debe aparecer el repositorio de MySQL 8.4 Community.

> El nombre y la revisión del RPM pueden cambiar. Antes de una nueva instalación, confirmar el archivo vigente en la página oficial de MySQL.

## 7.3. Deshabilitar el módulo MySQL incluido en Oracle Linux 8

Oracle Linux 8 incluye un módulo que puede ocultar los paquetes del repositorio oficial de MySQL.

Ejecutar:

```bash
sudo dnf module disable mysql -y
```

## 7.4. Instalar MySQL Server

Ejecutar:

```bash
sudo dnf install -y mysql-community-server
```

Validar la versión instalada:

```bash
mysqld --version
```

Debe mostrar MySQL 8.4.

## 7.5. Iniciar y habilitar MySQL

Ejecutar:

```bash
sudo systemctl enable --now mysqld
```

Validar:

```bash
sudo systemctl status mysqld --no-pager
```

Resultado esperado:

```text
active (running)
```

## 7.6. Obtener la contraseña temporal de `root`

Ejecutar:

```bash
sudo grep 'temporary password' /var/log/mysqld.log
```

Guardar temporalmente la contraseña mostrada en un lugar seguro. No copiarla en esta documentación.

## 7.7. Cambiar la contraseña de `root`

Conectarse:

```bash
mysql -uroot -p
```

Ingresar la contraseña temporal.

Dentro de MySQL ejecutar:

```sql
ALTER USER 'root'@'localhost'
IDENTIFIED BY '<CONTRASENA_ROOT_MYSQL>';
```

La contraseña debe cumplir la política de MySQL: longitud suficiente, mayúsculas, minúsculas, números y caracteres especiales.

Salir:

```sql
EXIT;
```

Validar la nueva contraseña:

```bash
mysql -uroot -p -e "SELECT VERSION();"
```

---

# 8. Instalar el repositorio oficial de Zabbix 7.4

Ejecutar:

```bash
sudo rpm -Uvh \
  https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/zabbix-release-latest-7.4.el8.noarch.rpm
```

Limpiar y reconstruir la caché:

```bash
sudo dnf clean all
sudo dnf makecache
```

Verificar:

```bash
sudo dnf repolist | grep -i zabbix
```

Utilizar los paquetes oficiales del repositorio de Zabbix. No mezclar paquetes de Zabbix provenientes de otros repositorios.

---

# 9. Instalar Zabbix Server, frontend, Nginx y Agent 2

Ejecutar:

```bash
sudo dnf install -y \
  zabbix-server-mysql \
  zabbix-web-mysql \
  zabbix-nginx-conf \
  zabbix-sql-scripts \
  zabbix-selinux-policy \
  zabbix-agent2 \
  nginx
```

Comprobar los paquetes:

```bash
rpm -qa | grep -E '^zabbix|^nginx' | sort
```

Validar versiones:

```bash
zabbix_server --version
zabbix_agent2 --version
nginx -v
php -v
```

---

# 10. Crear la base de datos de Zabbix

## 10.1. Conectarse a MySQL

```bash
mysql -uroot -p
```

## 10.2. Crear base, usuario y permisos

Dentro de MySQL ejecutar:

```sql
CREATE DATABASE zabbix
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'localhost'
  IDENTIFIED BY '<CONTRASENA_BD_ZABBIX>';

GRANT ALL PRIVILEGES ON zabbix.*
  TO 'zabbix'@'localhost';

SET GLOBAL log_bin_trust_function_creators = 1;

EXIT;
```

No reutilizar la contraseña de `root` para el usuario `zabbix`.

## 10.3. Importar el esquema inicial

Ejecutar en Oracle Linux:

```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz |
mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Ingresar la contraseña de `<CONTRASENA_BD_ZABBIX>`.

La importación puede tardar varios minutos. No interrumpir el proceso.

## 10.4. Confirmar la importación

Ejecutar:

```bash
mysql -uzabbix -p -D zabbix \
  -e "SELECT COUNT(*) AS tablas FROM information_schema.tables WHERE table_schema='zabbix';"
```

El número debe ser mayor que cero.

## 10.5. Desactivar la opción temporal

Conectarse como `root`:

```bash
mysql -uroot -p
```

Ejecutar:

```sql
SET GLOBAL log_bin_trust_function_creators = 0;
EXIT;
```

---

# 11. Configurar Zabbix Server

## 11.1. Archivo

Ruta completa:

```text
/etc/zabbix/zabbix_server.conf
```

## 11.2. Comprobar que existe

```bash
sudo test -f /etc/zabbix/zabbix_server.conf \
  && echo "ARCHIVO ENCONTRADO" \
  || echo "ARCHIVO NO ENCONTRADO"
```

## 11.3. Crear respaldo

```bash
sudo cp -a \
  /etc/zabbix/zabbix_server.conf \
  /etc/zabbix/zabbix_server.conf.backup
```

## 11.4. Abrir el archivo

```bash
sudo vi /etc/zabbix/zabbix_server.conf
```

En `vi`:

```text
i       iniciar edición
Esc     salir del modo edición
:wq     guardar y salir
:q!     salir sin guardar
```

## 11.5. Configurar la base de datos

Buscar o agregar una sola vez:

```ini
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=<CONTRASENA_BD_ZABBIX>
```

Guardar y salir.

## 11.6. Proteger el archivo

```bash
sudo chown root:zabbix /etc/zabbix/zabbix_server.conf
sudo chmod 640 /etc/zabbix/zabbix_server.conf
```

## 11.7. Validar las líneas sin mostrar la contraseña

```bash
sudo grep -E '^(DBHost|DBName|DBUser)=' \
  /etc/zabbix/zabbix_server.conf

sudo grep -q '^DBPassword=' \
  /etc/zabbix/zabbix_server.conf \
  && echo "DBPassword configurado" \
  || echo "DBPassword no configurado"
```

---

# 12. Configurar Nginx para la interfaz

## 12.1. Archivo

Ruta habitual instalada por `zabbix-nginx-conf`:

```text
/etc/nginx/conf.d/zabbix.conf
```

## 12.2. Comprobar y respaldar

```bash
sudo test -f /etc/nginx/conf.d/zabbix.conf \
  && echo "ARCHIVO ENCONTRADO" \
  || echo "ARCHIVO NO ENCONTRADO"

sudo cp -a \
  /etc/nginx/conf.d/zabbix.conf \
  /etc/nginx/conf.d/zabbix.conf.backup
```

## 12.3. Abrir

```bash
sudo vi /etc/nginx/conf.d/zabbix.conf
```

## 12.4. Configurar puerto y nombre

Para el laboratorio, dejar las directivas así:

```nginx
listen 8080;
server_name _;
```

Si las líneas comienzan con `#`, quitar el carácter `#`.

No dejar dos directivas `listen` activas para el mismo servidor.

Guardar y salir.

## 12.5. Validar Nginx

```bash
sudo nginx -t
```

Resultado esperado:

```text
syntax is ok
test is successful
```

No reiniciar Nginx si la validación muestra errores.

---

# 13. Configurar PHP-FPM y zona horaria

## 13.1. Archivo

Ruta habitual:

```text
/etc/php-fpm.d/zabbix.conf
```

## 13.2. Comprobar y respaldar

```bash
sudo test -f /etc/php-fpm.d/zabbix.conf \
  && echo "ARCHIVO ENCONTRADO" \
  || echo "ARCHIVO NO ENCONTRADO"

sudo cp -a \
  /etc/php-fpm.d/zabbix.conf \
  /etc/php-fpm.d/zabbix.conf.backup
```

## 13.3. Abrir

```bash
sudo vi /etc/php-fpm.d/zabbix.conf
```

Buscar:

```ini
; php_value[date.timezone] = Europe/Riga
```

Dejar:

```ini
php_value[date.timezone] = America/Mexico_City
```

Guardar y salir.

La zona horaria seleccionada posteriormente en el perfil de cada usuario de Zabbix puede sobrescribir la zona global del frontend.

## 13.4. Validar PHP-FPM

```bash
sudo php-fpm -t
```

Resultado esperado:

```text
NOTICE: configuration file ... test is successful
```

---

# 14. Configurar SELinux

No deshabilitar SELinux para hacer funcionar Zabbix.

Verificar estado:

```bash
getenforce
```

Cuando aparezca `Enforcing`, ejecutar:

```bash
sudo setsebool -P httpd_can_connect_zabbix on
sudo setsebool -P httpd_can_network_connect_db on
```

El paquete `zabbix-selinux-policy` también instala reglas básicas para Zabbix.

Revisar denegaciones recientes cuando un servicio no funcione:

```bash
sudo ausearch -m AVC -ts recent
```

---

# 15. Abrir el firewall

## 15.1. Verificar servicio y zona

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
```

## 15.2. Abrir la interfaz web del laboratorio

```bash
sudo firewall-cmd --permanent \
  --zone=public \
  --add-port=8080/tcp
```

## 15.3. Abrir Zabbix Server para agentes y proxies

```bash
sudo firewall-cmd --permanent \
  --zone=public \
  --add-port=10051/tcp
```

## 15.4. Aplicar

```bash
sudo firewall-cmd --reload
```

## 15.5. Validar

```bash
sudo firewall-cmd --zone=public --list-ports
```

Debe incluir:

```text
8080/tcp
10051/tcp
```

En producción, restringir el puerto `10051/TCP` a las redes o direcciones de agentes y proxies autorizados.

---

# 16. Iniciar los servicios

Ejecutar:

```bash
sudo systemctl enable --now \
  zabbix-server \
  zabbix-agent2 \
  nginx \
  php-fpm
```

Reiniciar para aplicar la configuración:

```bash
sudo systemctl restart \
  zabbix-server \
  zabbix-agent2 \
  nginx \
  php-fpm
```

Validar:

```bash
for SERVICIO in mysqld zabbix-server zabbix-agent2 nginx php-fpm; do
  printf '%-20s ' "$SERVICIO"
  systemctl is-active "$SERVICIO"
done
```

Todos deben mostrar:

```text
active
```

---

# 17. Revisar puertos

Ejecutar:

```bash
sudo ss -lntp |
grep -E ':3306|:8080|:10050|:10051'
```

Resultados esperados:

| Puerto | Servicio |
|---:|---|
| `3306` | MySQL local |
| `8080` | Nginx / frontend |
| `10050` | Agent 2 |
| `10051` | Zabbix Server |

MySQL no necesita publicarse hacia otras redes cuando Zabbix Server y MySQL se encuentran en la misma máquina.

---

# 18. Revisar logs antes de abrir la interfaz

## Zabbix Server

```bash
sudo tail -n 50 /var/log/zabbix/zabbix_server.log
```

## Agent 2

```bash
sudo tail -n 50 /var/log/zabbix/zabbix_agent2.log
```

## Nginx

```bash
sudo journalctl -u nginx -n 50 --no-pager
```

## PHP-FPM

```bash
sudo journalctl -u php-fpm -n 50 --no-pager
```

## MySQL

```bash
sudo journalctl -u mysqld -n 50 --no-pager
```

No continuar cuando `zabbix-server` muestre errores de conexión o autenticación con MySQL.

---

# 19. Abrir la interfaz web

Desde otro equipo de la red, abrir:

```text
http://<IP_ZABBIX_SERVER>:8080
```

El asistente web solicitará:

```text
Tipo de base de datos: MySQL
Servidor de base de datos: localhost
Puerto: 0 o 3306
Nombre de base de datos: zabbix
Usuario: zabbix
Contraseña: <CONTRASENA_BD_ZABBIX>
Nombre del servidor: <HOSTNAME_ZABBIX_SERVER>
Zona horaria: America/Mexico_City
```

Finalizar el asistente.

Credenciales iniciales de Zabbix:

```text
Usuario: Admin
Contraseña: zabbix
```

Cambiar la contraseña inmediatamente después del primer acceso.

---

# 20. Validación final

## 20.1. Servicios

```bash
systemctl is-active mysqld
systemctl is-active zabbix-server
systemctl is-active zabbix-agent2
systemctl is-active nginx
systemctl is-active php-fpm
```

## 20.2. Página web local

```bash
curl -I http://127.0.0.1:8080
```

Debe devolver una respuesta HTTP válida, por ejemplo:

```text
HTTP/1.1 200 OK
```

## 20.3. Puerto de Zabbix Server

```bash
sudo ss -lntp | grep ':10051'
```

## 20.4. Base de datos

```bash
mysql -uzabbix -p -D zabbix \
  -e "SELECT COUNT(*) AS hosts FROM hosts;"
```

## 20.5. Interfaz

Confirmar que se puede:

- Iniciar sesión.
- Abrir **Monitoreo → Equipos**.
- Abrir **Recopilación de datos → Equipos**.
- Consultar **Administración → Cola**.

---

# 21. Checklist

- [ ] Oracle Linux 8.10 confirmado.
- [ ] Zona horaria y sincronización revisadas.
- [ ] MySQL 8.4 instalado.
- [ ] Contraseña temporal de MySQL sustituida.
- [ ] Repositorio oficial de Zabbix 7.4 instalado.
- [ ] Paquetes de servidor, frontend, Nginx y Agent 2 instalados.
- [ ] Base de datos `zabbix` creada.
- [ ] Esquema inicial importado.
- [ ] `DBPassword` configurado.
- [ ] Nginx validado con `nginx -t`.
- [ ] PHP-FPM validado.
- [ ] SELinux conservado activo.
- [ ] Firewall configurado.
- [ ] Todos los servicios en estado `active`.
- [ ] Interfaz disponible por HTTP.
- [ ] Contraseña inicial de `Admin` cambiada.
- [ ] Respaldo de archivos de configuración creado.

---

# 22. Problemas frecuentes

| Síntoma | Revisión inicial |
|---|---|
| `zabbix-server` no inicia | `/var/log/zabbix/zabbix_server.log` y credenciales de MySQL |
| Error al importar esquema | Confirmar archivo `server.sql.gz`, contraseña y base `zabbix` |
| Página no abre | `nginx -t`, servicio Nginx, puerto `8080` y firewall |
| `502 Bad Gateway` | Estado y log de `php-fpm` |
| Asistente no conecta a MySQL | Usuario, contraseña, base, SELinux y servicio `mysqld` |
| Hora incorrecta | `timedatectl`, PHP-FPM y perfil del usuario Zabbix |
| Puerto `10051` no escucha | Estado y log de `zabbix-server` |
| MySQL 8.4 no aparece en `dnf` | Módulo MySQL de Oracle Linux no deshabilitado o repositorio incorrecto |

---

# 23. Consideraciones para producción

Antes de utilizar esta arquitectura en producción, definir:

- Dimensionamiento por cantidad de métricas.
- Retención de históricos y tendencias.
- Separación de la base de datos.
- Respaldo de MySQL y archivos de configuración.
- HTTPS con certificado válido.
- Restricción de firewall por origen.
- TLS entre servidores, proxies y agentes.
- Alta disponibilidad.
- Monitoreo del propio Zabbix Server.
- Procedimiento de actualización y reversa.
- Pruebas de restauración.

---

# 24. Referencias oficiales

- [Instalación de Zabbix desde paquetes](https://www.zabbix.com/documentation/7.4/es/manual/installation/install_from_packages)
- [Requisitos de Zabbix 7.4](https://www.zabbix.com/documentation/7.4/es/manual/installation/requirements)
- [Repositorio oficial para Oracle Linux 8](https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/)
- [Creación de la base de datos](https://www.zabbix.com/documentation/7.4/en/manual/appendix/install/db_scripts)
- [Notas oficiales para Nginx](https://www.zabbix.com/documentation/7.4/en/manual/appendix/install/nginx)
- [MySQL Yum Repository](https://dev.mysql.com/downloads/repo/yum/)
- [Instalación de MySQL mediante Yum](https://dev.mysql.com/doc/mysql-repo-excerpt/8.0/en/linux-installation-yum-repo.html)
