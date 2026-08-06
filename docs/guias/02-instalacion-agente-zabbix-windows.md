# Instalación de Zabbix Agent 2 en Windows

> Alcance: esta guía instala y configura únicamente Zabbix Agent 2 en Windows. El Zabbix Server debe estar funcionando antes de comenzar.

## 1. Objetivo

Al finalizar:

- El servicio **Zabbix Agent 2** estará instalado y activo.
- El agente podrá enviar comprobaciones activas al Zabbix Server.
- El host Windows aparecerá con métricas recientes en Zabbix.

## 2. Datos que se deben conocer

Antes de instalar, definir:

```text
<HOSTNAME_WINDOWS>       Nombre técnico que tendrá el host en Zabbix
<IP_ZABBIX_SERVER>       Dirección accesible del Zabbix Server
<PUERTO_ZABBIX_SERVER>   Puerto publicado del Zabbix Server; en el laboratorio es 11051
<PUERTO_AGENTE_WINDOWS>  Puerto donde escuchará el agente; en el laboratorio es 11050
```

Para comprobaciones activas, `Hostname` debe coincidir exactamente con el campo **Nombre del equipo** en Zabbix, incluyendo mayúsculas, minúsculas y guiones.

## 3. Descargar e instalar Agent 2

1. Descargar el instalador MSI de Zabbix Agent 2 para Windows desde el sitio oficial de Zabbix.
2. Ejecutar el MSI como administrador.
3. Completar el asistente.
4. Mantener la ruta predeterminada cuando no exista una política distinta.

Ruta habitual de instalación:

```text
C:\Program Files\Zabbix Agent 2
```

## 4. Localizar el archivo de configuración

El archivo principal normalmente se encuentra en:

```text
C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf
```

Abrir PowerShell como administrador y comprobarlo:

```powershell
Test-Path "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf"
```

Resultado esperado:

```text
True
```

## 5. Crear un respaldo

Ejecutar:

```powershell
Copy-Item `
  "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf.respaldo"
```

Confirmar:

```powershell
Get-ChildItem "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf*"
```

## 6. Abrir el archivo como administrador

Desde PowerShell abierto como administrador:

```powershell
notepad "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf"
```

Si el archivo se abre pero no permite guardar, cerrar Bloc de notas, volver a abrir PowerShell como administrador y repetir el comando.

## 7. Configurar el agente

Buscar con `Ctrl + F` y configurar:

```ini
Hostname=<HOSTNAME_WINDOWS>
Server=<IP_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>:<PUERTO_ZABBIX_SERVER>
ListenPort=<PUERTO_AGENTE_WINDOWS>
```

Ejemplo de laboratorio:

```ini
Hostname=<HOSTNAME_WINDOWS>
Server=127.0.0.1
ServerActive=127.0.0.1:11051
ListenPort=11050
```

### Qué hace cada parámetro

| Parámetro | Función |
|---|---|
| `Hostname` | Identificador usado por las comprobaciones activas |
| `Server` | Direcciones autorizadas para consultas pasivas |
| `ServerActive` | Dirección y puerto a los que el agente enviará datos activos |
| `ListenPort` | Puerto local de consultas pasivas |

No dejar dos líneas activas para el mismo parámetro. Las líneas comentadas comienzan con `#` y no se aplican.

## 8. Guardar y validar el archivo

En Bloc de notas seleccionar **Archivo → Guardar** y cerrar.

Comprobar las líneas activas:

```powershell
Select-String `
  -Path "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -Pattern '^(Hostname|Server|ServerActive|ListenPort)='
```

## 9. Validar la configuración antes de reiniciar

Ejecutar:

```powershell
& "C:\Program Files\Zabbix Agent 2\zabbix_agent2.exe" `
  -c "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -T
```

Resultado esperado: validación correcta sin errores de sintaxis.

Si aparece un error, no reiniciar todavía. Corregir la línea y repetir la prueba.

## 10. Iniciar o reiniciar el servicio

Comprobar si existe:

```powershell
Get-Service "Zabbix Agent 2"
```

Reiniciar:

```powershell
Restart-Service "Zabbix Agent 2"
```

Si está detenido:

```powershell
Start-Service "Zabbix Agent 2"
```

Confirmar:

```powershell
Get-Service "Zabbix Agent 2"
```

Resultado esperado:

```text
Status : Running
```

## 11. Comprobar el puerto local

Ejecutar:

```powershell
Get-NetTCPConnection -State Listen |
Where-Object LocalPort -eq <PUERTO_AGENTE_WINDOWS>
```

En el laboratorio:

```powershell
Get-NetTCPConnection -State Listen |
Where-Object LocalPort -eq 11050
```

Si no aparece, revisar el servicio y `ListenPort`.

## 12. Crear el host en Zabbix

En la interfaz web:

```text
Recopilación de datos → Equipos → Crear equipo
```

Configurar:

```text
Nombre del equipo: <HOSTNAME_WINDOWS>
Nombre visible: descripción amigable opcional
Grupo: Windows servers
Plantilla: Windows by Zabbix agent active
Estado: Habilitado
```

Para una plantilla activa no es obligatorio crear una interfaz de agente.

El campo **Nombre del equipo** debe coincidir exactamente con `Hostname=`.

## 13. Validar datos en Zabbix

Esperar algunos minutos y abrir:

```text
Monitoreo → Últimos datos
```

Filtrar por el host Windows y buscar:

```text
Zabbix agent ping
```

Resultado esperado:

```text
Up (1)
```

También deben comenzar a aparecer métricas de CPU, memoria, disco, red y servicios.

## 14. Incidencia: el puerto 10050 está ocupado

Síntoma:

```text
cannot start server listener
Listen failed: listen tcp 0.0.0.0:10050
```

Validar quién utiliza el puerto:

```powershell
Get-NetTCPConnection -LocalPort 10050 -ErrorAction SilentlyContinue
```

En el laboratorio se decidió utilizar:

```ini
ListenPort=11050
```

Después se validó la configuración y se reinició el servicio.

## 15. Revisar registros

La ubicación exacta del log depende de la configuración del agente. Buscar el parámetro:

```powershell
Select-String `
  -Path "C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf" `
  -Pattern '^LogFile='
```

Abrir las últimas líneas del archivo indicado:

```powershell
Get-Content "<RUTA_LOG>" -Tail 50
```

## 16. Checklist

- [ ] MSI de Agent 2 instalado.
- [ ] Archivo de configuración localizado.
- [ ] Respaldo creado.
- [ ] `Hostname` coincide con Zabbix.
- [ ] `ServerActive` apunta a un puerto accesible.
- [ ] `ListenPort` no está ocupado.
- [ ] Validación `-T` correcta.
- [ ] Servicio `Running`.
- [ ] Host creado con plantilla activa.
- [ ] `Zabbix agent ping = Up (1)`.
- [ ] Métricas recientes visibles.

## 17. Referencias

- [Zabbix agent en Microsoft Windows](https://www.zabbix.com/documentation/7.4/en/manual/appendix/install/windows_agent)
- [Configuración de Zabbix Agent 2](https://www.zabbix.com/documentation/7.4/en/manual/appendix/config/zabbix_agent2)
- [Incidencias de agentes](../base-conocimiento/agentes-zabbix.md)
