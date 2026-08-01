# Implementación de Zabbix con Docker, Windows y Oracle Linux

> Fecha de corte: **31 de julio de 2026**  
> Estado: **Zabbix funciona para Windows y para el sistema operativo Oracle Linux. El monitoreo de Oracle Database todavía está pendiente de completar.**

## Seguridad de este documento

Este repositorio es público. Por ese motivo se utilizan valores genéricos como:

- `<IP_ZABBIX_SERVER>`
- `<HOSTNAME_LINUX>`
- `<ORACLE_SERVICE>`
- `<CONTRASENA_SEGURA>`

No se deben publicar direcciones internas, nombres reales de servidores, usuarios de aplicación ni contraseñas.

---

## 1. Arquitectura implementada

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
        └── Oracle Database
```

Puertos utilizados en el laboratorio:

| Componente | Puerto publicado | Puerto interno/destino |
|---|---:|---:|
| Interfaz web Zabbix | `8080` | `8080` |
| HTTPS web Zabbix | `8443` | `8443` |
| Zabbix Server | `11051` | `10051` del contenedor |
| Agent 2 de Windows | `11050` | Local en Windows |
| Agent 2 de Oracle Linux | `10050` | Local en Oracle Linux |

Los puertos `10050` y `10051` estaban dentro de rangos reservados en el equipo Windows; por eso se publicaron puertos alternos.

---

# Parte A. Preparación de Windows y Docker Desktop

## 2. Verificar WSL

En PowerShell:

```powershell
wsl --version
```

El ambiente utilizado ya tenía WSL 2 instalado.

## 3. Instalar Docker Desktop

1. Descargar Docker Desktop para Windows desde el sitio oficial.
2. Ejecutar el instalador.
3. Seleccionar el backend de **WSL 2**.
4. Reiniciar Windows cuando sea solicitado.
5. Abrir Docker Desktop y esperar a que el motor Linux esté iniciado.

## 4. Incidencia: Docker no detectó virtualización

Aunque el Administrador de tareas mostraba la virtualización habilitada, Docker indicó que no podía iniciar.

Se comprobó el hipervisor:

```powershell
Get-ComputerInfo -Property HyperV*
```

Resultado relevante:

```text
HyperVisorPresent : True
```

Se habilitó el arranque automático del hipervisor desde PowerShell como administrador:

```powershell
bcdedit /set hypervisorlaunchtype auto
```

También se encontró que la característica de WSL estaba deshabilitada:

```powershell
Get-WindowsOptionalFeature -Online |
Where-Object { $_.FeatureName -in @(
    "VirtualMachinePlatform",
    "Microsoft-Windows-Subsystem-Linux"
)} |
Select-Object FeatureName, State
```

Se habilitó con:

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

Después de reiniciar Windows, Docker Desktop inició correctamente.

## 5. Validar Docker

```powershell
docker run hello-world
```

Resultado esperado:

```text
Hello from Docker!
```

---

# Parte B. Instalación de Zabbix 7.4 con Docker Compose

## 6. Clonar el repositorio oficial

```powershell
mkdir C:\docker
cd C:\docker
git clone https://github.com/zabbix/zabbix-docker.git
cd zabbix-docker
git checkout 7.4
```

Verificar Compose:

```powershell
docker compose version
```

Zabbix 7.4 requiere Docker Compose 2.24.0 o posterior.

## 7. Configurar puertos web

En `.env` se configuró:

```dotenv
ZABBIX_WEB_NGINX_HTTP_PORT=8080
ZABBIX_WEB_NGINX_HTTPS_PORT=8443
```

## 8. Incidencia: imágenes `Windows_NT-7.4-latest` inexistentes

El comando inicial intentó descargar imágenes como:

```text
zabbix/zabbix-server-mysql:Windows_NT-7.4-latest
```

Esto ocurrió porque PowerShell proporcionó `OS=Windows_NT` al archivo Compose.

Solución en cada nueva sesión de PowerShell:

```powershell
$env:OS="alpine"
docker compose config --images
```

Las imágenes correctas deben incluir:

```text
zabbix/zabbix-server-mysql:alpine-7.4-latest
zabbix/zabbix-web-nginx-mysql:alpine-7.4-latest
mysql:8.4-oracle
```

## 9. Incidencia: MySQL rechazó al usuario Zabbix

El contenedor `server-db-init` terminó con:

```text
Access denied for user 'zabbix'
```

Los archivos secretos dentro de `env_vars` tenían finales de línea Windows `CRLF`. El contenedor esperaba formato Linux `LF`.

Se convirtieron los archivos:

```powershell
$archivos = @(
    ".\env_vars\.MYSQL_USER",
    ".\env_vars\.MYSQL_PASSWORD",
    ".\env_vars\.MYSQL_ROOT_PASSWORD"
)

$utf8SinBom = New-Object System.Text.UTF8Encoding($false)

foreach ($archivo in $archivos) {
    $ruta = (Resolve-Path $archivo).Path
    $texto = [System.IO.File]::ReadAllText($ruta).TrimEnd([char[]]"`r`n")
    [System.IO.File]::WriteAllText($ruta, $texto + "`n", $utf8SinBom)
}
```

Como la instalación era nueva, se eliminó la base incompleta:

```powershell
$env:OS="alpine"
docker compose down
Remove-Item -Recurse -Force .\zbx_env\var\lib\mysql
```

## 10. Incidencia: puerto `10051` reservado por Windows

Se verificaron los rangos excluidos:

```powershell
netsh interface ipv4 show excludedportrange protocol=tcp
```

El puerto `10051` estaba dentro de un rango reservado. Se cambió en `.env`:

```dotenv
ZABBIX_SERVER_PORT=11051
```

Zabbix Server continúa escuchando en `10051` dentro del contenedor; Windows publica el puerto `11051`.

## 11. Iniciar Zabbix

```powershell
cd C:\docker\zabbix-docker
$env:OS="alpine"
docker compose up -d
```

Verificar:

```powershell
docker compose ps
```

Estado esperado:

- MySQL: `healthy`
- Zabbix Server: `Up`
- Zabbix Web: `healthy`

Acceso web:

```text
http://localhost:8080
```

Credenciales predeterminadas del laboratorio:

```text
Usuario: Admin
Contraseña: zabbix
```

Estas credenciales solo deben conservarse en un laboratorio no expuesto.

## 12. Comandos de operación

Iniciar:

```powershell
$env:OS="alpine"
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

Eliminar contenedores y redes, conservando los archivos persistentes:

```powershell
docker compose down
```

## 13. Incidencia: no existe el pipe `dockerDesktopLinuxEngine`

Mensaje observado:

```text
failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine
```

Causa: Docker Desktop estaba cerrado o el motor Linux todavía no iniciaba.

Solución: abrir Docker Desktop y esperar a que el motor esté en ejecución antes de ejecutar Compose.

---

# Parte C. Monitoreo del equipo Windows

## 14. Instalar Zabbix Agent 2

Se utilizó el instalador MSI oficial para Windows.

Configuración conceptual:

```ini
Hostname=<HOSTNAME_WINDOWS>
Server=127.0.0.1
ServerActive=127.0.0.1:11051
```

## 15. Incidencia: Agent 2 no inició en el puerto `10050`

Windows tenía reservado el puerto `10050`.

Se editó:

```text
C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf
```

La línea debía estar activa, sin `#`:

```ini
ListenPort=11050
```

Validación:

```powershell
& "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" `
  -c "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -T
```

Inicio del servicio:

```powershell
Start-Service "Zabbix Agent 2"
Get-Service "Zabbix Agent 2"
```

Resultado esperado: `Running`.

## 16. Dar de alta Windows en Zabbix

En **Recopilación de datos → Equipos → Crear equipo**:

```text
Nombre del equipo: <HOSTNAME_WINDOWS>
Grupo: Windows servers
Plantilla: Windows by Zabbix agent active
Interfaces: ninguna
Estado: habilitado
```

Para comprobaciones activas no se requiere interfaz, porque el agente inicia la comunicación hacia Zabbix Server.

Resultado obtenido: métricas de CPU, RAM, discos, red, servicios y disponibilidad activa.

---

# Parte D. Instalación de Agent 2 en Oracle Linux 8.10

## 17. Incidencia: `Segmentation fault` con agente estático

Se había instalado un binario manual bajo:

```text
/opt/zabbix/sbin/zabbix_agentd
```

El binario mostraba versión y validaba la configuración:

```bash
/opt/zabbix/sbin/zabbix_agentd -V
/opt/zabbix/sbin/zabbix_agentd -c /opt/zabbix/conf/zabbix_agentd.conf -T
```

Pero al iniciar terminaba con:

```text
Segmentation fault (core dumped)
```

Los casos antiguos ZBX-9206 y Zabbix Agent 4.4.2 no correspondían directamente a esta instalación 7.4. La solución adoptada fue retirar el binario manual de la ruta operativa e instalar **Zabbix Agent 2 desde el repositorio oficial**.

## 18. Instalar repositorio y Agent 2

Respaldar la configuración anterior:

```bash
cp -a /opt/zabbix/conf/zabbix_agentd.conf \
      /root/zabbix_agentd.conf.opt.backup
```

Instalar repositorio:

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
```

## 19. Configurar Agent 2 en Linux

Editar:

```bash
vi /etc/zabbix/zabbix_agent2.conf
```

Configuración base:

```ini
Server=<IP_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>:11051
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

- `ServerActive` habilita las comprobaciones activas.
- `Server` autoriza las comprobaciones pasivas desde Zabbix Server.
- `Hostname` es un identificador lógico y debe coincidir exactamente con el campo **Nombre del equipo** en Zabbix.
- No es obligatorio que `Hostname` sea igual al hostname real del sistema operativo.

Validar e iniciar:

```bash
/usr/sbin/zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf -T
systemctl enable --now zabbix-agent2
systemctl status zabbix-agent2 --no-pager
```

## 20. Registrar Oracle Linux en Zabbix

Crear el equipo:

```text
Nombre del equipo: <HOSTNAME_LINUX>
Grupo: Linux servers
Plantilla: Linux by Zabbix agent active
Interfaces: ninguna inicialmente
Estado: habilitado
```

## 21. Incidencia: comprobaciones activas sin datos

Mensajes observados:

```text
cannot connect to [<IP_ZABBIX_SERVER>:11051]
host [<HOSTNAME_LINUX>] not found
```

Validaciones realizadas:

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/11051' \
  && echo "CONEXION OK" \
  || echo "SIN CONEXION"
```

También se revisó:

```bash
tail -n 50 /var/log/zabbix/zabbix_agent2.log
```

Solución aplicada:

1. Confirmar que `Hostname` coincidiera exactamente con el nombre del host en Zabbix.
2. Vincular directamente al host la plantilla `Linux by Zabbix agent active`.
3. Reiniciar Agent 2.
4. Esperar la siguiente actualización de comprobaciones activas.

El registro posteriormente mostró que las comprobaciones activas volvieron a estar disponibles.

---

# Parte E. Preparación para monitorear Oracle Database

## 22. Vincular la plantilla correcta

La plantilla debe vincularse **directamente al host Oracle Linux**, no dentro de la plantilla oficial de Linux:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

Incidencia encontrada: al intentar agregar Oracle dentro de la plantilla Linux, Zabbix solicitó una interfaz. La corrección fue cancelar el cambio sobre la plantilla oficial y editar el host.

## 23. Agregar interfaz pasiva al host

La plantilla `Oracle by Zabbix agent 2` utiliza elementos pasivos. Por eso se agregó al host una interfaz tipo **Agente**:

```text
IP: <IP_ORACLE_LINUX>
Puerto: 10050
Conectar mediante: IP
```

Agent 2 puede trabajar simultáneamente con comprobaciones activas y pasivas.

## 24. Verificar que Agent 2 escucha

```bash
ss -lntp | grep ':10050'
```

Resultado esperado:

```text
LISTEN ... *:10050 ... zabbix_agent2
```

## 25. Incidencia: interfaz pasiva en timeout

El servicio escuchaba en `10050`, pero Zabbix no podía conectarse. `firewalld` estaba activo y la interfaz principal pertenecía a la zona `public`.

Se abrió el puerto únicamente desde Zabbix Server:

```bash
firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="<IP_ZABBIX_SERVER>/32" port protocol="tcp" port="10050" accept'

firewall-cmd --reload
firewall-cmd --zone=public --list-rich-rules
```

No se recomienda abrir `10050` para toda la red.

## 26. Datos de Oracle identificados

Se confirmó:

```sql
SHOW PARAMETER service_names;
SELECT CDB FROM V$DATABASE;
SELECT SYS_CONTEXT('USERENV', 'CON_NAME') FROM DUAL;
```

Resultado funcional:

- La base es **no-CDB** (`CDB = NO`).
- Existe un `SERVICE_NAME` publicado.
- La plantilla debe usar el `SERVICE_NAME`; no debe utilizar directamente el SID en `{$ORACLE.SERVICE}`.

## 27. Macros que deberán configurarse en el host

Ruta:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Macros
```

Valores pendientes:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = <ORACLE_SERVICE>
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <CONTRASENA_SEGURA>
```

La contraseña debe guardarse como **Texto secreto** y nunca debe publicarse en GitHub.

---

# Estado actual

| Componente | Estado |
|---|---|
| Docker Desktop y WSL 2 | Correcto |
| Zabbix Server 7.4 en Docker | Correcto |
| Interfaz web | Correcto |
| MySQL de Zabbix | Correcto |
| Monitoreo del equipo Windows | Correcto |
| Zabbix Agent 2 en Oracle Linux | Correcto |
| Métricas del sistema operativo Linux | Correctas o en validación final |
| Acceso pasivo al puerto `10050` | Regla de firewall aplicada; confirmar disponibilidad en verde |
| Plugin Oracle de Agent 2 | Cargado según el log del agente |
| Plantilla `Oracle by Zabbix agent 2` | Vinculada o en proceso de vinculación directa al host |
| Usuario Oracle `ZABBIX_MON` | **Pendiente de crear** |
| Macros de conexión Oracle | **Pendientes de completar** |
| Métricas Oracle Database | **No conectadas todavía** |

---

# Siguiente fase pendiente: conectar Oracle

No debe considerarse concluido el monitoreo de Oracle hasta completar y validar estos pasos:

- [ ] Confirmar que la interfaz pasiva del host aparece disponible en Zabbix.
- [ ] Crear el usuario Oracle exclusivo `ZABBIX_MON` en la base no-CDB.
- [ ] Otorgar únicamente los permisos requeridos por la plantilla oficial.
- [ ] Confirmar si existe licencia de **Oracle Diagnostics Pack** antes de consultar `V$ACTIVE_SESSION_HISTORY`.
- [ ] Configurar las cuatro macros Oracle en el host.
- [ ] Probar `oracle.ping`.
- [ ] Confirmar en **Monitoreo → Últimos datos** que lleguen métricas de instancia, sesiones, procesos, SGA/PGA, tablespaces, redo, archive y FRA.
- [ ] Ajustar umbrales antes de habilitar alertas productivas.

## Prueba prevista

Desde un equipo que tenga `zabbix_get` y acceso al Agent 2:

```bash
zabbix_get -s <IP_ORACLE_LINUX> -p 10050 \
  -k 'oracle.ping["tcp://127.0.0.1:1521","ZABBIX_MON","<CONTRASENA>","<ORACLE_SERVICE>"]'
```

No escribir la contraseña real en un historial de comandos de producción. Para la validación definitiva se deberá usar un método seguro o una sesión temporal controlada.

---

# Referencias oficiales

- [Manual actual de Zabbix](https://www.zabbix.com/documentation/current/en/manual)
- [Instalación de Zabbix desde contenedores](https://www.zabbix.com/documentation/7.4/en/manual/installation/containers)
- [Agente: comprobaciones activas y pasivas](https://www.zabbix.com/documentation/current/en/manual/concepts/agent)
- [Plantillas para Zabbix Agent 2](https://www.zabbix.com/documentation/7.4/en/manual/config/templates_out_of_the_box/zabbix_agent2)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
- [Repositorio oficial para Oracle Linux 8](https://repo.zabbix.com/zabbix/7.4/stable/oracle/8/)
