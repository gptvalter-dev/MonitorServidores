# Monitoreo de Servidores

Guía inicial para reunir la información necesaria antes de seleccionar una herramienta de monitoreo.

## Objetivo

Elegir el software más adecuado para monitorear principalmente los **servidores productivos Linux**, las aplicaciones desplegadas en **Docker y Tomcat**, y las bases de datos **Oracle y MongoDB**.

> En esta etapa todavía no se elige una herramienta. Primero se recopila la información del entorno y después se comparan las opciones.

---

## Paso 1. Conocer el entorno productivo

Solicitar la cantidad actual de:

- Servidores Linux productivos.
- Contenedores Docker.
- Instancias de Tomcat.
- Bases de datos Oracle.
- Servidores o nodos MongoDB.

También confirmar cómo se administran los contenedores:

- Docker Compose.
- Docker Swarm.
- Kubernetes.
- Ejecución manual.

### Resultado esperado

| Componente | Cantidad | Observaciones |
|---|---:|---|
| Servidores Linux | Pendiente | Solo producción inicialmente |
| Contenedores Docker | Pendiente | Identificar aplicación de cada uno |
| Instancias Tomcat | Pendiente | Confirmar si están dentro de Docker |
| Bases Oracle | Pendiente | Confirmar versión y edición |
| Nodos MongoDB | Pendiente | Confirmar si existe Replica Set |

---

## Paso 2. Confirmar qué se necesita monitorear

### Linux

- CPU, memoria, disco y red.
- Procesos y servicios.

### Docker

- Contenedores activos y detenidos.
- Reinicios, consumo de recursos y health checks.
- Errores y eventos `OOMKilled`.

### Tomcat y Java

- Disponibilidad de Tomcat.
- Heap, Garbage Collector y threads.
- Errores de aplicación.

### Aplicaciones y APIs

- Disponibilidad y tiempo de respuesta.
- Errores HTTP `4xx` y `5xx`.
- Rendimiento de APIs y vencimiento de certificados.

### Oracle

- Disponibilidad de instancia y listener.
- Sesiones, conexiones, tablespaces y bloqueos.
- Uso de CPU y memoria del servidor.

### MongoDB

- Disponibilidad, operaciones y memoria.
- Consultas, índices y estado del Replica Set.

---

## Paso 3. Identificar restricciones técnicas

Confirmar:

- Si se permite instalar agentes en Linux.
- Si se permite consultar Docker y su socket.
- Si se puede habilitar JMX en Tomcat.
- Si se pueden crear usuarios de solo lectura en Oracle y MongoDB.
- Si el monitoreo debe ser local, interno o puede utilizar nube.
- Qué puertos y comunicaciones están permitidos.

---

## Paso 4. Definir la operación

Acordar:

- Responsables de administración y atención de alertas.
- Cobertura `24x7`.
- Medios de notificación.
- Retención de métricas.
- Usuarios de dashboards.
- Ventanas de mantenimiento.

---

## Paso 5. Definir recursos

La solución inicial será **open source y autoadministrada**. Se debe determinar:

- Infraestructura disponible.
- CPU, memoria y almacenamiento.
- Retención, respaldos y recuperación.
- Personal responsable.
- Capacitación y soporte opcional.

> El software puede no tener costo de licencia, pero requiere infraestructura, mantenimiento y tiempo operativo.

---

## Paso 6. Estimar crecimiento

- Servidores esperados en uno o dos años.
- Crecimiento de contenedores y aplicaciones.
- Posible adopción de Kubernetes.
- Necesidad futura de APM, logs y trazas.
- Nuevas sedes o ambientes.

---

## Información mínima para comparar herramientas

- [ ] Cantidad de servidores, contenedores, Tomcat y bases de datos.
- [ ] Método de despliegue de contenedores.
- [ ] Alcance exacto del monitoreo.
- [ ] Restricciones de agentes, accesos y red.
- [ ] Responsables y medios de alerta.
- [ ] Retención requerida.
- [ ] Recursos disponibles.
- [ ] Crecimiento esperado.

---

## Documentación

- [Comparativo: Zabbix vs. Prometheus](docs/comparativo-zabbix-prometheus.md)
- [Implementación de Zabbix con Docker, Windows y Oracle Linux](docs/implementacion-zabbix-docker-oracle.md)
- [Base de conocimiento de incidencias](docs/base-conocimiento/README.md)

## Estado actual

Zabbix 7.4 funciona en Docker y ya monitorea Windows y Oracle Linux. La conexión con Oracle Database sigue pendiente.

## Siguiente paso

Crear el usuario de monitoreo Oracle, configurar las macros del host y validar `oracle.ping`.
