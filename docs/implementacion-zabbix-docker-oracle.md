# Índice de implementación y exploración de Zabbix

> Fecha de corte: **6 de agosto de 2026**  
> Estado: **exploración técnica en curso**.

Este archivo es el punto de entrada a la documentación del laboratorio. Los procedimientos se separaron por objetivo para evitar mezclar instalación, configuración, interpretación y parametrización.

El proyecto busca determinar si Zabbix puede utilizarse para monitorear:

1. Sistemas operativos Windows.
2. Sistemas operativos Oracle Linux.
3. Oracle Database.
4. Aplicaciones y servicios.
5. Métricas, alertas y notificaciones.

---

# 1. Orden recomendado

Seguir las guías en este orden:

1. Instalar Zabbix Server en Windows con Docker Desktop.
2. Instalar Agent 2 en Windows.
3. Instalar Agent 2 en Oracle Linux.
4. Configurar el monitoreo de Oracle Database.
5. Aprender a leer las gráficas.
6. Interpretar las métricas principales.
7. Parametrizar métricas y umbrales.
8. Configurar alertas y notificaciones.

No avanzar al siguiente punto cuando la validación final de la guía actual no se haya cumplido.

---

# 2. Guías

| Paso | Guía | Propósito | Estado |
|---:|---|---|---|
| 1 | [Instalación de Zabbix Server en Windows con Docker](guias/01-instalacion-zabbix-windows-docker.md) | Preparar WSL 2, Docker Desktop, MySQL, Zabbix Server y la interfaz web | Validada en laboratorio |
| 2 | [Instalación de Zabbix Agent 2 en Windows](guias/02-instalacion-agente-zabbix-windows.md) | Instalar el agente Windows y validar comprobaciones activas | Validada en laboratorio |
| 3 | [Instalación de Zabbix Agent 2 en Oracle Linux](guias/03-instalacion-agente-zabbix-oracle-linux.md) | Instalar el agente Linux, configurar red, firewall y comprobaciones activas/pasivas | Validada en laboratorio |
| 4 | [Monitoreo de Oracle Database](guias/04-monitoreo-oracle-database.md) | Configurar usuario, macros, Oracle Client y `Oracle Ping` | Funcional; auditoría pendiente |
| 5 | [Interpretación de gráficas](guias/05-interpretacion-graficas-zabbix.md) | Comprender ejes, periodos, leyendas, picos y zona horaria | Iniciada |
| 6 | [Interpretación inicial de métricas](guias/06-interpretacion-metricas-iniciales.md) | Documentar métricas de Linux y Oracle revisadas durante el laboratorio | En desarrollo |
| 7 | [Parametrización de métricas](guias/07-parametrizacion-metricas-zabbix.md) | Ajustar macros, intervalos y crear métricas personalizadas | Pendiente de prueba completa |
| 8 | [Alertas y notificaciones](guias/08-alertas-y-notificaciones-zabbix.md) | Configurar avisos de problema, recuperación y escalamiento | Pendiente de prueba |

---

# 3. Base de conocimiento

Las guías describen el procedimiento normal. La base de conocimiento conserva los errores reales encontrados, su causa y la solución aplicada.

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
Procedimientos normales, validaciones y checklist.

docs/base-conocimiento/
Síntomas, diagnóstico, causa, solución y evidencia de incidencias.
```

---

# 4. Reglas de documentación

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

No se usarán instrucciones aisladas como:

```text
Editar .env
```

En su lugar se documentará la ubicación, apertura, respaldo, cambio, guardado y validación.

---

# 5. Seguridad

Este repositorio es público. Utilizar valores genéricos:

```text
<IP_ZABBIX_SERVER>
<IP_ORIGEN_COMPROBACION_PASIVA>
<IP_ORACLE_LINUX>
<HOSTNAME_WINDOWS>
<HOSTNAME_LINUX>
<ORACLE_SERVICE>
<ORACLE_HOME>
<CONTRASENA_SEGURA>
```

No publicar:

- Contraseñas.
- Direcciones internas reales.
- Nombres reales de servidores productivos.
- Usuarios de aplicación.
- Cadenas de conexión productivas.
- Tokens o secretos.

---

# 6. Arquitectura del laboratorio

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

Flujos principales:

```text
Comprobaciones activas Linux
Oracle Linux ────────────────> Zabbix Server:11051

Comprobaciones pasivas Oracle
Zabbix Server ───────────────> Agent 2:10050 ─────────> Oracle Database
```

---

# 7. Estado actual

| Componente | Estado |
|---|---|
| Docker Desktop y WSL 2 | Correcto para laboratorio |
| Zabbix Server 7.4 en Docker | Operativo |
| Interfaz web | Operativa |
| MySQL de Zabbix | Operativo |
| Agent 2 en Windows | Funcional en laboratorio |
| Agent 2 en Oracle Linux | Instalado, activo y habilitado |
| Comprobaciones activas Linux | `Zabbix agent ping = Up (1)` |
| Interfaz pasiva Linux | Disponible en `10050/TCP` |
| Regla de firewall | Persistente y validada |
| Plantilla Oracle | Vinculada directamente al host |
| Macros Oracle | Configuradas |
| Oracle Client para Agent 2 | Configurado mediante override de `systemd` |
| Oracle Ping | `Up (1)` |
| Métricas Oracle | En recopilación y revisión |
| Interpretación de métricas | Iniciada |
| Métrica personalizada | Pendiente |
| Notificaciones | Pendientes |
| Monitoreo de aplicaciones | Pendiente |

**Avance general estimado del laboratorio: 40%.**

---

# 8. Hallazgos principales

- Docker Compose simplifica el despliegue porque prepara contenedores, red, almacenamiento y dependencias.
- Las variables generales de Compose se encuentran en `.env`.
- Las variables específicas de componentes se encuentran en `env_vars/.env_<componente>`.
- Los cambios locales deben respaldarse antes de ejecutar actualizaciones del repositorio.
- Las comprobaciones activas requieren que `Hostname` coincida exactamente con el nombre técnico del host en Zabbix.
- `ServerActive` debe apuntar a una dirección y puerto realmente accesibles desde el servidor monitoreado.
- Las comprobaciones pasivas requieren autorización tanto en `Server=` como en el firewall.
- La IP real de origen puede ser distinta a la dirección inicialmente esperada.
- La zona horaria del perfil del usuario puede sobrescribir la configuración global del frontend.
- `Oracle Ping = Up (1)` confirma conectividad, servicio, credenciales, macros y Oracle Client.
- Los umbrales de las plantillas son generales y deben validarse contra el ambiente real.
- Las plantillas oficiales no deben modificarse directamente.

---

# 9. Pendientes de la exploración

- [ ] Confirmar métricas recientes de CPU, memoria, discos y red.
- [ ] Completar la matriz de interpretación de métricas.
- [ ] Auditar privilegios exactos de `ZABBIX_MON`.
- [ ] Clonar la plantilla Oracle para excluir funciones no licenciadas.
- [ ] Revisar todos los elementos Oracle no soportados.
- [ ] Crear una métrica personalizada.
- [ ] Validar un trigger personalizado.
- [ ] Configurar una notificación de problema.
- [ ] Configurar una notificación de recuperación.
- [ ] Seleccionar una aplicación representativa.
- [ ] Monitorear disponibilidad, puerto, proceso y endpoint de la aplicación.
- [ ] Documentar diferencias entre laboratorio y producción.
- [ ] Elaborar checklist final de aceptación.

---

# 10. Criterios para concluir el laboratorio

## Sistema operativo

- CPU, memoria, discos y red con valores recientes.
- Comprobaciones activas sin alerta de ausencia de datos.
- Al menos un proceso o servicio relevante monitoreado.

## Oracle Database

- `Oracle Ping = Up (1)`.
- Métricas básicas recibidas.
- Privilegios mínimos auditados.
- Plantilla compatible con el licenciamiento disponible.
- Umbrales principales interpretados.

## Parametrización

- Una macro sobrescrita y documentada.
- Una métrica personalizada funcional.
- Un trigger probado y recuperado.

## Notificaciones

- Aviso de problema recibido.
- Aviso de recuperación recibido.
- Evidencia de tiempos y destinatarios.

## Aplicaciones

- Disponibilidad de una aplicación de prueba.
- Tiempo de respuesta.
- Validación de puerto, proceso o servicio.
- Alerta funcional por indisponibilidad.

---

# 11. Diferencias entre laboratorio y producción

La instalación actual utiliza Windows, Docker Desktop, puertos alternos y un servidor de prueba.

Una implementación productiva deberá evaluar:

- Servidor Linux dedicado para Zabbix.
- Base de datos dimensionada.
- Respaldo y recuperación.
- Alta disponibilidad.
- Zabbix Proxy por ubicación o segmento.
- TLS entre agentes, proxies y servidor.
- Gestión segura de secretos.
- Reglas de firewall definitivas.
- Retención de históricos y tendencias.
- Capacidad de crecimiento.
- Integración con correo, mensajería o mesa de servicio.

---

# 12. Referencias oficiales

- [Manual actual de Zabbix](https://www.zabbix.com/documentation/current/es/manual)
- [Instalación desde contenedores](https://www.zabbix.com/documentation/7.4/es/manual/installation/containers)
- [Actualización desde contenedores](https://www.zabbix.com/documentation/7.4/es/manual/installation/upgrade/containers)
- [Zabbix Agent en Windows](https://www.zabbix.com/documentation/7.4/en/manual/appendix/install/windows_agent)
- [Comprobaciones activas](https://www.zabbix.com/documentation/current/en/manual/guides/monitor_active)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
