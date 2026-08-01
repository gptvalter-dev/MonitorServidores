# Incidencias: Docker Desktop y Windows

## 1. Docker Desktop no detecta virtualización

**Síntoma**

Docker Desktop no inicia y muestra un mensaje relacionado con virtualización, aunque el Administrador de tareas indica:

```text
Virtualización: Habilitada
```

**Diagnóstico**

```powershell
Get-ComputerInfo -Property HyperV*
```

Validar que aparezca:

```text
HyperVisorPresent : True
```

También revisar el arranque del hipervisor:

```powershell
bcdedit /enum {current} | findstr /i hypervisorlaunchtype
```

**Causa identificada**

El hipervisor no estaba configurado explícitamente para iniciar con Windows.

**Solución**

Abrir PowerShell como administrador:

```powershell
bcdedit /set hypervisorlaunchtype auto
```

Reiniciar Windows.

**Estado:** resuelta.

---

## 2. WSL instalado, pero la característica de Windows estaba deshabilitada

**Síntoma**

`wsl --version` funcionaba, pero Docker Desktop continuaba sin iniciar.

**Diagnóstico**

```powershell
Get-WindowsOptionalFeature -Online |
Where-Object { $_.FeatureName -in @(
    "VirtualMachinePlatform",
    "Microsoft-Windows-Subsystem-Linux"
)} |
Select-Object FeatureName, State
```

Resultado encontrado:

```text
VirtualMachinePlatform             Enabled
Microsoft-Windows-Subsystem-Linux Disabled
```

**Solución**

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

Reiniciar Windows.

**Validación**

```powershell
docker run hello-world
```

Debe mostrar:

```text
Hello from Docker!
```

**Estado:** resuelta.

---

## 3. Puertos `10050` y `10051` reservados por Windows

**Síntoma**

Docker o Zabbix Agent 2 no pueden abrir los puertos predeterminados y muestran errores similares a:

```text
bind: An attempt was made to access a socket in a way forbidden by its access permissions
```

**Diagnóstico**

```powershell
netsh interface ipv4 show excludedportrange protocol=tcp
```

Los puertos estaban incluidos dentro de rangos excluidos por Windows.

**Solución aplicada en el laboratorio**

- Zabbix Server publicado en Windows: `11051` → contenedor `10051`.
- Zabbix Agent 2 de Windows: `11050`.

En `.env`:

```dotenv
ZABBIX_SERVER_PORT=11051
```

En `zabbix_agent2.conf` de Windows:

```ini
ListenPort=11050
```

La línea debe estar activa, sin `#`.

**Estado:** resuelta.

---

## 4. No existe `dockerDesktopLinuxEngine`

**Síntoma**

```text
failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine
The system cannot find the file specified
```

**Causa identificada**

Docker Desktop estaba cerrado o el motor Linux todavía no terminaba de iniciar.

**Solución**

1. Abrir Docker Desktop.
2. Esperar a que indique que el motor está en ejecución.
3. Ejecutar nuevamente:

```powershell
$env:OS="alpine"
docker compose up -d
```

**Estado:** resuelta.
