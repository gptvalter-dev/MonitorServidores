# Incidencias: agentes Zabbix

## 1. Zabbix Agent 2 de Windows no inicia

**Síntoma**

```text
cannot start server listener
Listen failed: listen tcp 0.0.0.0:10050
```

**Causa identificada**

El puerto `10050` estaba reservado por Windows y la línea modificada permanecía comentada.

**Solución**

Configurar en `C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf`:

```ini
ListenPort=11050
```

Validar e iniciar:

```powershell
& "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" `
  -c "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -T
Start-Service "Zabbix Agent 2"
Get-Service "Zabbix Agent 2"
```

**Estado:** resuelta.

---

## 2. Agente clásico estático termina con `Segmentation fault`

**Ambiente**

- Oracle Linux 8.10.
- Binario manual en `/opt/zabbix/sbin/zabbix_agentd`.
- Zabbix Agent 7.4.x enlazado estáticamente.

**Síntoma**

El binario mostraba la versión y validaba la configuración, pero al iniciar terminaba con:

```text
Segmentation fault (core dumped)
```

**Solución aplicada**

Se sustituyó el binario manual por Zabbix Agent 2 desde el repositorio oficial:

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/zabbix-release-latest-7.4.el8.noarch.rpm
dnf clean all
dnf makecache
dnf install -y zabbix-agent2
```

**Estado:** resuelta.

---

## 3. Comprobaciones activas sin datos

**Causas identificadas**

- `Hostname` no coincidía exactamente con el nombre técnico del equipo en Zabbix.
- La plantilla `Linux by Zabbix agent active` no estaba vinculada directamente al host.
- El agente aún no había actualizado su configuración activa.

**Configuración validada**

```ini
ServerActive=192.20.0.10:11051
Hostname=ZAM-SV-073-19C
```

En Zabbix:

```text
Nombre del equipo: ZAM-SV-073-19C
Plantilla: Linux by Zabbix agent active
```

**Estado:** resuelta.

---

## 4. Interfaz pasiva en timeout y rechazo por permisos

**Síntomas**

Primero Zabbix mostró:

```text
Get value from agent failed: cannot establish TCP connection to
[192.0.0.73:10050]: timed out
```

Después de abrir el puerto, mostró:

```text
Received empty response from Zabbix Agent at [192.0.0.73].
Assuming that agent dropped connection because of access permissions.
```

### Diagnóstico de red

Agent 2 sí escuchaba en todas las interfaces:

```bash
sudo ss -lntp | grep ':10050'
```

```text
LISTEN 0 4096 *:10050 *:* users:(("zabbix_agent2",pid=<PID>,fd=<FD>))
```

La interfaz `ens32` pertenecía a la zona `public`. La regla existente autorizaba únicamente:

```text
192.20.0.10/32 -> 10050/tcp
```

La prueba desde Windows confirmó que la conexión salía desde:

```text
SourceAddress    : 192.20.0.12
TcpTestSucceeded : False
```

Una captura en Oracle Linux confirmó la llegada de los paquetes SYN:

```bash
sudo timeout 60 tcpdump -nni ens32 tcp port 10050 -c 3
```

```text
IP 192.20.0.12.<PUERTO> > 192.0.0.73.10050: Flags [S]
```

### Corrección de `firewalld`

Se agregó una regla temporal para validar el origen real:

```bash
sudo firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="192.20.0.12/32" port port="10050" protocol="tcp" accept'
```

Resultado:

```text
TcpTestSucceeded : True
```

### Corrección de autorización en Agent 2

La configuración original solo permitía:

```ini
Server=192.20.0.10
```

Se autorizó también la IP real de origen:

```ini
Server=192.20.0.10,192.20.0.12
```

Después se reinició y validó el servicio:

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

```text
active
```

### Resultado en Zabbix

La interfaz pasiva cambió a:

```text
192.0.0.73:10050
Estado: Disponible
Error: ninguno
```

### Pendiente antes del cierre definitivo

La regla de `firewalld` para `192.20.0.12/32` sigue siendo temporal. Debe hacerse permanente después de completar la validación funcional de `oracle.ping`:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="192.20.0.12/32" port port="10050" protocol="tcp" accept'
sudo firewall-cmd --reload
```

La regla anterior para `192.20.0.10/32` solo debe retirarse después de confirmar que no corresponde a otro Zabbix Server o proxy.

**Estado:** acceso pasivo resuelto; pendiente persistir la regla de firewall cuando concluya la prueba Oracle.
