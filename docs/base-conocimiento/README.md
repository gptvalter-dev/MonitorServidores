# Base de conocimiento de monitoreo

Esta carpeta conserva **incidencias reales** encontradas durante la implementación: síntoma, causa, diagnóstico, solución, validación y estado.

No debe duplicar los procedimientos normales de `docs/guias/` ni el checklist preventivo general.

## Índice

| Categoría | Contenido |
|---|---|
| [Docker Desktop y Windows](docker-windows.md) | WSL/virtualización, motor Linux y comunicación contenedor ↔ host del laboratorio |
| [Zabbix en Docker Compose](zabbix-docker.md) | Imágenes, secretos CRLF, MySQL y publicación de puertos |
| [Zabbix Agent 2](agentes-zabbix.md) | Arranque, checks activos/pasivos, autorización y plugins que bloquean el agente |
| [Oracle Database](oracle.md) | Plantillas, interfaz, `SERVICE_NAME`, privilegios, Oracle Client y métricas |
| [MongoDB Docker](mongodb-docker-lecciones-linux.md) | Permisos, LLD, compatibilidad MongoDB 8.x, WiredTiger y multi-instancia |

## Dónde documentar cada cosa

```text
docs/guias/
  Procedimiento normal reproducible.

docs/base-conocimiento/
  Incidencias reales y su resolución.

docs/lecciones-aprendidas-y-prevencion-linux.md
  Principios preventivos transversales.

docs/checklist-preventivo-linux.md
  Controles ejecutables antes de desplegar/integrar.
```

## Formato de una incidencia

1. Síntoma o error.
2. Ambiente afectado.
3. Causa identificada.
4. Diagnóstico.
5. Solución.
6. Validación.
7. Estado: resuelta, pendiente o workaround.

## Seguridad

Este repositorio es público. No publicar:

- contraseñas;
- IP internas reales;
- nombres reales de servidores productivos;
- usuarios funcionales de aplicaciones;
- cadenas de conexión con credenciales;
- tokens o secretos.

Usar placeholders como:

```text
<IP_ZABBIX_SERVER>
<IP_HOST>
<HOSTNAME_LINUX>
<MONGODB_PASSWORD>
<ORACLE_SERVICE>
```
