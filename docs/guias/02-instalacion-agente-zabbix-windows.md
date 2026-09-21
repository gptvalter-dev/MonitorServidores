# Instalación de Zabbix Agent 2 en Windows

> Alcance: procedimiento normal para instalar/configurar Zabbix Agent 2 en Windows. Incidencias reales: [Agentes Zabbix](../base-conocimiento/agentes-zabbix.md).

## 1. Resultado esperado

- Servicio `Zabbix Agent 2` instalado y `Running`.
- Checks activos funcionales.
- Checks pasivos disponibles si alguna integración los necesita.
- `Zabbix agent ping = Up (1)`.

## 2. Datos requeridos

```text
<HOSTNAME_WINDOWS>        Nombre técnico exacto en Zabbix
<IP_O_DNS_ZABBIX_SERVER>  Dirección accesible del Zabbix Server
<PUERTO_ZABBIX_SERVER>    Normalmente 10051
<PUERTO_AGENT>            Normalmente 10050
<ORIGEN_CHECK_PASIVO>     Server/Proxy autorizado para consultar Agent 2
```

> En el laboratorio Windows se usaron puertos alternos porque los predeterminados estaban ocupados/reservados. No reutilizarlos en otros entornos sin verificar primero.

## 3. Revisar si ya existe Agent 2

Abrir PowerShell como administrador:

```powershell
Get-Service *zabbix* -ErrorAction SilentlyContinue
```

Si existe, confirmar versión antes de instalar otra copia:

```powershell
& "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" -V
```

## 4. Instalar Agent 2

1. Descargar el MSI oficial de la misma rama compatible con el Zabbix Server.
2. Ejecutarlo como administrador.
3. Mantener la ruta estándar cuando no exista una política diferente.

Ruta habitual:

```text
C:\Program Files\Zabbix Agent 2
```

Configuración:

```text
C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf
```

## 5. Respaldar configuración

```powershell
Copy-Item `
  "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf.respaldo"
```

## 6. Configuración base

Abrir:

```powershell
notepad "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf"
```

Si se usarán checks activos y pasivos:

```ini
Hostname=<HOSTNAME_WINDOWS>
Server=<ORIGEN_CHECK_PASIVO>
ServerActive=<IP_O_DNS_ZABBIX_SERVER>:<PUERTO_ZABBIX_SERVER>
ListenPort=<PUERTO_AGENT>
```

Qué controla cada parámetro:

| Parámetro | Función |
|---|---|
| `Hostname` | Identidad usada por checks activos |
| `Server` | Orígenes autorizados para checks pasivos |
| `ServerActive` | Destino de checks activos |
| `ListenPort` | Puerto local de checks pasivos |

No dejar líneas activas duplicadas.

## 7. Validar antes de reiniciar

```powershell
Get-Content "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" |
Select-String '^Server=|^ServerActive=|^Hostname=|^ListenPort='
```

Luego:

```powershell
& "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" `
  -c "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -T
```

No reiniciar si la validación falla. Un plugin externo también puede impedir el arranque aunque el archivo principal sea correcto; revisar el mensaje completo.

## 8. Reiniciar y validar servicio

```powershell
Restart-Service "Zabbix Agent 2"
Get-Service "Zabbix Agent 2"
```

Resultado esperado:

```text
Running
```

Cuando se usen checks pasivos, confirmar listener:

```powershell
Get-NetTCPConnection -State Listen |
Where-Object LocalPort -eq <PUERTO_AGENT>
```

## 9. Crear o validar el host en Zabbix

Ruta:

```text
Recopilación de datos → Equipos
```

Para monitoreo de Windows por checks activos:

```text
Nombre del equipo: <HOSTNAME_WINDOWS>
Plantilla: Windows by Zabbix agent active
Estado: Habilitado
```

Una plantilla exclusivamente activa no necesita interfaz Agent para esas métricas. **Si después se vincula una integración pasiva, como MongoDB mediante Agent 2, se debe agregar una interfaz Agent alcanzable desde Zabbix Server/Proxy.**

## 10. Validar datos

En:

```text
Monitoreo → Últimos datos
```

confirmar:

```text
Zabbix agent ping = Up (1)
```

También deben aparecer CPU, memoria, disco, red y servicios.

## 11. Logs

Localizar la ruta configurada:

```powershell
Select-String `
  -Path "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -Pattern '^LogFile='
```

Leer últimas líneas:

```powershell
Get-Content "<RUTA_LOG>" -Tail 100
```

## 12. Referencias

- Zabbix Agent 2 Windows: https://www.zabbix.com/documentation/7.4/en/manual/appendix/install/windows_agent
- Agent 2 configuration: https://www.zabbix.com/documentation/7.4/en/manual/appendix/config/zabbix_agent2
- Incidencias: [Agentes Zabbix](../base-conocimiento/agentes-zabbix.md)
