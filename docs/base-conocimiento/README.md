# Base de conocimiento de monitoreo

Esta sección concentra las incidencias, diagnósticos y soluciones encontradas durante la implementación de Zabbix.

La guía de instalación principal describe el procedimiento normal. Esta base de conocimiento se utiliza cuando aparece un error o un comportamiento inesperado.

## Índice de consulta

| Categoría | Contenido |
|---|---|
| [Docker Desktop y Windows](docker-windows.md) | Virtualización, WSL 2, rangos reservados y motor Linux de Docker |
| [Zabbix en Docker Compose](zabbix-docker.md) | Imágenes incorrectas, secretos CRLF, MySQL y puertos de Zabbix Server |
| [Agentes Zabbix](agentes-zabbix.md) | Agent 2 en Windows y Oracle Linux, comprobaciones activas y pasivas |
| [Monitoreo de Oracle](oracle.md) | Plantilla Oracle, interfaz, `SERVICE_NAME`, macros y estado pendiente |

## Forma de documentar nuevas incidencias

Cada incidencia debe contener:

1. Síntoma o mensaje de error.
2. Ambiente afectado.
3. Causa identificada.
4. Diagnóstico realizado.
5. Solución aplicada.
6. Validación posterior.
7. Estado: resuelta, pendiente o solución temporal.

## Seguridad

No publicar en esta carpeta:

- Contraseñas.
- Direcciones IP internas reales.
- Nombres reales de servidores productivos.
- Usuarios funcionales de aplicaciones.
- Cadenas de conexión completas con credenciales.

Utilizar variables como `<IP_ZABBIX_SERVER>`, `<HOSTNAME_LINUX>` y `<ORACLE_SERVICE>`.
