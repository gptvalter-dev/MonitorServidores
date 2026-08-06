# Instalación de Zabbix Server en Windows con Docker Desktop

> Alcance: esta guía instala únicamente la plataforma central de Zabbix en un equipo Windows. No incluye la instalación de agentes ni la integración con Oracle Database.

## 1. Objetivo

Al finalizar deberán estar funcionando:

- Docker Desktop con backend WSL 2.
- MySQL para Zabbix.
- Zabbix Server 7.4.
- Interfaz web de Zabbix.
- Acceso a la consola web desde el navegador.

## 2. Requisitos

En el equipo Windows se necesita:

- Permisos de administrador.
- Acceso a Internet.
- WSL 2 habilitado.
- Docker Desktop.
- Git.
- PowerShell.

La documentación oficial indica que Docker Compose es la forma más rápida de desplegar los componentes porque descarga imágenes, crea redes, prepara almacenamiento e inicia los servicios en el orden correcto.

## 3. Abrir PowerShell como administrador

1. Abrir el menú **Inicio**.
2. Escribir `PowerShell`.
3. Hacer clic derecho sobre **Windows PowerShell**.
4. Seleccionar **Ejecutar como administrador**.
5. Aceptar el aviso de Control de cuentas de usuario.

La ventana debe mostrar `Administrador` en el título.

## 4. Verificar WSL 2

Ejecutar:

```powershell
wsl --version
```

Verificar que las características necesarias estén habilitadas:

```powershell
Get-WindowsOptionalFeature -Online |
Where-Object { $_.FeatureName -in @(
    "VirtualMachinePlatform",
    "Microsoft-Windows-Subsystem-Linux"
)} |
Select-Object FeatureName, State
```

Resultado esperado:

```text
VirtualMachinePlatform                 Enabled
Microsoft-Windows-Subsystem-Linux     Enabled
```

Si alguna aparece como `Disabled`, debe habilitarse antes de continuar.

## 5. Instalar y validar Docker Desktop

1. Descargar Docker Desktop desde el sitio oficial.
2. Ejecutar el instalador.
3. Mantener habilitado el uso de WSL 2.
4. Reiniciar Windows si el instalador lo solicita.
5. Abrir Docker Desktop.
6. Esperar hasta que el motor Linux aparezca iniciado.

Validar desde PowerShell:

```powershell
docker run hello-world
```

Resultado esperado:

```text
Hello from Docker!
```

También validar Docker Compose:

```powershell
docker compose version
```

La documentación de Zabbix 7.4 recomienda Docker Compose 2.24 o posterior.

## 6. Verificar Git

Ejecutar:

```powershell
git --version
```

Si el comando no existe, instalar Git para Windows y volver a abrir PowerShell.

## 7. Crear la carpeta de trabajo

Ejecutar:

```powershell
New-Item -ItemType Directory -Path C:\docker -Force
Set-Location C:\docker
```

Confirmar la ubicación actual:

```powershell
Get-Location
```

Resultado esperado:

```text
C:\docker
```

## 8. Clonar el repositorio oficial

Ejecutar:

```powershell
git clone https://github.com/zabbix/zabbix-docker.git
Set-Location C:\docker\zabbix-docker
git checkout 7.4
```

Confirmar la rama:

```powershell
git branch --show-current
```

Resultado esperado:

```text
7.4
```

## 9. Localizar el archivo `.env`

El archivo se encuentra en:

```text
C:\docker\zabbix-docker\.env
```

Confirmar que existe:

```powershell
Set-Location C:\docker\zabbix-docker
Get-ChildItem -Force .env
```

El parámetro `-Force` es necesario porque los nombres que comienzan con punto pueden no mostrarse en algunas vistas.

## 10. Crear un respaldo de `.env`

Antes de modificarlo:

```powershell
Copy-Item .env .env.respaldo
```

Confirmar ambos archivos:

```powershell
Get-ChildItem -Force .env*
```

Deben aparecer al menos:

```text
.env
.env.respaldo
```

## 11. Abrir `.env`

Desde PowerShell:

```powershell
notepad C:\docker\zabbix-docker\.env
```

En Bloc de notas:

1. Buscar cada variable con `Ctrl + F`.
2. Modificar el valor después del signo `=`.
3. No agregar espacios alrededor de `=`.

Configurar:

```dotenv
ZABBIX_WEB_NGINX_HTTP_PORT=8080
ZABBIX_WEB_NGINX_HTTPS_PORT=8443
ZABBIX_SERVER_PORT=11051
```

### Qué significa cada variable

| Variable | Uso |
|---|---|
| `ZABBIX_WEB_NGINX_HTTP_PORT` | Puerto HTTP de la interfaz web |
| `ZABBIX_WEB_NGINX_HTTPS_PORT` | Puerto HTTPS de la interfaz web |
| `ZABBIX_SERVER_PORT` | Puerto publicado del Zabbix Server para agentes activos |

En este laboratorio se utiliza `11051` porque el puerto `10051` del equipo Windows no estaba disponible.

## 12. Guardar correctamente `.env`

En Bloc de notas:

1. Seleccionar **Archivo → Guardar**.
2. Cerrar Bloc de notas.

Confirmar que Windows no creó `.env.txt`:

```powershell
Get-ChildItem -Force .env*
```

Si aparece `.env.txt`, renombrarlo:

```powershell
Rename-Item .env.txt .env
```

Validar las variables guardadas:

```powershell
Select-String -Path .env -Pattern '^ZABBIX_WEB_NGINX_HTTP_PORT=|^ZABBIX_WEB_NGINX_HTTPS_PORT=|^ZABBIX_SERVER_PORT='
```

## 13. Configurar la imagen base

En cada nueva ventana de PowerShell usada para este laboratorio:

```powershell
$env:OS="alpine"
```

Confirmar:

```powershell
$env:OS
```

Resultado esperado:

```text
alpine
```

## 14. Validar la configuración antes de iniciar

Desde:

```text
C:\docker\zabbix-docker
```

Ejecutar:

```powershell
docker compose config --images
```

Deben aparecer imágenes similares a:

```text
zabbix/zabbix-server-mysql:alpine-7.4-latest
zabbix/zabbix-web-nginx-mysql:alpine-7.4-latest
mysql:8.4-oracle
```

Si aparecen etiquetas como `Windows_NT-7.4-latest`, revisar que `$env:OS` sea `alpine`.

## 15. Iniciar Zabbix

Ejecutar:

```powershell
Set-Location C:\docker\zabbix-docker
$env:OS="alpine"
docker compose up -d
```

La primera ejecución puede tardar porque Docker descarga las imágenes.

## 16. Revisar los contenedores

Ejecutar:

```powershell
docker compose ps
```

Estado esperado:

- MySQL: `healthy`.
- Zabbix Server: `Up`.
- Zabbix Web: `healthy`.

Si algún contenedor aparece como `Exited`, revisar sus registros:

```powershell
docker logs -f <NOMBRE_CONTENEDOR>
```

Para detener la visualización de logs usar `Ctrl + C`.

## 17. Abrir la interfaz web

Abrir el navegador y entrar a:

```text
http://localhost:8080
```

Credenciales iniciales del laboratorio:

```text
Usuario: Admin
Contraseña: zabbix
```

Cambiar la contraseña antes de utilizar la plataforma fuera de un laboratorio controlado.

## 18. Zona horaria de la interfaz web

La documentación oficial permite configurar la zona horaria del contenedor web mediante el archivo de variables del componente web. En el repositorio se encuentra bajo:

```text
C:\docker\zabbix-docker\env_vars\.env_web
```

Antes de modificarlo:

```powershell
Copy-Item .\env_vars\.env_web .\env_vars\.env_web.respaldo
notepad .\env_vars\.env_web
```

La variable es:

```dotenv
PHP_TZ=America/Mexico_City
```

Después debe recrearse el contenedor web:

```powershell
$env:OS="alpine"
docker compose up -d
```

La zona horaria también puede configurarse por usuario desde su perfil; el valor del perfil tiene prioridad sobre el valor global.

## 19. Persistencia de datos

El repositorio oficial utiliza directorios y volúmenes para conservar configuración e históricos entre reinicios. No eliminar carpetas de datos ni volúmenes sin respaldo.

Antes de una actualización del repositorio:

1. Respaldar `.env`.
2. Respaldar los archivos modificados dentro de `env_vars`.
3. Identificar las imágenes y contenedores actuales.
4. Revisar las notas de actualización de Zabbix.

## 20. Operación diaria

### Iniciar o actualizar contenedores

```powershell
Set-Location C:\docker\zabbix-docker
$env:OS="alpine"
docker compose up -d
```

### Detener sin eliminar

```powershell
docker compose stop
```

### Reanudar

```powershell
docker compose start
```

### Ver estado

```powershell
docker compose ps
```

### Eliminar contenedores y redes del proyecto

```powershell
docker compose down
```

`down` no debe ejecutarse sin comprender qué datos son persistentes y qué recursos podrían eliminarse.

## 21. Checklist

- [ ] WSL 2 habilitado.
- [ ] Docker Desktop iniciado.
- [ ] `docker run hello-world` exitoso.
- [ ] Git disponible.
- [ ] Repositorio clonado en `C:\docker\zabbix-docker`.
- [ ] Rama `7.4` activa.
- [ ] Respaldo de `.env` creado.
- [ ] Puertos configurados.
- [ ] `$env:OS=alpine` configurado.
- [ ] `docker compose config --images` correcto.
- [ ] MySQL saludable.
- [ ] Zabbix Server activo.
- [ ] Zabbix Web saludable.
- [ ] Interfaz disponible en `http://localhost:8080`.

## 22. Referencias oficiales

- [Instalación de Zabbix desde contenedores](https://www.zabbix.com/documentation/7.4/es/manual/installation/containers)
- [Actualización de Zabbix desde contenedores](https://www.zabbix.com/documentation/7.4/es/manual/installation/upgrade/containers)
- [Repositorio oficial zabbix-docker](https://github.com/zabbix/zabbix-docker)
- [Incidencias de Docker y Windows](../base-conocimiento/docker-windows.md)
- [Incidencias de Zabbix Docker](../base-conocimiento/zabbix-docker.md)
