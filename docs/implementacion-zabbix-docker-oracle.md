# Índice de implementación y exploración de Zabbix

> Fecha de actualización: **20 de septiembre de 2026**  
> Estado: **exploración técnica en curso**.

Este archivo es el punto de entrada a la documentación del proyecto. Los procedimientos, incidencias, interpretación de métricas y controles preventivos se mantienen separados para facilitar su consulta y repetición.

---

# 1. Dirección objetivo

La experiencia del laboratorio permitió validar Zabbix inicialmente sobre Windows + Docker Desktop, pero la **dirección objetivo del proyecto es que la plataforma central quede sobre Linux**.

Objetivo de arquitectura:

```text
Linux / Oracle Linux dedicado
├── Zabbix Server
├── Zabbix Frontend
├── Base de datos de Zabbix
├── Zabbix Agent 2
└── Servicios administrados por systemd

Servidores monitoreados
├── Oracle Linux con Oracle Database
├── Linux con aplicaciones
├── Linux con Docker
└── Otros equipos según inventario
```

La instalación Windows + Docker Desktop se conserva como:

- laboratorio ya validado;
- referencia de aprendizaje;
- fuente de incidencias y soluciones;
- comparación contra la instalación Linux definitiva.

No debe asumirse que una configuración que funcionó con Docker Desktop, WSL 2 o puertos alternos de Windows debe copiarse directamente a Linux.

---

# 2. Orden recomendado

## Paso 1. Instalar Zabbix Server

Existen dos guías:

```text
1A. Windows con Docker Desktop          → laboratorio validado
1B. Oracle Linux con paquetes oficiales → arquitectura objetivo
```

Para la siguiente etapa se debe priorizar la guía **1B** y validar completamente la instalación Linux.

Después continuar con:

2. Instalar Agent 2 en el sistema operativo que corresponda.
3. Validar red, comprobaciones activas y pasivas.
4. Vincular la plantilla del sistema operativo.
5. Configurar integraciones específicas: Oracle, Docker o aplicaciones.
6. Interpretar métricas.
7. Parametrizar umbrales.
8. Configurar alertas y notificaciones.
9. Validar reinicios, persistencia y recuperación.

No avanzar al siguiente nivel si el anterior todavía presenta errores.

---

# 3. Guías de implementación

| Paso | Guía | Propósito | Estado |
|---:|---|---|---|
| 1A | [Instalación de Zabbix Server en Windows con Docker](guias/01-instalacion-zabbix-windows-docker.md) | Laboratorio con WSL 2, Docker Desktop, MySQL y Zabbix | Validada |
| 1B | [Instalación de Zabbix Server en Oracle Linux](guias/01b-instalacion-zabbix-server-oracle-linux.md) | Instalar Zabbix Server, base de datos, frontend y Agent 2 como servicios Linux | Documentada; pendiente de ejecución completa |
| 2 | [Instalación de Zabbix Agent 2 en Windows](guias/02-instalacion-agente-zabbix-windows.md) | Monitorear equipos Windows cuando existan | Validada |
| 3 | [Instalación de Zabbix Agent 2 en Oracle Linux](guias/03-instalacion-agente-zabbix-oracle-linux.md) | Instalar Agent 2, red, firewall y comprobaciones | Validada |
| 4 | [Monitoreo de Oracle Database](guias/04-monitoreo-oracle-database.md) | Configurar usuario, macros, Oracle Client y `Oracle Ping` | Funcional; auditoría pendiente |
| 5 | [Interpretación de gráficas](guias/05-interpretacion-graficas-zabbix.md) | Comprender ejes, periodos, leyendas y zona horaria | Iniciada |
| 6 | [Interpretación inicial de métricas](guias/06-interpretacion-metricas-iniciales.md) | Documentar métricas de Linux y Oracle | En desarrollo |
| 7 | [Parametrización de métricas](guias/07-parametrizacion-metricas-zabbix.md) | Ajustar macros, intervalos y métricas personalizadas | Pendiente de prueba completa |
| 8 | [Alertas y notificaciones](guias/08-alertas-y-notificaciones-zabbix.md) | Configurar avisos, recuperación y escalamiento | Pendiente de prueba |
| 9 | [Ejecución remota y actualizaciones](guias/09-ejecucion-remota-y-actualizaciones.md) | Documentar capacidad de ejecutar scripts y acciones remotas con controles de seguridad | Documentada; pendiente de laboratorio |

---

# 4. Lecciones aprendidas y prevención

Estos dos documentos deben revisarse **antes de desplegar la plataforma Linux o incorporar un nuevo host**:

| Documento | Propósito |
|---|---|
| [Lecciones aprendidas y prevención Linux](lecciones-aprendidas-y-prevencion-linux.md) | Explica los problemas encontrados, su causa y el principio preventivo que debe conservarse |
| [Checklist preventivo Linux](checklist-preventivo-linux.md) | Lista de revisión previa para Zabbix Server, Agent 2, red, firewall, Oracle, Docker, plantillas, métricas y seguridad |

Principio general adoptado:

```text
Primero validar infraestructura.
Después validar agente.
Después validar integración.
Después interpretar la métrica.
Finalmente parametrizar la alerta.
```

---

# 5. Base de conocimiento

Las guías describen el procedimiento normal. La base de conocimiento conserva los errores reales encontrados, diagnóstico, causa y solución.

| Tema | Archivo |
|---|---|
| Índice de incidencias | [Base de conocimiento](base-conocimiento/README.md) |
| Docker Desktop y Windows | [docker-windows.md](base-conocimiento/docker-windows.md) |
| Zabbix Docker | [zabbix-docker.md](base-conocimiento/zabbix-docker.md) |
| Agentes Windows y Linux | [agentes-zabbix.md](base-conocimiento/agentes-zabbix.md) |
| Oracle Database | [oracle.md](base-conocimiento/oracle.md) |

Separación utilizada:

```text
docs/guias/
Procedimientos normales, validaciones y checklist de cada función.

docs/base-conocimiento/
Síntomas, diagnóstico, causa, solución y evidencia de incidencias.

docs/lecciones-aprendidas-y-prevencion-linux.md
Aprendizajes transversales del proyecto.

docs/checklist-preventivo-linux.md
Control previo obligatorio antes de nuevas instalaciones e integraciones.
```

Cuando se ejecute la instalación definitiva de Zabbix Server en Oracle Linux, las incidencias específicas deberán registrarse en un archivo propio dentro de `docs/base-conocimiento/`.

---

# 6. Reglas de documentación

Cada procedimiento que modifique un archivo debe indicar:

1. En qué equipo se realiza.
2. Qué usuario o permisos se necesitan.
3. La ruta completa del archivo.
4. Cómo comprobar que existe.
5. Cómo crear un respaldo.
6. Cómo abrirlo.
7. Qué líneas modificar.
8. Cómo guardar y salir.
9. Cómo validar la sintaxis.
10. Cómo aplicar el cambio.
11. Qué resultado se espera.
12. Qué revisar si el resultado no coincide.
13. Cómo revertir el cambio cuando corresponda.

No se usarán instrucciones incompletas como:

```text
Editar archivo.
Configurar firewall.
Agregar plantilla.
```

Cada una debe convertirse en un procedimiento ejecutable de principio a fin.

Los procedimientos tomados de documentación oficial pero todavía no ejecutados deben marcarse como **pendientes de validación**.

---

# 7. Seguridad

Este repositorio es público. Utilizar únicamente valores genéricos:

```text
<IP_ZABBIX_SERVER>
<IP_ORIGEN_COMPROBACION_PASIVA>
<IP_ORACLE_LINUX>
<HOSTNAME_LINUX>
<ORACLE_SERVICE>
<ORACLE_HOME>
<CONTRASENA_SEGURA>
<CONTRASENA_ROOT_BD>
<CONTRASENA_BD_ZABBIX>
```

No publicar:

- contraseñas;
- direcciones internas reales;
- nombres reales de servidores productivos;
- usuarios de aplicación;
- cadenas de conexión productivas;
- tokens;
- secretos.

---

# 8. Arquitectura actualmente validada en laboratorio

```text
Windows
├── Docker Desktop + WSL 2
│   ├── MySQL 8.4
│   ├── Zabbix Server 7.4
│   └── Zabbix Web Nginx
├── Zabbix Agent 2 para Windows
│
└── Red interna
    └── Oracle Linux 8.10
        ├── Zabbix Agent 2
        └── Oracle Database 19c
```

Esta arquitectura demostró funcionalidad, pero también expuso complejidades de:

- múltiples interfaces;
- NAT;
- IP real de origen;
- puertos alternos;
- Docker Desktop;
- WSL 2.

Estas experiencias son la principal razón para simplificar la plataforma central sobre Linux.

---

# 9. Arquitectura Linux objetivo

```text
Zabbix Server Linux
├── Base de datos
├── Zabbix Server
├── Frontend
├── Agent 2
└── Firewall/SELinux controlados

        │
        ├── Linux + Oracle Database
        │   ├── Linux by Zabbix agent active
        │   └── Oracle by Zabbix agent 2
        │
        └── Linux + Docker + aplicaciones
            ├── Linux by Zabbix agent active
            ├── Docker by Zabbix agent 2
            └── Validaciones HTTP/puerto/endpoint por aplicación
```

En servidores Docker no se considerará una aplicación saludable únicamente porque el contenedor aparezca como `running`.

---

# 10. Estado funcional alcanzado

| Componente | Estado |
|---|---|
| Zabbix Server 7.4 en Docker/Windows | Operativo |
| Frontend | Operativo |
| Base de datos de Zabbix en laboratorio | Operativa |
| Agent 2 en Oracle Linux | Operativo |
| Comprobaciones activas Linux | `Zabbix agent ping = Up (1)` |
| Comprobaciones pasivas | Validadas |
| Firewall persistente de Oracle Linux | Validado |
| Oracle Client para Agent 2 | Configurado mediante `systemd` |
| Oracle Ping | `Up (1)` |
| Métricas Oracle | En revisión |
| Zabbix Server directamente en Oracle Linux | Pendiente de ejecución completa |
| Servidor Linux con Docker/aplicaciones | Pendiente de incorporación |
| Métricas personalizadas | Pendientes |
| Notificaciones | Pendientes |

---

# 11. Principales riesgos que ahora deben prevenirse

- Configurar una IP sin probar conectividad real desde el host.
- Confundir comprobaciones activas con pasivas.
- Usar un `Hostname` diferente al nombre técnico del host en Zabbix.
- Abrir firewall temporalmente y olvidar persistencia.
- Autorizar una IP equivocada en `Server=`.
- Asumir que un puerto abierto confirma que la integración funciona.
- Modificar plantillas oficiales directamente.
- Agregar una plantilla de aplicación dentro de la plantilla del sistema operativo sin diseño explícito.
- Otorgar privilegios amplios para resolver rápidamente errores Oracle.
- Habilitar métricas Oracle sin revisar licenciamiento.
- Asumir que un servicio `systemd` hereda variables del usuario interactivo.
- Cambiar umbrales antes de entender la métrica.
- Confundir contenedor `running` con aplicación saludable.
- Otorgar permisos excesivos al usuario `zabbix` para Docker o ejecución remota.
- Dar por terminada una configuración sin probar reinicio del servicio o del servidor.

El detalle y las verificaciones concretas se encuentran en [Checklist preventivo Linux](checklist-preventivo-linux.md).

---

# 12. Próximos pasos

- [ ] Ejecutar Zabbix Server en un Oracle Linux de prueba siguiendo la guía `01b`.
- [ ] Ejecutar el checklist preventivo antes de iniciar.
- [ ] Validar base de datos, frontend, SELinux, firewall y reinicio completo.
- [ ] Monitorear el propio Zabbix Server Linux.
- [ ] Incorporar el servidor Linux de aplicaciones con Docker.
- [ ] Inventariar contenedores, puertos, redes, volúmenes y aplicaciones.
- [ ] Aplicar `Linux by Zabbix agent active`.
- [ ] Aplicar `Docker by Zabbix agent 2` después de validar permisos del socket.
- [ ] Crear pruebas de disponibilidad por aplicación.
- [ ] Completar interpretación de métricas.
- [ ] Auditar privilegios Oracle.
- [ ] Completar parametrización de umbrales.
- [ ] Probar notificaciones.
- [ ] Probar una acción remota controlada.
- [ ] Documentar respaldo y restauración de la plataforma Linux.

---

# 13. Criterio para considerar lista la plataforma Linux

No basta con que el frontend abra.

Debe confirmarse:

- Zabbix Server inicia después de reiniciar Linux.
- Base de datos inicia después de reiniciar Linux.
- Frontend inicia después de reiniciar Linux.
- Agent 2 inicia después de reiniciar Linux.
- SELinux permanece habilitado o su excepción está formalmente documentada.
- Firewall es persistente.
- Zona horaria y sincronización son correctas.
- `10051/TCP` está disponible únicamente desde redes autorizadas.
- `10050/TCP` se utiliza solo donde sea necesario.
- El propio Zabbix Server está monitoreado.
- Existe respaldo de base de datos y archivos de configuración.
- Existe procedimiento de restauración.
- Existe procedimiento de actualización.
- Se ha monitoreado exitosamente al menos un servidor Linux, una Oracle Database y un servidor Linux con Docker.

---

# 14. Referencias oficiales

- [Manual actual de Zabbix](https://www.zabbix.com/documentation/current/es/manual)
- [Instalación desde paquetes](https://www.zabbix.com/documentation/7.4/es/manual/installation/install_from_packages)
- [Requisitos de Zabbix 7.4](https://www.zabbix.com/documentation/7.4/es/manual/installation/requirements)
- [Repositorio oficial para Oracle Linux 8](https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/)
- [Comprobaciones activas](https://www.zabbix.com/documentation/current/en/manual/guides/monitor_active)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
- [Integración oficial de Docker](https://www.zabbix.com/integrations/docker)
