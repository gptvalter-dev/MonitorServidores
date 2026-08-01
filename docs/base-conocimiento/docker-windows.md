# Incidencias: Docker Desktop y Windows

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

Abrir PowerShell como administrador:

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

Abrir Docker Desktop, esperar a que el motor esté en ejecución y repetir el comando de Docker o Compose.

**Estado:** resuelta.
