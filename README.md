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

Un inventario sencillo como el siguiente:

| Componente | Cantidad | Observaciones |
|---|---:|---|
| Servidores Linux | Pendiente | Solo producción inicialmente |
| Contenedores Docker | Pendiente | Identificar aplicación de cada uno |
| Instancias Tomcat | Pendiente | Confirmar si están dentro de Docker |
| Bases Oracle | Pendiente | Confirmar versión y edición |
| Nodos MongoDB | Pendiente | Confirmar si existe Replica Set |

---

## Paso 2. Confirmar qué se necesita monitorear

Validar que el alcance incluya:

### Linux

- CPU.
- Memoria RAM.
- Disco y espacio disponible.
- Red.
- Procesos y servicios.

### Docker

- Contenedores activos y detenidos.
- Reinicios.
- Consumo de CPU y memoria.
- Health checks.
- Errores y eventos `OOMKilled`.

### Tomcat y Java

- Disponibilidad de Tomcat.
- Uso de memoria JVM.
- Heap y Garbage Collector.
- Threads activos y bloqueados.
- Errores de aplicación.

### Aplicaciones y APIs

- Disponibilidad HTTP/HTTPS.
- Tiempo de respuesta.
- Errores HTTP `4xx` y `5xx`.
- Rendimiento de APIs.
- Vencimiento de certificados.

### Oracle

- Disponibilidad de la instancia y listener.
- Sesiones y conexiones.
- Uso de tablespaces.
- Bloqueos y consultas lentas.
- Uso de CPU y memoria del servidor.

### MongoDB

- Disponibilidad.
- Operaciones por segundo.
- Uso de memoria.
- Consultas e índices.
- Estado y retraso del Replica Set.

---

## Paso 3. Identificar restricciones técnicas

Preguntar al equipo responsable de infraestructura:

- ¿Se permite instalar agentes en los servidores Linux?
- ¿Se permite consultar Docker y su socket?
- ¿Se puede habilitar JMX en Tomcat?
- ¿Se pueden crear usuarios de solo lectura en Oracle y MongoDB?
- ¿La plataforma de monitoreo debe instalarse sobre Linux?
- ¿El monitoreo debe permanecer dentro de la red interna?
- ¿Se permite utilizar una solución en la nube o SaaS?
- ¿Qué puertos y comunicaciones están permitidos?

### Por qué es importante

Estas restricciones pueden descartar herramientas antes de realizar una prueba técnica.

---

## Paso 4. Definir cómo se operará el monitoreo

Acordar:

- Quién administrará la herramienta.
- Quién recibirá las alertas.
- Si el monitoreo será `24x7`.
- Qué medio se utilizará para alertar: correo, Teams, Telegram u otro.
- Cuánto tiempo se conservarán las métricas.
- Cuántos usuarios consultarán los dashboards.
- Cómo se manejarán las ventanas de mantenimiento.

---

## Paso 5. Definir el presupuesto

Determinar:

- Si debe utilizarse software gratuito u open source.
- Si se acepta software comercial.
- Presupuesto anual disponible.
- Si se contratará soporte oficial.
- Si se acepta un costo por servidor, contenedor, métrica o volumen de logs.
- Costo estimado de la infraestructura donde se instalará la plataforma.

> Una herramienta sin costo de licencia también requiere tiempo de instalación, configuración, respaldo y mantenimiento.

---

## Paso 6. Estimar el crecimiento

Preguntar:

- ¿Cuántos servidores se esperan en uno o dos años?
- ¿Aumentará la cantidad de contenedores y aplicaciones?
- ¿Se planea implementar Kubernetes?
- ¿Posteriormente se necesitará APM?
- ¿Se centralizarán logs?
- ¿Se requerirán trazas de aplicaciones?
- ¿Se agregarán más sedes o ambientes?

---

## Información mínima para comparar herramientas

Antes de evaluar Zabbix, Checkmk, Prometheus, Nagios, Datadog u otras opciones, se debe contar al menos con:

- [ ] Cantidad de servidores Linux productivos.
- [ ] Cantidad de contenedores Docker.
- [ ] Cantidad de instancias Tomcat.
- [ ] Cantidad de bases Oracle y MongoDB.
- [ ] Método de despliegue de contenedores.
- [ ] Alcance exacto de monitoreo.
- [ ] Restricciones para instalar agentes y habilitar accesos.
- [ ] Modalidad permitida: local, nube o ambas.
- [ ] Responsables de administración y atención de alertas.
- [ ] Retención requerida para métricas y logs.
- [ ] Presupuesto disponible.
- [ ] Crecimiento esperado.

---

## Siguiente paso

Completar primero el **Paso 1: inventario productivo**. Con esa información se podrá preparar una matriz objetiva de evaluación y descartar las herramientas que no cumplan con el entorno real.
