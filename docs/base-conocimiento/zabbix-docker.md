# Incidencias: Zabbix en Docker Compose

## 1. Compose intenta usar imágenes `Windows_NT-7.4-latest`

**Síntoma**

```text
failed to resolve reference "zabbix/zabbix-server-mysql:Windows_NT-7.4-latest"
```

**Causa identificada**

PowerShell proporciona la variable de entorno del sistema operativo como `OS=Windows_NT`, y Compose la utiliza para construir la etiqueta de las imágenes.

**Diagnóstico**

```powershell
docker compose config --images
```

**Solución**

En cada nueva sesión de PowerShell:

```powershell
$env:OS="alpine"
docker compose config --images
```

Resultado esperado:

```text
zabbix/zabbix-server-mysql:alpine-7.4-latest
zabbix/zabbix-web-nginx-mysql:alpine-7.4-latest
mysql:8.4-oracle
```

Después:

```powershell
docker compose up -d
```

**Estado:** resuelta.

---

## 2. `server-db-init` falla con acceso denegado a MySQL

**Síntoma**

```text
Access denied for user 'zabbix'
service "server-db-init" didn't complete successfully
```

**Causa identificada**

Los archivos secretos de `env_vars` tenían finales de línea Windows `CRLF`. Los contenedores Linux leían caracteres adicionales en el usuario o la contraseña.

**Diagnóstico**

```powershell
(Get-Item .\env_vars\.MYSQL_USER).Length
```

El archivo `.MYSQL_USER` medía 8 bytes; se esperaba `zabbix` con un solo salto `LF`, es decir, 7 bytes.

**Solución**

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

Como era una instalación nueva, se eliminó la base incompleta:

```powershell
$env:OS="alpine"
docker compose down
Remove-Item -Recurse -Force .\zbx_env\var\lib\mysql
```

Después se inició nuevamente:

```powershell
docker compose up -d
```

**Precaución:** no eliminar el directorio MySQL cuando existan datos que deban conservarse.

**Estado:** resuelta.

---

## 3. Zabbix Server no puede publicar el puerto `10051`

**Síntoma**

```text
ports are not available
listen tcp 0.0.0.0:10051: bind: An attempt was made to access a socket...
```

**Causa identificada**

El puerto `10051` estaba reservado por Windows.

**Diagnóstico**

```powershell
netsh interface ipv4 show excludedportrange protocol=tcp
```

**Solución**

En `.env`:

```dotenv
ZABBIX_SERVER_PORT=11051
```

El mapeo resultante es:

```text
Windows 11051 → contenedor Zabbix Server 10051
```

Los agentes activos remotos deben utilizar:

```ini
ServerActive=<IP_ZABBIX_SERVER>:11051
```

**Estado:** resuelta.

---

## 4. Verificación rápida de contenedores

```powershell
docker compose ps
```

Estado esperado:

| Servicio | Estado esperado |
|---|---|
| `mysql-server` | `healthy` |
| `zabbix-server` | `Up` |
| `zabbix-web-nginx-mysql` | `healthy` |

Logs útiles:

```powershell
docker compose logs --no-color server-db-init
docker compose logs --no-color zabbix-server
docker compose logs --no-color zabbix-web-nginx-mysql
```
