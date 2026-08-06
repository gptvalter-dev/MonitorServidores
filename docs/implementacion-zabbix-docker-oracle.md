# Guía de exploración, instalación y parametrización de Zabbix

> Fecha de corte: **6 de agosto de 2026**  
> Estado: **exploración técnica en curso**. El servidor Zabbix, el agente de Oracle Linux, las comprobaciones activas de Linux y el monitoreo básico de Oracle Database ya funcionan. Continúan pendientes la interpretación completa de métricas, la parametrización de métricas personalizadas, las notificaciones y el monitoreo de aplicaciones.

Esta guía concentra el procedimiento completo aprendido durante el laboratorio. Está escrita para que una persona sin experiencia previa en Zabbix pueda repetir la instalación, entender qué está configurando, validar cada paso y diagnosticar los errores más frecuentes.

Las incidencias detalladas se documentan en la [base de conocimiento](base-conocimiento/README.md).

## Forma de documentar los cambios

Cuando un procedimiento requiera modificar un archivo, la guía debe indicar siempre:

1. En qué equipo se realiza el cambio.
2. Qué terminal o aplicación debe abrirse.
3. La ruta completa del archivo.
4. Cómo comprobar que el archivo existe.
5. Cómo crear un respaldo antes de modificarlo.
6. El comando exacto para abrirlo.
7. Qué líneas deben buscarse o agregarse.
8. Cómo guardar y cerrar el editor.
9. Cómo comprobar que el cambio quedó aplicado.
10. Qué resultado debe observarse antes de continuar.

No se debe asumir que la persona conoce la ubicación de un archivo, el uso del editor o la forma de validar un cambio.

---

# 1. Objetivo del laboratorio

El propósito de esta prueba es determinar si Zabbix puede utilizarse para monitorear de forma centralizada:

1. Sistemas operativos de servidores.
2. Bases de datos Oracle.
3. Aplicaciones, servicios, procesos, puertos y endpoints.

La prueba también debe permitir responder:

- Qué métricas se obtienen con las plantillas estándar.
- Cómo interpretar cada métrica.
- Qué valores son normales para cada servidor.
- Qué umbrales deben cambiarse.
- Cómo crear métricas personalizadas.
- Cómo generar avisos de problema y recuperación.
- Qué diferencias existen entre el laboratorio y producción.

## Avance estimado

| Frente | Estado | Avance estimado |
|---|---|---:|
| Zabbix Server con Docker | Operativo | 90% |
| Instalación de Zabbix Agent 2 | Operativo | 100% |
| Comunicación pasiva con Oracle Linux | Validada | 100% |
| Comprobaciones activas de Linux | Validada | 100% |
| Monitoreo básico de Oracle Database | En exploración | 60% |
| Interpretación de métricas | En exploración | 20% |
| Métricas personalizadas | No iniciado | 0% |
| Alertas y notificaciones | No iniciado | 0% |
| Monitoreo de aplicaciones | No iniciado | 0% |

**Avance general estimado del laboratorio: 40%.**

---

# 2. Conceptos básicos antes de iniciar

## Zabbix Server

Es el componente central. Recibe datos de los agentes, almacena históricos, evalúa condiciones y genera problemas.

## Zabbix Agent 2

Es el servicio instalado en el servidor que se desea monitorear. Obtiene datos del sistema operativo y de integraciones como Oracle.

## Comprobación activa

El agente inicia la comunicación y envía información al Zabbix Server.

```text
Servidor monitoreado ─────> Zabbix Server
```

La directiva principal es:

```ini
ServerActive=<IP_ZABBIX_SERVER>:<PUERTO_ZABBIX_SERVER>
```

## Comprobación pasiva

El Zabbix Server inicia la consulta contra el agente.

```text
Zabbix Server ─────> Servidor monitoreado:10050
```

La directiva principal es:

```ini
Server=<IP_AUTORIZADA>
```

## Equipo o host

Es el registro del servidor dentro de Zabbix.

## Plantilla

Es un conjunto de métricas, reglas de descubrimiento, gráficas y alertas reutilizables.

## Elemento o métrica

Es un valor individual, por ejemplo:

```text
CPU utilizada
Memoria disponible
Espacio libre
Oracle Ping
Sesiones Oracle
```

## Trigger

Es una condición que determina cuándo una métrica representa un problema.

## Macro

Es un parámetro reutilizable. Se utiliza para direcciones, usuarios, contraseñas y umbrales.

Ejemplo:

```text
{$ORACLE.SERVICE}
{$ORACLE.USER}
{$ORACLE.PGA.USE.MAX.WARN}
```

---

# 3. Seguridad y manejo de información

Este repositorio es público. Utilizar valores genéricos:

- `<IP_ZABBIX_SERVER>`
- `<IP_ORIGEN_COMPROBACION_PASIVA>`
- `<IP_ORACLE_LINUX>`
- `<HOSTNAME_WINDOWS>`
- `<HOSTNAME_LINUX>`
- `<ORACLE_SERVICE>`
- `<ORACLE_HOME>`
- `<CONTRASENA_SEGURA>`

No publicar:

- Contraseñas.
- Direcciones internas reales.
- Usuarios de aplicación.
- Nombres reales de servidores productivos.
- Cadenas de conexión productivas.

---

# 4. Arquitectura del laboratorio

```text
Windows
├── Docker Desktop + WSL 2
│   ├── MySQL 8.4
│   ├── Zabbix Server 7.4
│   └── Zabbix Web Nginx
├── Zabbix Agent 2 para Windows
│
└── Red interna
    └── Oracle Linux 8.10
        ├── Zabbix Agent 2
        └── Oracle Database 19c
```

| Componente | Puerto publicado | Destino |
|---|---:|---:|
| Interfaz web Zabbix | `8080` | `8080` |
| HTTPS web Zabbix | `8443` | `8443` |
| Zabbix Server | `11051` | `10051` del contenedor |
| Agent 2 de Windows | `11050` | Windows local |
| Agent 2 de Oracle Linux | `10050` | Oracle Linux |

## Flujo utilizado

```text
Comprobaciones activas Linux
Oracle Linux ────────────────> Zabbix Server:11051

Comprobaciones pasivas Oracle
Zabbix Server ───────────────> Agent 2:10050 ─────────> Oracle Database
```

---

# Parte A. Preparar Windows y Docker Desktop

# 5. Verificar WSL 2

Ejecutar en PowerShell:

```powershell
wsl --version
```

Verificar características de Windows:

```powershell
Get-WindowsOptionalFeature -Online |
Where-Object { $_.FeatureName -in @(
    "VirtualMachinePlatform",
    "Microsoft-Windows-Subsystem-Linux"
)} |
Select-Object FeatureName, State
```

Ambas deben aparecer habilitadas.

# 6. Instalar Docker Desktop

1. Descargar Docker Desktop para Windows.
2. Ejecutar el instalador.
3. Utilizar el backend WSL 2.
4. Reiniciar Windows si se solicita.
5. Abrir Docker Desktop.
6. Esperar a que el motor Linux se encuentre iniciado.

Validar:

```powershell
docker run hello-world
```

Resultado esperado:

```text
Hello from Docker!
```

Consulta de errores: [Docker Desktop y Windows](base-conocimiento/docker-windows.md).

---

# Parte B. Instalar Zabbix 7.4 con Docker Compose

# 7. Clonar el repositorio oficial

Abrir **PowerShell** en Windows y ejecutar:

```powershell
mkdir C:\docker
cd C:\docker
git clone https://github.com/zabbix/zabbix-docker.git
cd zabbix-docker
git checkout 7.4
```

Comprobar la carpeta actual:

```powershell
Get-Location
```

Resultado esperado:

```text
C:\docker\zabbix-docker
```

Verificar Compose:

```powershell
docker compose version
```

# 8. Configurar variables y puertos

Este cambio se realiza en el equipo **Windows** donde se clonó el repositorio de Zabbix Docker.

## 8.1. Ubicación del archivo

La ruta completa del archivo es:

```text
C:\docker\zabbix-docker\.env
```

El punto inicial forma parte del nombre. El archivo se llama exactamente:

```text
.env
```

No debe llamarse `.env.txt`.

## 8.2. Abrir PowerShell y entrar a la carpeta

Abrir **PowerShell** y ejecutar:

```powershell
cd C:\docker\zabbix-docker
Get-Location
```

Resultado esperado:

```text
C:\docker\zabbix-docker
```

## 8.3. Comprobar que el archivo existe

Ejecutar:

```powershell
Test-Path .\.env
```

Resultado esperado:

```text
True
```

También puede mostrarse la información del archivo con:

```powershell
Get-Item -Force .\.env
```

Si `Test-Path` devuelve `False`, detener el procedimiento y comprobar que el repositorio se clonó en `C:\docker\zabbix-docker`. No crear un archivo vacío sin revisar primero la instalación.

## 8.4. Crear un respaldo

Antes de modificar el archivo, ejecutar:

```powershell
Copy-Item .\.env .\.env.backup
```

Comprobar que existen ambos archivos:

```powershell
Get-ChildItem -Force .env*
```

Deben aparecer, por lo menos:

```text
.env
.env.backup
```

## 8.5. Abrir el archivo con Bloc de notas

Ejecutar desde la misma ventana de PowerShell:

```powershell
notepad .\.env
```

Se abrirá el archivo existente en Bloc de notas.

## 8.6. Buscar y modificar las variables

Dentro de Bloc de notas:

1. Presionar `Ctrl + F`.
2. Buscar `ZABBIX_WEB_NGINX_HTTP_PORT`.
3. Modificar su valor a `8080`.
4. Buscar `ZABBIX_WEB_NGINX_HTTPS_PORT`.
5. Modificar su valor a `8443`.
6. Buscar `ZABBIX_SERVER_PORT`.
7. Modificar su valor a `11051`.

Las líneas deben quedar así:

```dotenv
ZABBIX_WEB_NGINX_HTTP_PORT=8080
ZABBIX_WEB_NGINX_HTTPS_PORT=8443
ZABBIX_SERVER_PORT=11051
```

Si una variable ya existe, modificar esa línea. No agregar otra línea duplicada con el mismo nombre.

Si una variable no existe, agregarla una sola vez al final del archivo.

## 8.7. Qué significa cada puerto

| Variable | Función |
|---|---|
| `ZABBIX_WEB_NGINX_HTTP_PORT=8080` | Puerto HTTP usado para abrir la interfaz web desde Windows |
| `ZABBIX_WEB_NGINX_HTTPS_PORT=8443` | Puerto HTTPS reservado para la interfaz web |
| `ZABBIX_SERVER_PORT=11051` | Puerto publicado en Windows para que los agentes activos se conecten al Zabbix Server |

En este laboratorio se utiliza `11051` porque el puerto habitual `10051` no estaba disponible en Windows. Dentro del contenedor, Zabbix Server continúa escuchando en `10051`.

## 8.8. Guardar y cerrar

En Bloc de notas:

1. Presionar `Ctrl + S`.
2. Cerrar Bloc de notas.

Como se abrió un archivo existente, Bloc de notas debe guardar el cambio sobre `.env` y no crear `.env.txt`.

## 8.9. Comprobar lo guardado

Regresar a PowerShell y ejecutar:

```powershell
Get-Content .\.env |
Select-String 'ZABBIX_WEB_NGINX_HTTP_PORT|ZABBIX_WEB_NGINX_HTTPS_PORT|ZABBIX_SERVER_PORT'
```

Resultado esperado:

```text
ZABBIX_WEB_NGINX_HTTP_PORT=8080
ZABBIX_WEB_NGINX_HTTPS_PORT=8443
ZABBIX_SERVER_PORT=11051
```

Comprobar también que no se creó accidentalmente `.env.txt`:

```powershell
Get-ChildItem -Force .env*
```

## 8.10. Definir la variante de imágenes

En la misma ventana de PowerShell ejecutar:

```powershell
$env:OS="alpine"
```

Comprobar el valor:

```powershell
$env:OS
```

Resultado esperado:

```text
alpine
```

Esta variable solo se conserva en la ventana actual de PowerShell. Debe ejecutarse nuevamente cuando se abra una nueva sesión antes de utilizar `docker compose`.

## 8.11. Validar la configuración antes de iniciar

Ejecutar:

```powershell
docker compose config --images
```

Deben aparecer imágenes `alpine-7.4-latest` para los componentes Zabbix.

Después ejecutar:

```powershell
docker compose config |
Select-String '8080|8443|11051'
```

La salida debe mostrar los puertos configurados. Si aparece un error, no iniciar los contenedores hasta corregir el archivo `.env`.

# 9. Iniciar Zabbix

Abrir PowerShell y entrar a la carpeta:

```powershell
cd C:\docker\zabbix-docker
```

Definir la variante de imágenes:

```powershell
$env:OS="alpine"
```

Iniciar los contenedores:

```powershell
docker compose up -d
```

Validar:

```powershell
docker compose ps
```

Estado esperado:

- MySQL: `healthy`.
- Zabbix Server: `Up`.
- Zabbix Web: `healthy`.

Acceso:

```text
http://localhost:8080
```

Credenciales iniciales del laboratorio:

```text
Usuario: Admin
Contraseña: zabbix
```

No conservar esta contraseña en ambientes expuestos o productivos.

# 10. Operación básica de Docker

Antes de ejecutar los comandos, abrir PowerShell y entrar a:

```powershell
cd C:\docker\zabbix-docker
$env:OS="alpine"
```

Iniciar o actualizar:

```powershell
docker compose up -d
```

Detener sin borrar datos:

```powershell
docker compose stop
```

Reanudar:

```powershell
docker compose start
```

Eliminar contenedores y redes:

```powershell
docker compose down
```

Consulta de errores: [Zabbix en Docker Compose](base-conocimiento/zabbix-docker.md).

---

# Parte C. Monitorear Windows

# 11. Instalar y configurar Zabbix Agent 2

Instalar el MSI oficial de Zabbix Agent 2.

## 11.1. Ubicación del archivo de configuración

La ruta predeterminada es:

```text
C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf
```

## 11.2. Abrir PowerShell como administrador

1. Abrir el menú Inicio.
2. Escribir `PowerShell`.
3. Hacer clic derecho en **Windows PowerShell**.
4. Elegir **Ejecutar como administrador**.

## 11.3. Comprobar que el archivo existe

```powershell
Test-Path 'C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf'
```

Resultado esperado:

```text
True
```

## 11.4. Crear respaldo

```powershell
Copy-Item `
  'C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf' `
  'C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf.backup'
```

## 11.5. Abrir el archivo como administrador

```powershell
Start-Process notepad.exe `
  -Verb RunAs `
  -ArgumentList '"C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf"'
```

Buscar y dejar activas estas líneas:

```ini
Hostname=<HOSTNAME_WINDOWS>
Server=127.0.0.1
ServerActive=127.0.0.1:11051
ListenPort=11050
```

La línea `ListenPort` debe estar activa, sin `#` al inicio.

No mantener dos líneas activas con el mismo parámetro.

Guardar con `Ctrl + S` y cerrar Bloc de notas.

## 11.6. Comprobar la configuración guardada

```powershell
Select-String `
  -Path 'C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf' `
  -Pattern '^(Hostname|Server|ServerActive|ListenPort)='
```

Validar la sintaxis:

```powershell
& "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" `
  -c "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -T
```

Iniciar:

```powershell
Start-Service "Zabbix Agent 2"
Get-Service "Zabbix Agent 2"
```

# 12. Crear el host Windows

Ruta:

```text
Recopilación de datos → Equipos → Crear equipo
```

Configuración:

```text
Nombre del equipo: <HOSTNAME_WINDOWS>
Grupo: Windows servers
Plantilla: Windows by Zabbix agent active
Interfaces: ninguna
Estado: habilitado
```

Para comprobaciones activas no se requiere interfaz.

Validar en:

```text
Monitoreo → Últimos datos
```

Métricas básicas esperadas:

- CPU.
- Memoria.
- Discos.
- Red.
- Servicios.
- Disponibilidad activa.

Consulta de errores: [Agentes Zabbix](base-conocimiento/agentes-zabbix.md).

---

# Parte D. Instalar y configurar Agent 2 en Oracle Linux

# 13. Instalar el repositorio oficial

Respaldar una configuración manual anterior, cuando exista:

```bash
cp -a /opt/zabbix/conf/zabbix_agentd.conf \
      /root/zabbix_agentd.conf.opt.backup
```

Instalar el repositorio:

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/zabbix-release-latest-7.4.el8.noarch.rpm
dnf clean all
dnf makecache
```

Instalar Agent 2:

```bash
dnf install -y zabbix-agent2
```

Rutas principales:

```text
Ejecutable:    /usr/sbin/zabbix_agent2
Configuración: /etc/zabbix/zabbix_agent2.conf
Servicio:      zabbix-agent2
Log:           /var/log/zabbix/zabbix_agent2.log
```

# 14. Configurar Agent 2

Este cambio se realiza en **Oracle Linux**.

## 14.1. Ruta del archivo

```text
/etc/zabbix/zabbix_agent2.conf
```

## 14.2. Comprobar que existe

```bash
sudo ls -l /etc/zabbix/zabbix_agent2.conf
```

## 14.3. Crear un respaldo con fecha y hora

```bash
sudo cp -a \
  /etc/zabbix/zabbix_agent2.conf \
  /etc/zabbix/zabbix_agent2.conf.backup.$(date +%Y%m%d-%H%M%S)
```

Comprobar los respaldos:

```bash
sudo ls -l /etc/zabbix/zabbix_agent2.conf*
```

## 14.4. Abrir el archivo con `vi`

```bash
sudo vi /etc/zabbix/zabbix_agent2.conf
```

Uso básico de `vi`:

1. Presionar `/` y escribir el nombre del parámetro que se desea buscar, por ejemplo `/ServerActive`.
2. Presionar `Enter`.
3. Presionar `i` para entrar al modo de edición.
4. Modificar el texto.
5. Presionar `Esc` para salir del modo de edición.
6. Escribir `:wq` y presionar `Enter` para guardar y salir.
7. Para salir sin guardar, presionar `Esc`, escribir `:q!` y presionar `Enter`.

Configuración base:

```ini
Server=<IP_ZABBIX_SERVER>,<IP_ORIGEN_COMPROBACION_PASIVA>
ServerActive=<IP_ZABBIX_SERVER>:11051
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

## Qué controla cada línea

```ini
Server=
```

Autoriza las direcciones que pueden consultar pasivamente al agente.

```ini
ServerActive=
```

Indica la dirección a la que el agente enviará las comprobaciones activas.

```ini
Hostname=
```

Debe coincidir exactamente con el campo **Nombre del equipo** en Zabbix.

```ini
ListenPort=
```

Puerto donde el agente recibe comprobaciones pasivas.

## 14.5. Comprobar lo guardado

```bash
sudo grep -E '^(Server|ServerActive|Hostname|ListenPort)=' \
  /etc/zabbix/zabbix_agent2.conf
```

Validar la sintaxis:

```bash
/usr/sbin/zabbix_agent2 \
  -c /etc/zabbix/zabbix_agent2.conf \
  -T
```

Iniciar y habilitar el servicio:

```bash
sudo systemctl enable --now zabbix-agent2
sudo systemctl status zabbix-agent2 --no-pager
```

Resultado esperado:

```text
active (running)
```

# 15. Crear el host Oracle Linux

```text
Nombre del equipo: <HOSTNAME_LINUX>
Grupo: Linux servers
Plantilla: Linux by Zabbix agent active
Estado: habilitado
```

La plantilla activa no requiere inicialmente una interfaz.

---

# 16. Validar comprobaciones activas

## Paso 1. Probar conectividad hacia Zabbix Server

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/11051' \
  && echo "CONEXION OK" \
  || echo "SIN CONEXION"
```

Si aparece `SIN CONEXION`, no continuar con Zabbix hasta encontrar la dirección correcta.

## Paso 2. Probar direcciones posibles

Cuando existen varias interfaces en Windows o Docker, probar cada dirección:

```bash
for IP in <IP_1> <IP_2>; do
  echo "Probando $IP:11051"
  timeout 5 bash -c "cat < /dev/null > /dev/tcp/$IP/11051" \
    && echo "CONEXION OK" \
    || echo "SIN CONEXION"
done
```

## Hallazgo del laboratorio

La dirección configurada inicialmente no era accesible desde Oracle Linux. Otra dirección del mismo equipo sí aceptaba conexiones al puerto `11051`.

La corrección fue:

```ini
ServerActive=<IP_QUE_RESPONDE>:11051
Hostname=<HOSTNAME_LINUX>
```

Después:

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

## Paso 3. Revisar errores recientes

```bash
sudo journalctl -u zabbix-agent2 \
  --since "10 minutes ago" \
  --no-pager |
grep -Ei 'active check configuration|cannot connect|host .*not found|no active checks|failed to send|no route'
```

Sin salida significa que no se encontraron esos errores durante el periodo revisado.

## Paso 4. Validar en Zabbix

Ruta:

```text
Monitoreo → Últimos datos
```

Buscar:

```text
Zabbix agent ping
```

Resultado esperado:

```text
Up (1)
```

La comprobación debe tener una antigüedad reciente, por ejemplo segundos o pocos minutos.

## Resultado alcanzado

```text
Zabbix agent ping = Up (1)
```

Esto confirma que las comprobaciones activas de Linux funcionan.

---

# Parte E. Configurar comprobaciones pasivas para Oracle

# 17. Vincular plantillas

Las dos plantillas deben vincularse directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

No vincular la plantilla Oracle dentro de la plantilla Linux.

No modificar directamente las plantillas oficiales.

# 18. Agregar interfaz pasiva

En el host `<HOSTNAME_LINUX>`, agregar una interfaz tipo **Agente**:

```text
IP: <IP_ORACLE_LINUX>
Conectar mediante: IP
Puerto: 10050
Predeterminada: Sí
```

Verificar que Agent 2 escucha:

```bash
ss -lntp | grep ':10050'
```

Resultado esperado:

```text
LISTEN ... *:10050 ... zabbix_agent2
```

# 19. Identificar el origen real de la consulta pasiva

Desde el equipo donde se ejecuta Zabbix:

```powershell
Test-NetConnection <IP_ORACLE_LINUX> -Port 10050 -InformationLevel Detailed
```

En Oracle Linux:

```bash
sudo timeout 60 tcpdump -nni <INTERFAZ_RED> tcp port 10050 -c 3
```

La dirección observada debe incluirse en:

```ini
Server=<IP_ORIGEN_REAL>
```

## Hallazgo del laboratorio

La consulta pasiva llegaba desde una dirección distinta a la esperada. Por eso el agente rechazaba la conexión aunque el puerto estuviera abierto.

# 20. Autorizar el firewall permanentemente

Identificar la zona:

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
```

Autorizar únicamente el origen real:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="<IP_ORIGEN_COMPROBACION_PASIVA>/32" port port="10050" protocol="tcp" accept'

sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-rich-rules
```

Validaciones necesarias:

- La regla aparece después del `reload`.
- La interfaz del agente aparece como **Disponible**.
- `oracle.ping` continúa en `Up (1)`.

No eliminar otras reglas hasta confirmar si corresponden a otro Zabbix Server, proxy u origen autorizado.

---

# Parte F. Configurar Oracle Database

# 21. Identificar el servicio Oracle

En SQL*Plus:

```sql
SHOW PARAMETER service_names;
SELECT CDB FROM V$DATABASE;
SELECT SYS_CONTEXT('USERENV', 'CON_NAME') FROM DUAL;
```

Resultado del laboratorio:

- Base no-CDB.
- Usuario de monitoreo local.
- La macro `{$ORACLE.SERVICE}` utiliza el `SERVICE_NAME`, no el SID.

# 22. Crear el usuario de monitoreo

Ejemplo base:

```sql
CREATE USER ZABBIX_MON IDENTIFIED BY "<CONTRASENA_SEGURA>";
GRANT CREATE SESSION TO ZABBIX_MON;
```

Durante el laboratorio se otorgaron permisos explícitos sobre vistas requeridas por la plantilla.

```sql
GRANT SELECT ON SYS.V_$INSTANCE TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$DATABASE TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SYSMETRIC TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SYSTEM_PARAMETER TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SESSION TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$RECOVERY_FILE_DEST TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$OSSTAT TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$PROCESS TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$DATAFILE TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$PGASTAT TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SGASTAT TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$LOG TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$ARCHIVE_DEST TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$ASM_DISKGROUP TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$ASM_DISKGROUP_STAT TO ZABBIX_MON;
GRANT SELECT ON SYS.DBA_USERS TO ZABBIX_MON;
```

Esta lista debe auditarse antes de producción contra la versión exacta de la plantilla utilizada.

## Restricción de licenciamiento

El ambiente no cuenta con Oracle Diagnostics Pack. Se revocaron:

```sql
REVOKE SELECT_CATALOG_ROLE FROM ZABBIX_MON;
REVOKE SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY FROM ZABBIX_MON;
```

Antes de producción se debe clonar la plantilla Oracle y deshabilitar los elementos que consulten ASH u otras funciones no licenciadas.

No modificar directamente la plantilla oficial.

# 23. Configurar macros Oracle

Ruta:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Macros
```

Configurar:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = <ORACLE_SERVICE>
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <CONTRASENA_SEGURA>
```

La contraseña debe ser de tipo **Texto secreto**.

# 24. Validar credenciales directamente

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/<ORACLE_SERVICE>
```

Una conexión exitosa confirma:

- Listener disponible.
- Servicio correcto.
- Usuario válido.
- Contraseña válida.
- Cuenta habilitada.

# 25. Resolver `DPI-1047`

Síntoma:

```text
DPI-1047: Cannot locate a 64-bit Oracle Client library:
"libclntsh.so: cannot open shared object file: No such file or directory"
```

La biblioteca existía en `ORACLE_HOME`, pero el servicio `zabbix-agent2` no recibía las variables de entorno Oracle.

Verificar:

```bash
find <ORACLE_HOME> -name 'libclntsh.so*' -type f -o -type l
ls -l <ORACLE_HOME>/lib/libclntsh.so*
```

## 25.1. Crear la carpeta del archivo de configuración

```bash
sudo mkdir -p /etc/systemd/system/zabbix-agent2.service.d
```

## 25.2. Ruta exacta del archivo

```text
/etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

## 25.3. Abrir el archivo

```bash
sudo vi /etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

El archivo puede no existir todavía. `vi` lo creará al guardar.

Presionar `i` y escribir:

```ini
[Service]
Environment="ORACLE_HOME=<ORACLE_HOME>"
Environment="LD_LIBRARY_PATH=<ORACLE_HOME>/lib"
```

Después:

1. Presionar `Esc`.
2. Escribir `:wq`.
3. Presionar `Enter`.

## 25.4. Comprobar el contenido

```bash
sudo cat /etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

## 25.5. Aplicar el cambio

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
```

Validar:

```bash
sudo systemctl show zabbix-agent2 -p Environment --value | tr ' ' '\n'
```

Resultado esperado:

```text
ORACLE_HOME=<ORACLE_HOME>
LD_LIBRARY_PATH=<ORACLE_HOME>/lib
```

Este cambio aplica solo al servicio Agent 2.

# 26. Validar Oracle Ping

Ruta:

```text
Monitoreo → Últimos datos
```

Buscar:

```text
Oracle by Zabbix agent 2: Ping
```

Resultado esperado:

```text
Up (1)
```

Esto confirma:

1. Comunicación pasiva.
2. Carga de bibliotecas Oracle Client.
3. Conectividad con el listener.
4. Servicio Oracle válido.
5. Autenticación del usuario.
6. Macros correctas.

---

# Parte G. Cómo leer Zabbix

# 27. Diferencia entre `Ping` y `Zabbix agent ping`

```text
Ping
```

Corresponde a la integración Oracle.

```text
Zabbix agent ping
```

Corresponde a la disponibilidad del agente Linux mediante comprobaciones activas.

Ambas deben aparecer como:

```text
Up (1)
```

# 28. Cómo interpretar una gráfica

El eje horizontal muestra tiempo.

Ejemplo:

```text
23:35  23:40  23:45  00:00  00:05
```

Cada marca representa un momento del periodo mostrado.

Las etiquetas inferiores significan:

```text
último  mínimo  media  máximo
```

- `último`: valor más reciente.
- `mínimo`: menor valor del periodo.
- `media`: promedio del periodo.
- `máximo`: mayor valor del periodo.

# 29. Corregir la zona horaria

Cuando las horas aparecen adelantadas o atrasadas, revisar la zona horaria del perfil.

Ruta:

```text
Configuración de usuario → Perfil → Zona horaria
```

Para Ciudad de México:

```text
America/Mexico_City
```

La zona horaria del usuario tiene prioridad sobre la global.

Después de guardar, recargar la gráfica.

---

# Parte H. Primeras métricas interpretadas

# 30. Oracle PGA

PGA significa **Program Global Area**.

Es memoria privada utilizada por procesos Oracle para:

- Ordenamientos.
- `GROUP BY`.
- `HASH JOIN`.
- Cursores.
- Sesiones.
- Operaciones PL/SQL.

Métricas comunes:

| Métrica | Significado |
|---|---|
| Total in use | Memoria utilizada actualmente |
| Total allocated | Memoria total asignada |
| Total freeable | Memoria que podría liberarse |
| Aggregate target | Objetivo configurado de PGA |
| Global memory bound | Límite aproximado de un área de trabajo |

Interpretación inicial:

```text
PGA utilizada / PGA_AGGREGATE_TARGET × 100
```

No ajustar Oracle ni el trigger usando una sola lectura. Revisar tendencia, duración y memoria del sistema operativo.

# 31. Tiempo promedio de espera del disco

Ejemplo:

```text
sda: Disk average waiting time
```

Mide cuánto tarda una operación de entrada/salida desde que Linux la solicita hasta que termina.

Métricas comunes:

| Métrica | Significado |
|---|---|
| `r_await` | Espera promedio de lecturas |
| `w_await` | Espera promedio de escrituras |

Unidad:

```text
milisegundos
```

No mide espacio libre. Mide latencia del almacenamiento.

Analizar junto con:

```text
Disk utilization
Disk average queue size
Disk read rate
Disk write rate
```

Un pico aislado no siempre representa un problema. Un valor alto sostenido requiere revisar carga, colas, respaldos, consultas y almacenamiento.

# 32. REDO logs disponibles

Zabbix reportó:

```text
Redo logs available to switch = 0
```

Consulta utilizada:

```sql
SELECT GROUP#,
       THREAD#,
       SEQUENCE#,
       ROUND(BYTES/1024/1024) AS MB,
       MEMBERS,
       ARCHIVED,
       STATUS
FROM V$LOG
ORDER BY THREAD#, GROUP#;
```

Resultado observado:

```text
1 grupo CURRENT
2 grupos ACTIVE
0 grupos INACTIVE o UNUSED
```

La base tiene tres grupos de 200 MB y utiliza:

```text
NOARCHIVELOG
```

Conclusión provisional:

- El umbral predeterminado debe revisarse.
- No debe modificarse Oracle únicamente para cerrar la alerta.
- Deben revisarse frecuencia de log switches y checkpoints.

Esta revisión continúa pendiente.

---

# Parte I. Cómo parametrizar métricas

# 33. Ajustar umbrales mediante macros

Las plantillas utilizan macros para evitar modificar cada trigger.

Procedimiento general:

1. Abrir **Recopilación de datos → Equipos**.
2. Seleccionar el host.
3. Abrir **Macros**.
4. Localizar la macro heredada.
5. Crear una sobrescritura a nivel host.
6. Documentar el motivo del cambio.
7. Esperar una nueva recopilación.
8. Validar que el trigger se comporte correctamente.

No cambiar un umbral sin conocer:

- Unidad.
- Frecuencia de lectura.
- Periodo evaluado por el trigger.
- Comportamiento normal del servidor.
- Consecuencia operativa.

## Ficha mínima para cada métrica

```text
Nombre:
Origen:
Unidad:
Frecuencia:
Valor normal:
Umbral de advertencia:
Umbral crítico:
Duración requerida:
Acción operativa:
Responsable:
```

# 34. Métricas personalizadas

Pendiente de validar en el laboratorio:

- `UserParameter`.
- Scripts externos.
- Consultas SQL controladas.
- Elementos calculados.
- Elementos dependientes.
- Descubrimiento de bajo nivel.
- Procesos y servicios propios.
- Métricas específicas de aplicaciones.

El laboratorio no se considerará concluido hasta crear al menos una métrica personalizada y documentar:

```text
qué mide
cómo se obtiene
qué valor es normal
qué condición genera alerta
cómo se valida
```

---

# Parte J. Alertas y avisos

# 35. Notificaciones pendientes

La siguiente etapa debe configurar y comprobar:

- Medio de notificación.
- Destinatarios.
- Grupos de usuarios.
- Severidades.
- Ventanas horarias.
- Reintentos.
- Escalamiento.
- Mensaje de problema.
- Mensaje de recuperación.
- Evidencia de entrega.

Prueba mínima requerida:

1. Generar un problema controlado.
2. Recibir el aviso.
3. Resolver el problema.
4. Recibir el aviso de recuperación.

---

# Parte K. Monitoreo de aplicaciones

# 36. Alcance pendiente

Seleccionar una aplicación representativa y validar:

- Disponibilidad HTTP/HTTPS.
- Código de respuesta.
- Tiempo de respuesta.
- Puerto del servicio.
- Proceso Java.
- Servicio Tomcat.
- Endpoint o API crítica.
- Contenedor Docker, cuando aplique.

---

# 37. Matriz rápida de diagnóstico

| Síntoma | Causa probable | Validación |
|---|---|---|
| Agent 2 no inicia | Puerto ocupado o configuración inválida | `zabbix_agent2 -T` y `systemctl status` |
| `Zabbix agent ping` sin datos | `ServerActive` incorrecto o sin ruta | Probar puerto `11051` desde el servidor monitoreado |
| Interfaz pasiva en timeout | Firewall o ruta | `Test-NetConnection`, `tcpdump`, `ss` |
| Interfaz pasiva rechazada | IP no incluida en `Server=` | Revisar origen real y configuración del agente |
| `Oracle Ping = Down (0)` | Credenciales, servicio o biblioteca Oracle | Probar SQL*Plus y revisar log de Agent 2 |
| `DPI-1047` | Agent 2 no encuentra `libclntsh.so` | Configurar `ORACLE_HOME` y `LD_LIBRARY_PATH` en systemd |
| Gráfica con horario incorrecto | Zona horaria del perfil | Cambiar a `America/Mexico_City` |
| Alerta no coherente | Umbral genérico de plantilla | Revisar macro, unidad, periodo y arquitectura real |

---

# 38. Estado actual

| Componente | Estado |
|---|---|
| Docker Desktop y WSL 2 | Correcto para laboratorio |
| Zabbix Server 7.4 en Docker | Operativo |
| Interfaz web | Operativa |
| MySQL de Zabbix | Operativo |
| Monitoreo de Windows | Funcional en laboratorio |
| Zabbix Agent 2 en Oracle Linux | Instalado y activo |
| Interfaz pasiva `10050` | Disponible y persistida en firewall |
| Comprobaciones activas Linux | `Zabbix agent ping = Up (1)` |
| Plantilla Oracle | Vinculada directamente al host |
| Usuario Oracle `ZABBIX_MON` | Creado; permisos pendientes de auditoría final |
| Macros Oracle | Configuradas y validadas |
| Oracle Client en Agent 2 | Configurado mediante override de `systemd` |
| `oracle.ping` | `Up (1)` |
| Métricas Oracle | En recopilación y revisión |
| Interpretación de métricas | Iniciada |
| Métricas personalizadas | No iniciado |
| Notificaciones | No iniciado |
| Aplicaciones | No iniciado |

---

# 39. Siguiente fase

- [x] Confirmar comprobaciones activas de Linux.
- [x] Confirmar `Zabbix agent ping = Up (1)`.
- [x] Confirmar `oracle.ping = Up (1)`.
- [x] Corregir la zona horaria del perfil.
- [ ] Confirmar métricas recientes de CPU, memoria, discos y red.
- [ ] Auditar los permisos exactos de `ZABBIX_MON`.
- [ ] Clonar la plantilla Oracle para excluir consultas no permitidas.
- [ ] Revisar elementos Oracle no soportados.
- [ ] Crear una matriz de interpretación de métricas.
- [ ] Crear una métrica personalizada de prueba.
- [ ] Configurar una notificación de problema y recuperación.
- [ ] Seleccionar y monitorear una aplicación representativa.
- [ ] Ajustar alertas y umbrales al ambiente real.
- [ ] Documentar diferencias entre laboratorio y producción.
- [ ] Elaborar checklist final reproducible.

---

# 40. Criterios para cerrar el laboratorio

## Sistema operativo

- Disponibilidad actualizada.
- CPU, memoria, discos y red con valores recientes.
- Procesos o servicios relevantes monitoreados.
- Comprobaciones activas sin alerta de ausencia de datos.

## Oracle Database

- `oracle.ping = Up (1)`.
- Métricas básicas recibidas.
- Permisos mínimos auditados.
- Plantilla compatible con el licenciamiento disponible.
- Umbrales principales interpretados y ajustados.

## Aplicaciones

- Disponibilidad de una aplicación de prueba.
- Tiempo de respuesta.
- Puerto, proceso o servicio validado.
- Alerta funcional por indisponibilidad.

## Operación

- Una métrica personalizada funcional.
- Una notificación de problema recibida.
- Una notificación de recuperación recibida.
- Procedimiento y checklist suficientes para repetir la instalación.

---

# 41. Diferencias entre laboratorio y producción

El laboratorio utiliza Windows, Docker Desktop, puertos alternos y un servidor de prueba.

Una implementación productiva deberá evaluar:

- Servidor Linux dedicado para Zabbix.
- Base de datos dimensionada.
- Respaldo y recuperación.
- Alta disponibilidad.
- Zabbix Proxy por ubicación o segmento.
- TLS entre agentes, proxies y servidor.
- Gestión segura de secretos.
- Reglas de firewall definitivas.
- Retención de históricos y tendencias.
- Capacidad de crecimiento.
- Integración con correo, mensajería o mesa de servicio.

---

# Referencias oficiales

- [Manual actual de Zabbix](https://www.zabbix.com/documentation/current/en/manual)
- [Instalación mediante contenedores](https://www.zabbix.com/documentation/7.4/en/manual/installation/containers)
- [Comprobaciones activas y pasivas](https://www.zabbix.com/documentation/current/en/manual/concepts/agent)
- [Plantillas para Agent 2](https://www.zabbix.com/documentation/7.4/en/manual/config/templates_out_of_the_box/zabbix_agent2)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
