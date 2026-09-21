# Incidencias: Zabbix Agent 2

Este archivo conserva únicamente incidencias específicas del agente. Los controles preventivos generales están en [Checklist preventivo Linux](../checklist-preventivo-linux.md).

> Seguridad: se sustituyeron IPs y hostnames reales del laboratorio por placeholders.

## 1. Agent 2 en Windows no inicia por puerto reservado

**Síntoma**

```text
cannot start server listener
Listen failed: listen tcp 0.0.0.0:10050
```

**Causa**

El puerto predeterminado estaba reservado/ocupado en Windows.

**Solución del laboratorio**

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

> En Linux objetivo se debe preferir `10050` cuando no exista conflicto; no copiar `11050` por costumbre.

---

## 2. Agente clásico estático termina con `Segmentation fault`

**Ambiente**

- Oracle Linux 8.x.
- Binario manual bajo `/opt/zabbix`.
- Agente clásico enlazado estáticamente.

**Síntoma**

El binario mostraba versión y validaba configuración, pero al iniciar terminaba con:

```text
Segmentation fault (core dumped)
```

**Solución aplicada**

Se sustituyó la instalación manual por **Zabbix Agent 2 desde el repositorio oficial compatible**.

Ejemplo para la rama 7.4 en Oracle Linux 8:

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/zabbix-release-latest-7.4.el8.noarch.rpm
dnf clean all
dnf makecache
dnf install -y zabbix-agent2
```

**Estado:** resuelta.

---

## 3. Checks activos sin datos

**Síntomas**

```text
Zabbix agent is not available (or no data for 30m)
cannot connect to [<IP_ZABBIX_SERVER>:<PUERTO_ZABBIX_SERVER>]
no route to host
```

**Causa encontrada**

`ServerActive` apuntaba a una dirección del servidor que no era alcanzable desde el host monitoreado. Otra dirección/interfaz sí lo era.

**Corrección**

```ini
ServerActive=<IP_ZABBIX_SERVER_ALCANZABLE>:<PUERTO_ZABBIX_SERVER>
Hostname=<HOSTNAME_ZABBIX>
```

```bash
sudo zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

**Validación**

`Zabbix agent ping` debe tener valor reciente `Up (1)`.

**Estado:** resuelta.

---

## 4. Check pasivo: timeout y después rechazo por permisos

**Síntomas**

Primero:

```text
cannot establish TCP connection to [<IP_HOST>:10050]: timed out
```

Después de resolver red/firewall:

```text
Received empty response from Zabbix Agent...
Assuming that agent dropped connection because of access permissions.
```

**Diagnóstico**

1. Confirmar listener:

```bash
sudo ss -lntp | grep ':10050'
```

2. Identificar el origen real de la conexión:

```bash
sudo tcpdump -nni any tcp port 10050
```

3. Autorizar únicamente ese origen en Agent 2:

```ini
Server=<IP_ZABBIX_SERVER_O_PROXY_AUTORIZADO>
```

4. Crear regla permanente de firewall para el mismo origen.

**Validación**

- interfaz Agent disponible;
- `zabbix_get` responde cuando aplique;
- `Zabbix agent ping = Up (1)`;
- regla persiste después de `firewall-cmd --reload`.

**Estado:** resuelta.

---

## 5. Un plugin externo impide iniciar todo Agent 2

**Ambiente del laboratorio:** Windows, Agent 2 7.4.12.

**Síntoma**

Después de instalar plugins y modificar MongoDB, el servicio Agent 2 dejó de iniciar. La primera sospecha fue la configuración MongoDB.

La validación mostró:

```text
ERROR: Cannot register plugins: failed to register metrics of plugin "NVIDIA"
... zabbix-agent2-plugin-nvidia-gpu.exe ... is not a valid Win32 application
```

**Diagnóstico correcto**

```powershell
& "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" `
  -c "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -T
```

**Causa**

Un plugin NVIDIA instalado junto con otros plugins no podía ejecutarse y bloqueaba el registro de plugins del Agent 2. El problema no era MongoDB.

**Solución del laboratorio**

Deshabilitar/retirar de la ruta incluida la configuración del plugin NVIDIA que no se utilizaría, sin borrar evidencias hasta confirmar el diagnóstico. Después:

```text
Validation successful
```

El servicio volvió a iniciar normalmente.

**Lección**

- Validar siempre Agent 2 con `-T` antes de reiniciar.
- Instalar/activar solo plugins necesarios.
- Un fallo de un plugin puede impedir el arranque global del Agent 2.
- No atribuir automáticamente el fallo al último archivo que se editó.

**Estado:** resuelta.
