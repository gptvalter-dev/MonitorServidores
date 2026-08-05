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

La prueba desde Windows mostró:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

**Validaciones realizadas**

Agent 2 está escuchando correctamente en todas las interfaces:

```bash
sudo ss -lntp | grep ':10050'
```

Resultado:

```text
LISTEN 0 4096 *:10050 *:* users:(("zabbix_agent2",pid=<PID>,fd=<FD>))
```

`firewalld` está activo y `ens32` pertenece a la zona `public`:

```text
running
public
  interfaces: ens32
```

La zona contiene una regla para `10050/tcp`, pero está limitada a una dirección de origen distinta:

```text
rule family="ipv4" source address="<IP_ORIGEN_CONFIGURADA>/32" port port="10050" protocol="tcp" accept
```

Al ejecutar:

```powershell
Test-NetConnection <IP_ORACLE_LINUX> -Port 10050 -InformationLevel Detailed
```

se confirmó que la conexión realmente sale desde otra dirección:

```text
SourceAddress    : <IP_ORIGEN_REAL>
TcpTestSucceeded : False
```

**Causa identificada**

La regla de `firewalld` autoriza una IP distinta de la IP real de origen del servidor Zabbix. Por eso el ping funciona, pero el puerto `10050/tcp` termina en timeout.

**Procedimiento seguro de corrección**

1. Agregar primero una regla temporal para `<IP_ORIGEN_REAL>/32`.
2. Repetir `Test-NetConnection`.
3. Si devuelve `True`, hacer permanente la regla correcta.
4. Retirar la regla anterior únicamente después de validar Zabbix.

**Criterio de cierre**

```text
TcpTestSucceeded : True
```

Después, la interfaz `ZBX` debe aparecer disponible en Zabbix.

**Estado:** causa identificada; pendiente probar la regla con la IP real de origen.
