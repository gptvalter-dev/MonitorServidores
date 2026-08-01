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

**Síntoma**

El agente escucha en `10050`, pero la disponibilidad pasiva aparece en rojo o muestra timeout.

**Validación local**

```bash
ss -lntp | grep ':10050'
```

Resultado esperado:

```text
LISTEN ... *:10050 ... zabbix_agent2
```

Configuración mínima:

```ini
Server=<IP_ZABBIX_SERVER>
ListenPort=10050
```

**Causa identificada**

`firewalld` bloqueaba el acceso desde Zabbix Server.

**Diagnóstico**

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
```

**Solución segura**

Abrir el puerto únicamente para Zabbix Server:

```bash
firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="<IP_ZABBIX_SERVER>/32" port protocol="tcp" port="10050" accept'

firewall-cmd --reload
firewall-cmd --zone=public --list-rich-rules
```

**Estado:** regla aplicada; validar que la interfaz aparezca disponible en Zabbix.
