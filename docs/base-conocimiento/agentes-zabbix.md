# Incidencias: agentes Zabbix

## 1. Zabbix Agent 2 de Windows no inicia

**Síntoma**

```text
cannot start server listener
Listen failed: listen tcp 0.0.0.0:10050
```

**Causa identificada**

El puerto `10050` estaba reservado por Windows. Además, la línea modificada permanecía comentada:

```ini
# ListenPort=11050
```

**Solución**

Editar:

```text
C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf
```

Configurar sin `#`:

```ini
ListenPort=11050
```

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

**Estado:** resuelta.

---

## 2. Agente clásico estático termina con `Segmentation fault`

**Ambiente**

- Oracle Linux 8.10.
- Binario manual en `/opt/zabbix/sbin/zabbix_agentd`.
- Zabbix Agent 7.4.x enlazado estáticamente.

**Síntoma**

El binario mostraba la versión y validaba la configuración:

```bash
/opt/zabbix/sbin/zabbix_agentd -V
/opt/zabbix/sbin/zabbix_agentd -c /opt/zabbix/conf/zabbix_agentd.conf -T
```

Pero al iniciar:

```text
Segmentation fault (core dumped)
```

**Conclusión**

La sintaxis de configuración era válida; el fallo ocurría durante la inicialización en tiempo de ejecución. Los incidentes antiguos revisados para Zabbix 2.4 y 4.4 no correspondían directamente a la versión 7.4 utilizada.

**Solución aplicada**

Instalar Zabbix Agent 2 desde el repositorio oficial para Oracle Linux 8:

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/zabbix-release-latest-7.4.el8.noarch.rpm
dnf clean all
dnf makecache
dnf install -y zabbix-agent2
```

**Estado:** resuelta mediante sustitución del binario manual.

---

## 3. Comprobaciones activas sin datos

**Síntomas posibles**

```text
cannot connect to [<IP_ZABBIX_SERVER>:11051]
no active checks on server
host [<HOSTNAME_LINUX>] not found
```

**Diagnóstico de conectividad**

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

**Causas identificadas**

- `Hostname` no coincidía exactamente con el campo **Nombre del equipo** en Zabbix.
- La plantilla `Linux by Zabbix agent active` no estaba vinculada directamente al host.
- El agente aún no había actualizado su configuración activa.

**Solución**

Configurar:

```ini
ServerActive=<IP_ZABBIX_SERVER>:11051
Hostname=<HOSTNAME_LINUX>
```

En Zabbix:

```text
Nombre del equipo: <HOSTNAME_LINUX>
Plantilla: Linux by Zabbix agent active
```

Reiniciar:

```bash
systemctl restart zabbix-agent2
```

**Nota:** para una plantilla completamente activa, la interfaz del host puede permanecer vacía.

**Estado:** resuelta.

---

## 4. Interfaz pasiva en timeout

**Síntoma confirmado en Zabbix**

El indicador `ZBX` aparece amarillo y la interfaz pasiva muestra:

```text
No disponible
Get value from agent failed: cannot establish TCP connection to
[<IP_ORACLE_LINUX>:10050]: timed out
```

La prueba desde el servidor Windows donde se ejecuta Zabbix mostró:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

Esto confirma que el servidor responde en red, pero el puerto TCP `10050` no es alcanzable.

**Validaciones realizadas**

En Oracle Linux, Agent 2 sí está escuchando:

```bash
sudo ss -lntp | grep ':10050'
```

Resultado confirmado:

```text
LISTEN 0 4096 *:10050 *:* users:(("zabbix_agent2",pid=<PID>,fd=<FD>))
```

Por lo tanto, el problema no está en el servicio ni en `ListenPort`.

También se confirmó:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
```

Resultado:

```text
running
public
  interfaces: ens32
```

La interfaz de red principal `ens32` está asociada a la zona `public`, por lo que las reglas del puerto `10050` deben revisarse en esa zona.

**Siguiente validación**

```bash
sudo firewall-cmd --zone=public --list-all
```

Se debe verificar si aparece:

```text
ports: 10050/tcp
```

o una `rich rule` que permita `10050/tcp` desde `<IP_ZABBIX_SERVER>/32`.

**Causas aún posibles**

1. La zona `public` no permite `10050/tcp`.
2. La regla existe, pero autoriza otra IP de origen.
3. Existe un filtro de red intermedio entre Zabbix Server y Oracle Linux.

**Criterio de cierre**

Desde el servidor Zabbix:

```powershell
Test-NetConnection <IP_ORACLE_LINUX> -Port 10050
```

Debe mostrar:

```text
TcpTestSucceeded : True
```

Después, la interfaz `ZBX` debe aparecer disponible en Zabbix.

**Estado:** pendiente. Punto de reanudación: revisar la configuración completa de la zona `public`.
