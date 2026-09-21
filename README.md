# Monitoreo de Servidores

Repositorio de documentación para implementar y operar **Zabbix 7.4** como plataforma de monitoreo de servidores, Docker y bases de datos.

## Objetivo actual

La herramienta seleccionada es **Zabbix**. El laboratorio inicial en Windows + Docker Desktop permitió validar conceptos, conectividad, Agent 2, Oracle y MongoDB; la arquitectura objetivo debe quedar principalmente sobre **Linux/Oracle Linux**.

```text
Zabbix Server Linux
├── Base de datos de Zabbix
├── Zabbix Server
├── Frontend
└── Agent 2

Servidores monitoreados
├── Linux / Oracle Linux
├── Oracle Database
└── Linux + Docker
    ├── aplicaciones
    └── uno o varios MongoDB
```

## Estado resumido

| Componente | Estado |
|---|---|
| Zabbix Server 7.4 en laboratorio Windows/Docker | Operativo |
| Agent 2 Windows | Operativo |
| Agent 2 Oracle Linux | Operativo |
| Oracle Database mediante Agent 2 | Funcional; revisión de métricas/licenciamiento pendiente |
| MongoDB 8.2.x en Docker mediante Agent 2 | Funcional en laboratorio |
| Caída y recuperación de MongoDB | Validada |
| Adaptaciones de plantilla MongoDB 8.x | En revisión |
| Zabbix Server definitivo sobre Linux | Pendiente de ejecución completa |
| Linux con múltiples contenedores MongoDB | Próxima etapa de diseño/implementación |

## Organización de la documentación

- [Índice de implementación](docs/implementacion-zabbix-docker-oracle.md): punto de entrada y estado del proyecto.
- [Guías](docs/guias/): procedimientos normales reproducibles.
- [Base de conocimiento](docs/base-conocimiento/README.md): incidencias reales, causa, solución y validación.
- [Lecciones aprendidas](docs/lecciones-aprendidas-y-prevencion-linux.md): principios preventivos transversales.
- [Checklist preventivo Linux](docs/checklist-preventivo-linux.md): revisión previa obligatoria antes de desplegar o integrar un host.
- [Comparativo Zabbix vs. Prometheus](docs/comparativo-zabbix-prometheus.md): análisis histórico de selección de herramienta.

## Principio operativo

```text
1. Validar sistema operativo y red.
2. Validar Agent 2.
3. Validar la dependencia (Docker, Oracle, MongoDB, HTTP, etc.).
4. Validar la plantilla y sus macros.
5. Confirmar datos reales.
6. Revisar elementos no soportados.
7. Parametrizar alertas.
8. Probar una falla controlada y su recuperación.
9. Documentar el resultado.
```

## Seguridad documental

Este repositorio es público. No publicar contraseñas, IP internas reales, nombres productivos, tokens ni cadenas de conexión con credenciales. Utilizar marcadores como:

```text
<IP_ZABBIX_SERVER>
<HOSTNAME_LINUX>
<MONGODB_USER>
<MONGODB_PASSWORD>
<ORACLE_SERVICE>
```

## Próximo trabajo

1. Cerrar la validación pendiente de la métrica MongoDB 8.x `WiredTiger cache: pages evicted by application threads, rate`.
2. Diseñar y probar el escenario definitivo **Linux + Docker Engine + múltiples contenedores MongoDB + un Agent 2 en el host**.
3. Ejecutar la instalación del Zabbix Server definitivo sobre Linux siguiendo el checklist preventivo.
