# Exploración de Zabbix con Docker, Oracle Linux y Oracle Database

> Fecha de corte: **6 de agosto de 2026**  
> Estado: **exploración técnica en curso**. La plataforma base y el monitoreo pasivo de Oracle Database funcionan; las comprobaciones activas de Linux, la interpretación de métricas, las métricas personalizadas, las notificaciones y el monitoreo de aplicaciones siguen pendientes de completar.

Las incidencias y soluciones detalladas se encuentran en la [base de conocimiento](base-conocimiento/README.md). Esta guía concentra el procedimiento reproducible, el estado actual y los criterios para concluir el demo.

---

## 1. Objetivo del demo

El propósito de esta prueba no es desplegar todavía una plataforma productiva. Se busca determinar si Zabbix puede utilizarse para monitorear de forma centralizada:

1. Sistemas operativos de servidores.
2. Bases de datos Oracle.
3. Aplicaciones, servicios, puertos, procesos y endpoints.

La exploración también debe determinar:

- Qué métricas se obtienen mediante plantillas estándar.
- Cómo interpretar las métricas recopiladas.
- Qué umbrales deben ajustarse para cada ambiente.
- Cómo crear métricas personalizadas.
- Cómo configurar avisos y escalamiento.
- Qué diferencias existirán entre el laboratorio y producción.

### Avance estimado de la exploración

| Frente | Estado | Avance estimado |
|---|---|---:|
| Zabbix Server con Docker | Operativo | 90% |
| Instalación de Zabbix Agent 2 | Operativo | 90% |
| Comunicación pasiva con Oracle Linux | Validada | 100% |
| Monitoreo básico de Oracle Database | En exploración | 55% |
| Monitoreo básico del sistema operativo | En corrección | 45% |
| Interpretación de métricas | Pendiente | 15% |
| Métricas personalizadas | No iniciado | 0% |
| Alertas y notificaciones | No iniciado | 0% |
| Monitoreo de aplicaciones | No iniciado | 0% |

**Avance general estimado del demo: 35%.**

---

## 2. Seguridad y tratamiento de información

Este repositorio es público. Utilizar valores genéricos:

- `<IP_ZABBIX_SERVER>`
- `<IP_ORIGEN_COMPROBACION_PASIVA>`
- `<IP_ORACLE_LINUX>`
- `<HOSTNAME_WINDOWS>`
- `<HOSTNAME_LINUX>`
- `<ORACLE_SERVICE>`
- `<CONTRASENA_SEGURA>`

No publicar:

- Contraseñas.
- Direcciones internas reales.
- Usuarios de aplicación.
- Nombres reales de servidores productivos.
- Cadenas de conexión productivas.

---

## 3. Arquitectura del laboratorio

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

### Flujo utilizado

```text
Comprobaciones activas Linux
Oracle Linux ────────────────> Zabbix Server:11051

Comprobaciones pasivas Oracle
Zabbix Server ───────────────> Agent 2:10050 ─────────> Oracle Database
```

---

# Parte A. Preparar Windows y Docker Desktop

## 4. Verificar WSL 2

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

Ambas características deben estar habilitadas.

## 5. Instalar Docker Desktop

1. Descargar Docker Desktop para Windows.
2. Ejecutar el instalador.
3. Utilizar el backend WSL 2.
4. Reiniciar Windows si se solicita.
5. Abrir Docker Desktop y esperar a que el motor Linux esté iniciado.

Validar:

```powershell
docker run hello-world
```

Consulta de errores: [Docker Desktop y Windows](base-conocimiento/docker-windows.md).

---

# Parte B. Instalar Zabbix 7.4 con Docker Compose

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

## 7. Configurar variables y puertos

Editar `.env`:

```dotenv
ZABBIX_WEB_NGINX_HTTP_PORT=8080
ZABBIX_WEB_NGINX_HTTPS_PORT=8443
ZABBIX_SERVER_PORT=11051
```

En cada nueva sesión de PowerShell:

```powershell
$env:OS="alpine"
```

Verificar las imágenes:

```powershell
docker compose config --images
```

Deben aparecer imágenes `alpine-7.4-latest` para los componentes Zabbix.

## 8. Iniciar Zabbix

```powershell
cd C:\docker\zabbix-docker
$env:OS="alpine"
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

Credenciales predeterminadas del laboratorio:

```text
Usuario: Admin
Contraseña: zabbix
```

No conservar estas credenciales en ambientes expuestos o productivos.

## 9. Operación básica

Iniciar o actualizar:

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

Eliminar contenedores y redes:

```powershell
docker compose down
```

Consulta de errores: [Zabbix en Docker Compose](base-conocimiento/zabbix-docker.md).

---

# Parte C. Monitorear Windows

## 10. Instalar Zabbix Agent 2

Instalar el MSI oficial de Zabbix Agent 2.

Configuración de laboratorio:

```ini
Hostname=<HOSTNAME_WINDOWS>
Server=127.0.0.1
ServerActive=127.0.0.1:11051
ListenPort=11050
```

La línea `ListenPort` debe estar activa, sin `#`.

Validar:

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

## 11. Crear el host Windows

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

# Parte D. Instalar Agent 2 en Oracle Linux 8.10

## 12. Instalar el repositorio oficial

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

Rutas:

```text
Ejecutable:    /usr/sbin/zabbix_agent2
Configuración: /etc/zabbix/zabbix_agent2.conf
Servicio:      zabbix-agent2
Log:           /var/log/zabbix/zabbix_agent2.log
```

## 13. Configurar Agent 2

Editar:

```bash
vi /etc/zabbix/zabbix_agent2.conf
```

Configuración base:

```ini
Server=<IP_ZABBIX_SERVER>,<IP_ORIGEN_COMPROBACION_PASIVA>
ServerActive=<IP_ZABBIX_SERVER>:11051
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

Consideraciones:

- `Hostname` debe coincidir exactamente con **Nombre del equipo** en Zabbix.
- `Server=` controla qué orígenes pueden consultar pasivamente al agente.
- `ServerActive=` indica a dónde enviará el agente sus comprobaciones activas.
- La IP de origen real de una comprobación pasiva puede diferir de la IP publicada del servidor Zabbix.

Validar e iniciar:

```bash
/usr/sbin/zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf -T
systemctl enable --now zabbix-agent2
systemctl status zabbix-agent2 --no-pager
```

## 14. Crear el host Oracle Linux

```text
Nombre del equipo: <HOSTNAME_LINUX>
Grupo: Linux servers
Plantilla: Linux by Zabbix agent active
Estado: habilitado
```

Para la plantilla activa no se requiere inicialmente una interfaz.

Verificar conectividad hacia Zabbix Server:

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/11051' \
  && echo "CONEXION OK" \
  || echo "SIN CONEXION"
```

Revisar logs:

```bash
tail -n 50 /var/log/zabbix/zabbix_agent2.log
journalctl -u zabbix-agent2 -n 50 --no-pager
```

### Estado actual de las comprobaciones activas

La comunicación pasiva funciona, pero la interfaz muestra la comprobación activa como desconocida y permanece una alerta similar a:

```text
Linux: Zabbix agent is not available (or no data for 30m)
```

Pendiente de diagnóstico:

```bash
sudo grep -Ei 'active check|cannot connect|host .*not found|no active checks|failed to send|connection refused' \
  /var/log/zabbix/zabbix_agent2.log | tail -n 50
```

No considerar concluido el monitoreo del sistema operativo hasta que las métricas activas tengan valores recientes.

Consulta de errores: [Agentes Zabbix](base-conocimiento/agentes-zabbix.md).

---

# Parte E. Preparar el monitoreo de Oracle Database

## 15. Vincular plantillas al host

Las dos plantillas deben vincularse directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

No vincular la plantilla Oracle dentro de la plantilla Linux y no modificar directamente las plantillas oficiales.

## 16. Agregar interfaz pasiva

En el host `<HOSTNAME_LINUX>`, agregar interfaz tipo **Agente**:

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

## 17. Identificar el origen real de las consultas pasivas

Desde el servidor donde se ejecuta Zabbix, validar:

```powershell
Test-NetConnection <IP_ORACLE_LINUX> -Port 10050 -InformationLevel Detailed
```

En Oracle Linux, cuando exista duda sobre el origen:

```bash
sudo timeout 60 tcpdump -nni <INTERFAZ_RED> tcp port 10050 -c 3
```

La dirección observada debe incluirse en `Server=` y en la regla de `firewalld`.

## 18. Autorizar el firewall de forma permanente

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

Validación alcanzada en el demo:

- La regla permanece después de `firewall-cmd --reload`.
- La interfaz pasiva aparece como **Disponible**.
- `oracle.ping` continúa en `Up (1)` después de recargar el firewall.

No retirar otras reglas existentes sin confirmar antes si corresponden a un Zabbix Server, proxy u otro origen autorizado.

## 19. Identificar el servicio Oracle

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

## 20. Crear el usuario de monitoreo

Ejemplo base:

```sql
CREATE USER ZABBIX_MON IDENTIFIED BY "<CONTRASENA_SEGURA>";
GRANT CREATE SESSION TO ZABBIX_MON;
```

Durante el demo se otorgaron permisos explícitos sobre vistas dinámicas y de diccionario necesarias para la plantilla. La lista debe auditarse antes de producción contra la versión exacta de la plantilla utilizada.

Permisos usados en la exploración:

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

### Restricción de licenciamiento

El ambiente no cuenta con Oracle Diagnostics Pack. Por esta razón se revocaron:

```sql
REVOKE SELECT_CATALOG_ROLE FROM ZABBIX_MON;
REVOKE SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY FROM ZABBIX_MON;
```

Antes de producción se debe clonar la plantilla Oracle y deshabilitar los elementos, dependencias, gráficas o triggers que consulten ASH u otras funcionalidades no licenciadas. No modificar directamente la plantilla oficial.

## 21. Configurar macros del host

Ruta:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Macros
```

Configuración:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = <ORACLE_SERVICE>
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <CONTRASENA_SEGURA>
```

La contraseña debe configurarse como **Texto secreto**.

## 22. Validar credenciales directamente

Desde Oracle Linux:

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/<ORACLE_SERVICE>
```

La conexión exitosa confirma:

- Listener disponible.
- Servicio correcto.
- Usuario válido.
- Contraseña válida.
- Cuenta habilitada.

## 23. Resolver `DPI-1047` en Agent 2

Síntoma:

```text
DPI-1047: Cannot locate a 64-bit Oracle Client library:
"libclntsh.so: cannot open shared object file: No such file or directory"
```

La biblioteca Oracle existía en el `ORACLE_HOME`, pero el servicio `zabbix-agent2` iniciado por `systemd` no heredaba el entorno del usuario `oracle`.

Verificar la biblioteca:

```bash
find <ORACLE_HOME> -name 'libclntsh.so*' -type f -o -type l
ls -l <ORACLE_HOME>/lib/libclntsh.so*
```

Crear el directorio del override:

```bash
sudo mkdir -p /etc/systemd/system/zabbix-agent2.service.d
```

Crear:

```text
/etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Contenido:

```ini
[Service]
Environment="ORACLE_HOME=<ORACLE_HOME>"
Environment="LD_LIBRARY_PATH=<ORACLE_HOME>/lib"
```

Aplicar:

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
```

Validar el entorno efectivo:

```bash
sudo systemctl show zabbix-agent2 -p Environment --value | tr ' ' '\n'
```

Resultado esperado:

```text
ORACLE_HOME=<ORACLE_HOME>
LD_LIBRARY_PATH=<ORACLE_HOME>/lib
```

Este cambio afecta únicamente al servicio Agent 2 y no modifica globalmente el entorno del sistema.

## 24. Validar `oracle.ping`

Ruta:

```text
Monitoreo → Últimos datos
```

Buscar:

```text
Oracle by Zabbix agent 2: Ping
```

Resultado alcanzado:

```text
Up (1)
```

Esto confirma funcionalmente:

1. Comunicación pasiva entre Zabbix Server y Agent 2.
2. Carga de las bibliotecas Oracle Client.
3. Conectividad con el listener.
4. Servicio Oracle válido.
5. Autenticación de `ZABBIX_MON`.
6. Expansión correcta de las macros.

Consulta específica: [Monitoreo de Oracle](base-conocimiento/oracle.md).

---

# Parte F. Hallazgos de la exploración

## 25. Alerta de REDO logs disponibles

Zabbix reportó:

```text
Redo logs available to switch = 0
```

Consulta realizada:

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

La base utiliza tres grupos de 200 MB y está en modo:

```text
NOARCHIVELOG
```

Conclusión provisional:

- El umbral predeterminado de menos de tres grupos disponibles no es compatible con una configuración total de tres grupos, porque uno siempre estará `CURRENT`.
- El valor cero también requiere analizar la frecuencia de log switches y el comportamiento de checkpoints.
- No se debe agregar, eliminar o redimensionar REDO ni modificar el umbral sin completar primero el análisis.

Esta revisión quedó temporalmente aplazada.

## 26. Interpretación de métricas

La exploración confirmó que Zabbix recopila valores y genera alertas, pero todavía falta documentar:

- Qué significa cada métrica.
- Cuál es su rango normal.
- Qué condiciones requieren advertencia o incidente.
- Qué valores predeterminados no aplican al ambiente.
- Qué métricas son operativas y cuáles son únicamente informativas.

Cada métrica aceptada debe documentarse con:

```text
Nombre
Origen
Unidad
Frecuencia
Valor normal
Umbral de advertencia
Umbral crítico
Acción operativa
Responsable
```

## 27. Métricas personalizadas

Pendiente explorar:

- `UserParameter` en Agent 2.
- Scripts externos.
- Consultas SQL controladas.
- Elementos calculados y dependientes.
- Descubrimiento de bajo nivel.
- Procesos y servicios propios.
- Métricas específicas de aplicaciones.

No considerar concluido el demo hasta crear al menos una métrica personalizada representativa.

## 28. Avisos y escalamiento

Pendiente configurar y validar:

- Medios de notificación.
- Destinatarios y grupos.
- Severidades notificables.
- Ventanas horarias.
- Reintentos.
- Escalamiento.
- Mensajes de recuperación.
- Evidencia de entrega.

El demo debe incluir al menos una alerta controlada y su notificación de problema y recuperación.

## 29. Monitoreo de aplicaciones

Pendiente seleccionar una aplicación representativa y validar:

- Disponibilidad HTTP/HTTPS.
- Código de respuesta.
- Tiempo de respuesta.
- Puerto de servicio.
- Proceso Java.
- Servicio Tomcat.
- Endpoint o API crítica.
- Contenedor Docker, cuando aplique.

---

# Estado actual

| Componente | Estado |
|---|---|
| Docker Desktop y WSL 2 | Correcto para laboratorio |
| Zabbix Server 7.4 en Docker | Operativo |
| Interfaz web | Operativa |
| MySQL de Zabbix | Operativo |
| Monitoreo de Windows | Funcional en laboratorio |
| Zabbix Agent 2 en Oracle Linux | Instalado y activo |
| Interfaz pasiva `10050` | Disponible y persistida en firewall |
| Comprobaciones activas Linux | Pendientes de corregir |
| Plantilla Oracle | Vinculada directamente al host |
| Usuario Oracle `ZABBIX_MON` | Creado; permisos pendientes de auditoría final |
| Macros Oracle | Configuradas y validadas |
| Oracle Client en Agent 2 | Configurado mediante override de `systemd` |
| `oracle.ping` | `Up (1)` |
| Métricas Oracle | En recopilación y revisión |
| Interpretación de métricas | Pendiente |
| Métricas personalizadas | No iniciado |
| Notificaciones | No iniciado |
| Aplicaciones | No iniciado |

---

# Siguiente fase

- [ ] Corregir las comprobaciones activas de Linux.
- [ ] Confirmar métricas recientes de CPU, memoria, discos y red.
- [ ] Auditar los permisos exactos de `ZABBIX_MON` contra la plantilla utilizada.
- [ ] Clonar la plantilla Oracle y excluir consultas no permitidas por licenciamiento.
- [ ] Revisar elementos Oracle no soportados.
- [ ] Definir una matriz de interpretación de métricas.
- [ ] Crear una métrica personalizada de prueba.
- [ ] Configurar una notificación de problema y recuperación.
- [ ] Seleccionar y monitorear una aplicación representativa.
- [ ] Ajustar alertas y umbrales al ambiente real.
- [ ] Documentar diferencias entre laboratorio y producción.
- [ ] Elaborar checklist final reproducible.

---

# Criterios para cerrar el demo

El demo se considerará técnicamente concluido cuando se confirme:

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
- Validación de puerto, proceso o servicio.
- Alerta funcional por indisponibilidad.

## Operación

- Una métrica personalizada funcional.
- Una notificación de problema recibida.
- Una notificación de recuperación recibida.
- Procedimiento y checklist suficientes para repetir la instalación.

---

# Diferencias esperadas entre laboratorio y producción

El laboratorio utiliza Windows, Docker Desktop, puertos alternos y un servidor de prueba. Una implementación productiva deberá evaluar:

- Servidor Linux dedicado para Zabbix.
- Base de datos soportada y dimensionada.
- Respaldo y recuperación.
- Alta disponibilidad.
- Zabbix Proxy por ubicación o segmento.
- TLS entre agentes, proxies y servidor.
- Gestión segura de secretos.
- Segmentación y reglas de firewall definitivas.
- Retención de históricos y tendencias.
- Capacidad para crecimiento.
- Integración con correo, mensajería o mesa de servicio.

---

# Referencias oficiales

- [Manual actual de Zabbix](https://www.zabbix.com/documentation/current/en/manual)
- [Instalación mediante contenedores](https://www.zabbix.com/documentation/7.4/en/manual/installation/containers)
- [Comprobaciones activas y pasivas](https://www.zabbix.com/documentation/current/en/manual/concepts/agent)
- [Plantillas para Agent 2](https://www.zabbix.com/documentation/7.4/en/manual/config/templates_out_of_the_box/zabbix_agent2)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
