# Índice de implementación y exploración de Zabbix

> Actualizado: **20 de septiembre de 2026**  
> Estado: **exploración técnica en curso; arquitectura objetivo Linux**.

Este archivo es el punto de entrada operativo. No repite procedimientos, incidencias ni checklists: enlaza al documento correcto y mantiene el estado del proyecto.

## 1. Arquitectura objetivo

```text
Zabbix Server Linux / Oracle Linux
├── Base de datos de Zabbix
├── Zabbix Server
├── Frontend
└── Zabbix Agent 2

Servidores monitoreados
├── Linux / Oracle Linux
├── Oracle Database
└── Linux + Docker
    ├── aplicaciones
    └── múltiples MongoDB cuando aplique
```

El laboratorio Windows + Docker Desktop se conserva solo como referencia de aprendizaje y diagnóstico. No se deben trasladar automáticamente a Linux soluciones específicas de WSL, `host.docker.internal` o puertos alternos de Windows.

## 2. Orden de trabajo

```text
1. Sistema operativo / infraestructura
2. Red y firewall
3. Agent 2
4. Plantilla del sistema operativo
5. Integración específica (Oracle / Docker / MongoDB / aplicación)
6. Elementos y descubrimiento
7. Interpretación de métricas
8. Umbrales y triggers
9. Falla controlada y recuperación
10. Reinicio/persistencia
11. Documentación
```

No avanzar a la siguiente capa mientras la anterior tenga errores sin explicar.

## 3. Guías

| # | Guía | Propósito | Estado |
|---:|---|---|---|
| 01A | [Zabbix Server en Windows + Docker](guias/01-instalacion-zabbix-windows-docker.md) | Laboratorio Docker Desktop/WSL | Validada como laboratorio |
| 01B | [Zabbix Server en Oracle Linux](guias/01b-instalacion-zabbix-server-oracle-linux.md) | Plataforma central Linux | Documentada; ejecución completa pendiente |
| 02 | [Agent 2 en Windows](guias/02-instalacion-agente-zabbix-windows.md) | Hosts Windows | Validada |
| 03 | [Agent 2 en Oracle Linux](guias/03-instalacion-agente-zabbix-oracle-linux.md) | Hosts Linux | Validada |
| 04 | [Oracle Database](guias/04-monitoreo-oracle-database.md) | Oracle mediante Agent 2 | Funcional; auditoría de métricas/licencia pendiente |
| 05 | [Interpretación de gráficas](guias/05-interpretacion-graficas-zabbix.md) | Lectura de gráficas | En desarrollo |
| 06 | [Interpretación de métricas](guias/06-interpretacion-metricas-iniciales.md) | Métricas iniciales | En desarrollo |
| 07 | [Parametrización](guias/07-parametrizacion-metricas-zabbix.md) | Macros, intervalos, métricas | Pendiente de prueba completa |
| 08 | [Alertas y notificaciones](guias/08-alertas-y-notificaciones-zabbix.md) | Eventos y avisos | Pendiente de prueba |
| 09 | [Ejecución remota y actualizaciones](guias/09-ejecucion-remota-y-actualizaciones.md) | Acciones remotas controladas | Documentada; laboratorio pendiente |
| 10 | [MongoDB en Docker](guias/10-monitoreo-mongodb-docker.md) | Agent 2 + plugin MongoDB + Docker + multi-instancia | Laboratorio validado; Linux multi-Mongo pendiente |

## 4. Documentos transversales

- [Lecciones aprendidas](lecciones-aprendidas-y-prevencion-linux.md): principios generales; evita repetir errores.
- [Checklist preventivo Linux](checklist-preventivo-linux.md): controles previos y criterios de aceptación.
- [Base de conocimiento](base-conocimiento/README.md): errores reales y su solución.
- [Comparativo Zabbix vs Prometheus](comparativo-zabbix-prometheus.md): antecedente de selección de herramienta.

## 5. Estado funcional

| Componente | Estado |
|---|---|
| Zabbix Server 7.4 Windows/Docker | Operativo en laboratorio |
| Frontend y DB de Zabbix del laboratorio | Operativos |
| Agent 2 Windows | Operativo |
| Agent 2 Oracle Linux | Operativo |
| Checks activos/pasivos Linux | Validados |
| Oracle Ping | `Up (1)` |
| Métricas Oracle | En revisión |
| MongoDB 8.2.x en Docker | Monitoreo funcional |
| MongoDB: caída y recuperación | Validada |
| MongoDB: DB/collection discovery | Validado con permisos requeridos |
| MongoDB: compatibilidad WiredTiger 8.x | Parcialmente adaptada; un item pendiente |
| Zabbix Server definitivo Linux | Pendiente |
| Linux + varios MongoDB Docker | Pendiente; próxima etapa |
| Notificaciones | Pendientes |

## 6. Pendiente inmediato

Cerrar la validación de:

```text
WiredTiger cache: pages evicted by application threads, rate
```

La plantilla busca un campo antiguo. MongoDB 8.2.x expone:

```text
page evict attempts by application threads
```

No basta con corregir JSONPath: debe confirmarse si la serie sigue representando la misma semántica o debe renombrarse como **eviction attempts rate**.

Seguimiento: Issue #1 del repositorio.

## 7. Próxima etapa: Linux multi-Mongo

Objetivo:

```text
Linux
├── Zabbix Agent 2
├── plugin Docker
├── plugin MongoDB
└── Docker Engine
    ├── mongo01
    ├── mongo02
    ├── mongo03
    └── mongoNN
```

Antes de configurar Zabbix se deberá completar el inventario de:

- puertos/endpoints;
- redes Docker;
- volúmenes;
- memoria/CPU por contenedor;
- política de reinicio;
- Replica Set si existe;
- backup/restore;
- usuarios de monitoreo;
- estrategia de hosts lógicos y sesiones/macros.

La implementación debe pasar el [Checklist preventivo Linux](checklist-preventivo-linux.md), sección MongoDB.

## 8. Reglas de documentación

Todo procedimiento que cambie configuración debe indicar:

1. dónde se ejecuta;
2. permisos requeridos;
3. archivo/ruta afectada;
4. respaldo previo;
5. cambio exacto;
6. validación antes de reiniciar;
7. aplicación del cambio;
8. resultado esperado;
9. diagnóstico si falla;
10. reversión cuando aplique.

Los pasos tomados de documentación oficial pero todavía no ejecutados deben marcarse **pendientes de validación**.

## 9. Seguridad del repositorio

El repositorio es público. Solo usar placeholders:

```text
<IP_ZABBIX_SERVER>
<IP_HOST>
<HOSTNAME_LINUX>
<MONGODB_PASSWORD>
<ORACLE_SERVICE>
<ORACLE_HOME>
```

No versionar credenciales, IPs internas reales, nombres productivos, tokens ni secretos.

## 10. Criterio de aceptación de la plataforma Linux

La plataforma central no estará lista solo porque el frontend abra. Debe confirmarse:

- servicios habilitados y operativos después de reiniciar Linux;
- firewall persistente;
- SELinux controlado y documentado;
- hora/NTP correctos;
- Zabbix Server y Agent 2 monitoreados;
- backup y restore documentados/probados;
- al menos un Linux, una Oracle DB y un Linux con Docker monitoreados;
- integraciones sin elementos no soportados sin explicación;
- alertas y recuperación probadas.
