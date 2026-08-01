# Implementación de Zabbix con Docker, Windows y Oracle Linux

> Fecha de corte: **31 de julio de 2026**  
> Estado: **Zabbix funciona para Windows y para el sistema operativo Oracle Linux. El monitoreo de Oracle Database todavía está pendiente de completar.**

Las incidencias y soluciones se encuentran en la [base de conocimiento](base-conocimiento/README.md). Esta guía contiene únicamente el procedimiento de instalación y configuración.

## Seguridad

Este repositorio es público. Utilizar valores genéricos:

- `<IP_ZABBIX_SERVER>`
- `<IP_ORACLE_LINUX>`
- `<HOSTNAME_WINDOWS>`
- `<HOSTNAME_LINUX>`
- `<ORACLE_SERVICE>`
- `<CONTRASENA_SEGURA>`

No publicar contraseñas, direcciones internas, usuarios de aplicación ni nombres reales de servidores productivos.

---

## 1. Arquitectura del laboratorio

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

| Componente | Puerto publicado | Destino |
|---|---:|---:|
| Interfaz web Zabbix | `8080` | `8080` |
| HTTPS web Zabbix | `8443` | `8443` |
| Zabbix Server | `11051` | `10051` del contenedor |
| Agent 2 de Windows | `11050` | Windows local |
| Agent 2 de Oracle Linux | `10050` | Oracle Linux |

---

# Parte A. Preparar Windows y Docker Desktop

## 2. Verificar WSL 2

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

Ambas deben estar habilitadas.

## 3. Instalar Docker Desktop

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

## 4. Clonar el repositorio oficial

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

## 5. Configurar variables y puertos

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

Deben aparecer imágenes `alpine-7.4-latest` para Zabbix.

## 6. Iniciar Zabbix

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

## 7. Operación básica

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

## 8. Instalar Zabbix Agent 2

Instalar el MSI oficial de Zabbix Agent 2.

Configuración:

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

## 9. Crear el host Windows

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

Métricas esperadas: CPU, memoria, discos, red, servicios y disponibilidad activa.

Consulta de errores: [Agentes Zabbix](base-conocimiento/agentes-zabbix.md).

---

# Parte D. Instalar Agent 2 en Oracle Linux 8.10

## 10. Instalar el repositorio oficial

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
```

## 11. Configurar Agent 2

Editar:

```bash
vi /etc/zabbix/zabbix_agent2.conf
```

Configurar:

```ini
Server=<IP_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>:11051
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

El `Hostname` es el identificador lógico usado por las comprobaciones activas y debe coincidir exactamente con el campo **Nombre del equipo** en Zabbix.

Validar e iniciar:

```bash
/usr/sbin/zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf -T
systemctl enable --now zabbix-agent2
systemctl status zabbix-agent2 --no-pager
```

## 12. Crear el host Oracle Linux

```text
Nombre del equipo: <HOSTNAME_LINUX>
Grupo: Linux servers
Plantilla: Linux by Zabbix agent active
Interfaces: ninguna inicialmente
Estado: habilitado
```

Verificar conectividad activa:

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

Consulta de errores: [Agentes Zabbix](base-conocimiento/agentes-zabbix.md).

---

# Parte E. Preparar el monitoreo de Oracle Database

## 13. Vincular plantillas al host

Las dos plantillas deben vincularse directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

No modificar la plantilla oficial de Linux para agregarle Oracle.

## 14. Agregar interfaz pasiva

En el host `<HOSTNAME_LINUX>`, agregar interfaz tipo **Agente**:

```text
IP: <IP_ORACLE_LINUX>
Conectar mediante: IP
Puerto: 10050
```

Verificar que Agent 2 escucha:

```bash
ss -lntp | grep ':10050'
```

## 15. Autorizar el firewall

Identificar la zona:

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
```

Autorizar únicamente a Zabbix Server:

```bash
firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="<IP_ZABBIX_SERVER>/32" port protocol="tcp" port="10050" accept'

firewall-cmd --reload
firewall-cmd --zone=public --list-rich-rules
```

## 16. Identificar el servicio Oracle

En SQL*Plus:

```sql
SHOW PARAMETER service_names;
SELECT CDB FROM V$DATABASE;
SELECT SYS_CONTEXT('USERENV', 'CON_NAME') FROM DUAL;
```

Resultado obtenido en la prueba:

- Base no-CDB.
- Se utilizará un usuario local `ZABBIX_MON`.
- `{$ORACLE.SERVICE}` deberá contener el `SERVICE_NAME`, no asumir el SID.

## 17. Configurar macros del host

Ruta:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Macros
```

Macros pendientes:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = <ORACLE_SERVICE>
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <CONTRASENA_SEGURA>
```

La contraseña debe ser de tipo **Texto secreto**.

Consulta específica: [Monitoreo de Oracle](base-conocimiento/oracle.md).

---

# Estado actual

| Componente | Estado |
|---|---|
| Docker Desktop y WSL 2 | Correcto |
| Zabbix Server 7.4 en Docker | Correcto |
| Interfaz web | Correcto |
| MySQL de Zabbix | Correcto |
| Monitoreo de Windows | Correcto |
| Zabbix Agent 2 en Oracle Linux | Correcto |
| Métricas de Oracle Linux | Incorporadas o en validación final |
| Acceso pasivo `10050` | Configurado; confirmar disponibilidad |
| Plantilla Oracle | Vinculada o en proceso de validación |
| Usuario Oracle `ZABBIX_MON` | Pendiente |
| Macros Oracle | Pendientes |
| Métricas de Oracle Database | No conectadas todavía |

## Siguiente fase

- [ ] Confirmar disponibilidad pasiva del agente.
- [ ] Crear `ZABBIX_MON` con permisos mínimos.
- [ ] Confirmar la licencia de Oracle Diagnostics Pack antes de consultar `V$ACTIVE_SESSION_HISTORY`.
- [ ] Configurar las macros Oracle.
- [ ] Validar `oracle.ping`.
- [ ] Confirmar las métricas Oracle en **Monitoreo → Últimos datos**.

---

# Referencias oficiales

- [Manual actual de Zabbix](https://www.zabbix.com/documentation/current/en/manual)
- [Instalación mediante contenedores](https://www.zabbix.com/documentation/7.4/en/manual/installation/containers)
- [Comprobaciones activas y pasivas](https://www.zabbix.com/documentation/current/en/manual/concepts/agent)
- [Plantillas para Agent 2](https://www.zabbix.com/documentation/7.4/en/manual/config/templates_out_of_the_box/zabbix_agent2)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
