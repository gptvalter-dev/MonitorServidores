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

También se documentan dos alternativas para instalar Zabbix Server:

```text
Alternativa A: Windows con Docker Desktop
Alternativa B: Oracle Linux con paquetes y servicios systemd
```

---

# 1. Orden recomendado

## Paso 1. Elegir una instalación de Zabbix Server

Utilizar solamente una de las siguientes opciones para el laboratorio:

```text
1A. Zabbix Server en Windows con Docker Desktop.
1B. Zabbix Server en Oracle Linux mediante paquetes oficiales.
```

La opción `1A` es la instalación actualmente validada en el laboratorio.

La opción `1B` está documentada como alternativa más cercana a una instalación Linux tradicional, pero todavía debe ejecutarse y validarse.

Después continuar con:

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
| 1A | [Instalación de Zabbix Server en Windows con Docker](guias/01-instalacion-zabbix-windows-docker.md) | Preparar WSL 2, Docker Desktop, MySQL, Zabbix Server y la interfaz web | Validada en laboratorio |
| 1B | [Instalación de Zabbix Server en Oracle Linux](guias/01b-instalacion-zabbix-server-oracle-linux.md) | Instalar Zabbix Server 7.4, MySQL 8.4, Nginx, PHP-FPM y Agent 2 como servicios Linux | Documentada; pendiente de validación |
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

Cuando se ejecute la instalación del servidor Zabbix en Oracle Linux, las incidencias encontradas deberán registrarse en un archivo independiente dentro de `docs/base-conocimiento/`.

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

Los procedimientos basados en documentación oficial pero todavía no ejecutados deben marcarse como **pendientes de validación**, sin presentarlos como resultados comprobados.

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
<CONTRASENA_ROOT_MYSQL>
<CONTRASENA_BD_ZABBIX>
```

No publicar:

- Contraseñas.
- Direcciones internas reales.
- Nombres reales de servidores productivos.
- Usuarios de aplicación.
- Cadenas de conexión productivas.
- Tokens o secretos.

---

# 6. Arquitecturas documentadas

## 6.1. Arquitectura actualmente validada

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

## 6.2. Alternativa documentada para Oracle Linux

```text
Oracle Linux 8.10
├── MySQL Community Server 8.4
├── Zabbix Server 7.4
├── Zabbix Frontend
├── Nginx
├── PHP-FPM
└── Zabbix Agent 2
```

Servicios principales:

```text
mysqld
zabbix-server
zabbix-agent2
nginx
php-fpm
```

Puertos iniciales de laboratorio:

```text
8080/TCP   Interfaz web
10050/TCP  Zabbix Agent 2
10051/TCP  Zabbix Server
```

MySQL puede permanecer local cuando la base de datos se encuentra en el mismo servidor.

---

# 7. Estado actual

| Componente | Estado |
|---|---|
| Docker Desktop y WSL 2 | Correcto para laboratorio |
| Zabbix Server 7.4 en Docker | Operativo |
| Interfaz web | Operativa |
| MySQL de Zabbix | Operativo |
| Guía de Zabbix Server en Oracle Linux | Creada; pendiente de ejecución y validación |
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

La documentación de una alternativa no aumenta el avance técnico hasta que la instalación haya sido ejecutada y validada.

---

# 8. Hallazgos principales

- Docker Compose simplifica el despliegue porque prepara contenedores, red, almacenamiento y dependencias.
- Las variables generales de Compose se encuentran en `.env`.
- Las variables específicas de componentes se encuentran en `env_vars/.env_<componente>`.
- Los cambios locales deben respaldarse antes de ejecutar actualizaciones del repositorio.
- La instalación directa en Oracle Linux utiliza paquetes y servicios administrados con `systemd`.
- Zabbix no instala automáticamente el motor MySQL; debe instalarse y prepararse antes de importar el esquema.
- Oracle Linux 8 requiere revisar el módulo MySQL para evitar que oculte los paquetes del repositorio oficial de MySQL.
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

- [ ] Ejecutar la instalación de Zabbix Server 7.4 en un Oracle Linux de prueba.
- [ ] Validar MySQL 8.4, Nginx, PHP-FPM, SELinux y firewall de la instalación Linux.
- [ ] Registrar incidencias específicas de Zabbix Server en Oracle Linux.
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

## Alternativa Oracle Linux

Para considerar validada la instalación directa de Zabbix Server en Oracle Linux deben confirmarse:

- MySQL en estado `active`.
- Zabbix Server en estado `active`.
- Nginx en estado `active`.
- PHP-FPM en estado `active`.
- Agent 2 en estado `active`.
- Interfaz web accesible.
- Inicio de sesión correcto.
- Puerto `10051/TCP` escuchando.
- Reinicio del sistema sin pérdida de servicios.

---

# 11. Diferencias entre laboratorio y producción

La instalación actual utiliza Windows, Docker Desktop, puertos alternos y un servidor de prueba.

La alternativa Oracle Linux instala los componentes directamente como servicios del sistema operativo, pero todavía concentra servidor, frontend y base de datos en una sola máquina.

Una implementación productiva deberá evaluar:

- Servidor Linux dedicado para Zabbix.
- Base de datos dimensionada o separada.
- Respaldo y recuperación.
- Alta disponibilidad.
- Zabbix Proxy por ubicación o segmento.
- TLS entre agentes, proxies y servidor.
- HTTPS con certificado válido.
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
- [Instalación desde paquetes](https://www.zabbix.com/documentation/7.4/es/manual/installation/install_from_packages)
- [Requisitos de Zabbix 7.4](https://www.zabbix.com/documentation/7.4/es/manual/installation/requirements)
- [Repositorio oficial para Oracle Linux 8](https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/)
- [MySQL Yum Repository](https://dev.mysql.com/downloads/repo/yum/)
- [Zabbix Agent en Windows](https://www.zabbix.com/documentation/7.4/en/manual/appendix/install/windows_agent)
- [Comprobaciones activas](https://www.zabbix.com/documentation/current/en/manual/guides/monitor_active)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
