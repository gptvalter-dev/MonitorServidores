# Incidencias: Docker Desktop y Windows

> Este archivo documenta problemas **exclusivos del laboratorio Windows/Docker Desktop**. No copiar estas soluciones literalmente a la arquitectura Linux objetivo.

## 1. Docker Desktop no detecta virtualización

**Síntoma**

Docker Desktop no inicia por un error de virtualización, aunque el Administrador de tareas muestra:

```text
Virtualización: Habilitada
```

**Diagnóstico**

```powershell
Get-ComputerInfo -Property HyperV*
bcdedit /enum {current} | findstr /i hypervisorlaunchtype
```

**Causa**

El hipervisor no estaba configurado para iniciar con Windows.

**Solución**

```powershell
bcdedit /set hypervisorlaunchtype auto
```

Reiniciar Windows.

**Estado:** resuelta.

---

## 2. WSL está instalado, pero la característica de Windows está deshabilitada

**Síntoma**

`wsl --version` funciona, pero Docker Desktop no inicia.

**Diagnóstico**

```powershell
Get-WindowsOptionalFeature -Online |
Where-Object { $_.FeatureName -in @(
    "VirtualMachinePlatform",
    "Microsoft-Windows-Subsystem-Linux"
)} |
Select-Object FeatureName, State
```

**Solución**

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

Reiniciar Windows y validar:

```powershell
docker run hello-world
```

**Estado:** resuelta.

---

## 3. No existe `dockerDesktopLinuxEngine`

**Síntoma**

```text
failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine
The system cannot find the file specified
```

**Causa**

Docker Desktop estaba cerrado o el motor Linux aún no iniciaba.

**Solución**

Abrir Docker Desktop, esperar a que el motor esté operativo y repetir la prueba.

**Estado:** resuelta.

---

## 4. Un Zabbix Server dentro de Docker no llega al Agent 2 usando `127.0.0.1`

**Síntoma**

El Agent 2 de Windows estaba operativo, pero una plantilla de checks pasivos no recibía datos cuando la interfaz Zabbix apuntaba a:

```text
127.0.0.1:<PUERTO_AGENT>
```

**Causa**

Dentro del contenedor de Zabbix Server, `127.0.0.1` representa **el propio contenedor**, no el host Windows.

**Diagnóstico**

Desde el contenedor Zabbix se probó el host de Docker Desktop:

```powershell
docker exec <ZABBIX_SERVER_CONTAINER> sh -c \
  'nc -zvw3 host.docker.internal <PUERTO_AGENT>; echo EXIT_CODE:$?'
```

Después se validó una consulta real:

```powershell
docker exec <ZABBIX_SERVER_CONTAINER> \
  zabbix_get -s host.docker.internal -p <PUERTO_AGENT> -k agent.ping
```

Resultado esperado:

```text
1
```

**Solución del laboratorio**

En la interfaz **Agente** del host en Zabbix se utilizó:

```text
DNS: host.docker.internal
Conectar a: DNS
Puerto: <PUERTO_AGENT>
```

**Lección**

`127.0.0.1` depende del namespace de red donde se ejecuta el proceso. No asumir que loopback dentro de un contenedor representa el host físico.

**Importante para Linux**

`host.docker.internal` fue una solución del laboratorio Docker Desktop. La arquitectura Linux debe usar una ruta de red explícita y validada (IP/DNS del host, red Docker diseñada o `host-gateway` solo si se decide conscientemente).

**Estado:** resuelta en laboratorio.
