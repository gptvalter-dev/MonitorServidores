# Incidencias: Zabbix en Docker Compose

## 1. Compose intenta usar imágenes `Windows_NT-7.4-latest`

**Síntoma**

```text
failed to resolve reference "zabbix/zabbix-server-mysql:Windows_NT-7.4-latest"
```

**Causa**

PowerShell proporciona `OS=Windows_NT` y Compose lo usa para construir la etiqueta de las imágenes.

**Diagnóstico**

```powershell
docker compose config --images
```

**Solución**

```powershell
$env:OS="alpine"
docker compose config --images
docker compose up -d
```

Las imágenes de Zabbix deben usar la etiqueta `alpine-7.4-latest`.

**Estado:** resuelta.

---

## 2. `server-db-init` falla con acceso denegado a MySQL

**Síntoma**

```text
Access denied for user 'zabbix'
service "server-db-init" didn't complete successfully
```

**Causa**

Los secretos de `env_vars` tenían finales de línea Windows `CRLF`; los contenedores Linux leían caracteres adicionales.

**Diagnóstico**

```powershell
(Get-Item .\env_vars\.MYSQL_USER).Length
```

El archivo `.MYSQL_USER` medía 8 bytes en vez de 7.

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

Solo en una instalación nueva, eliminar la base incompleta y reconstruir:

```powershell
docker compose down
Remove-Item -Recurse -Force .\zbx_env\var\lib\mysql
docker compose up -d
```

> No eliminar el directorio MySQL si contiene información que deba conservarse.

**Estado:** resuelta.

---

## 3. Zabbix Server no puede publicar el puerto `10051`

**Síntoma**

```text
ports are not available
listen tcp 0.0.0.0:10051: bind: An attempt was made to access a socket...
```

**Causa**

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

Los agentes activos remotos deben usar:

```ini
ServerActive=<IP_ZABBIX_SERVER>:11051
```

**Estado:** resuelta.
