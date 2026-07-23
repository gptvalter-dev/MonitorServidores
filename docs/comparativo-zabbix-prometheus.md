# Comparativo: Zabbix vs. Prometheus

> Comparación inicial para monitorear servidores productivos Linux, contenedores Docker, despliegues Tomcat, Oracle, MongoDB y aplicaciones web.

## 1. Diferencia principal

| Zabbix | Prometheus |
|---|---|
| Plataforma integral de monitoreo de infraestructura y servicios. | Motor de métricas de series de tiempo orientado a aplicaciones y entornos dinámicos. |
| Incluye recolección, almacenamiento, alertas, inventario y dashboards. | Para una solución completa normalmente se complementa con Grafana y Alertmanager. |
| Se organiza mediante hosts, plantillas, elementos y triggers. | Se organiza mediante métricas, etiquetas, exporters y consultas PromQL. |

## 2. Arquitectura típica

### Zabbix

```text
Servidores y aplicaciones
          ↓
    Zabbix Agent 2
          ↓
     Zabbix Server
          ↓
Base de datos + interfaz + alertas
```

### Prometheus

```text
Linux    → Node Exporter
Docker   → cAdvisor
Tomcat   → JMX Exporter
HTTP/API → Blackbox Exporter
Oracle   → Oracle Exporter
MongoDB  → MongoDB Exporter
                  ↓
              Prometheus
                  ↓
       Grafana + Alertmanager
```

## 3. Comparativo general

| Criterio | Zabbix | Prometheus |
|---|---|---|
| Licencia | Open source | Open source |
| Facilidad inicial | Mayor | Menor |
| Componentes por administrar | Menos | Más |
| Administración centralizada | Alta | Distribuida entre varios componentes |
| Servidores Linux | Muy buena | Muy buena |
| Docker | Muy buena | Excelente |
| Tomcat y JVM | Muy buena mediante JMX | Excelente mediante JMX Exporter |
| Oracle | Integración oficial de Zabbix | Normalmente requiere exporter externo |
| MongoDB | Plugin e integración oficial de Zabbix | Normalmente requiere exporter externo |
| HTTP y disponibilidad | Integrado | Blackbox Exporter |
| Métricas propias de aplicaciones | Posible | Excelente |
| Consultas analíticas | Expresiones, históricos y tendencias | Muy flexible mediante PromQL |
| Alertas | Incluidas | Reglas más Alertmanager |
| Dashboards | Incluidos; Grafana es opcional | Interfaz básica; Grafana es habitual |
| Kubernetes | Adecuado | Excelente |
| Curva de aprendizaje | Media | Media-alta o alta |

## 4. Evaluación para el entorno actual

### Linux

Ambos permiten monitorear:

- CPU y carga.
- Memoria y swap.
- Filesystems, espacio e inodos.
- Discos e I/O.
- Red.
- Procesos y servicios.
- Disponibilidad del servidor.

**Diferencia:** Zabbix utiliza principalmente Agent 2 y plantillas. Prometheus utiliza principalmente Node Exporter y consultas PromQL.

### Docker

**Zabbix**

- Descubrimiento de contenedores mediante su integración.
- Estado, reinicios y health checks.
- CPU, memoria, red y almacenamiento.
- Administración desde plantillas.

**Prometheus**

- Normalmente utiliza cAdvisor o métricas del motor de contenedores.
- Modelo de etiquetas apropiado para separar servidor, contenedor, imagen, aplicación y ambiente.
- Mejor adaptación a contenedores dinámicos y crecimiento hacia Kubernetes.

**Ventaja inicial:** Zabbix por facilidad.  
**Ventaja a escala dinámica:** Prometheus.

### Tomcat y JVM

Los dos pueden obtener:

- Heap y non-heap.
- Garbage Collector.
- Threads.
- Sesiones.
- Solicitudes y errores.
- Tiempo de procesamiento.

**Zabbix:** utiliza JMX y Zabbix Java Gateway.  
**Prometheus:** utiliza JMX Exporter.

**Ventaja:** Prometheus cuando se requiere instrumentación detallada de Java; Zabbix cuando se busca monitoreo operativo más integrado.

### Oracle

**Zabbix** dispone de integración y plantillas oficiales para Oracle.

**Prometheus** normalmente necesita un exporter externo, por lo que se debe revisar:

- Mantenimiento del proyecto.
- Compatibilidad con la versión de Oracle.
- Permisos del usuario de monitoreo.
- Seguridad.
- Métricas disponibles.

> Antes de habilitar métricas avanzadas debe confirmarse el licenciamiento de Oracle Diagnostics Pack y Tuning Pack.

**Ventaja inicial:** Zabbix.

### MongoDB

**Zabbix** cuenta con plugins para monitorear nodos y clústeres MongoDB.

**Prometheus** normalmente utiliza un exporter externo.

Ambos pueden cubrir conexiones, operaciones, memoria, WiredTiger y Replica Set, pero en Prometheus la cobertura dependerá del exporter seleccionado.

**Ventaja inicial:** Zabbix por menor riesgo de integración.

### Aplicaciones y APIs

**Zabbix** es adecuado para:

- Disponibilidad HTTP/HTTPS.
- Códigos de respuesta.
- Tiempo de respuesta.
- Certificados.
- Escenarios web.

**Prometheus** destaca cuando la aplicación publica métricas propias, por ejemplo:

- Solicitudes por segundo.
- Latencia P95 y P99.
- Errores por endpoint.
- Connection pools.
- Tiempo consumido en Oracle o MongoDB.
- Métricas funcionales del negocio.

**Ventaja:** Prometheus para observabilidad de aplicaciones.

## 5. Administración

### Zabbix

Componentes principales:

- Zabbix Server.
- Base de datos.
- Interfaz web.
- Zabbix Agent 2.
- Java Gateway cuando se utilice JMX.

La mayor parte de la configuración se concentra en la interfaz web mediante plantillas, hosts, elementos, triggers y acciones.

### Prometheus

Componentes posibles:

- Prometheus.
- Grafana.
- Alertmanager.
- Node Exporter.
- cAdvisor.
- JMX Exporter.
- Blackbox Exporter.
- Oracle Exporter.
- MongoDB Exporter.

La configuración suele distribuirse en archivos YAML, reglas, exporters, dashboards y consultas PromQL.

**Conclusión operativa:** Zabbix normalmente implica menos componentes y menor esfuerzo inicial. Prometheus ofrece mayor flexibilidad, pero requiere más integración y conocimiento especializado.

## 6. Ventajas y desventajas

### Zabbix

**Ventajas**

- Plataforma integrada.
- Menor cantidad de componentes.
- Plantillas oficiales para varias tecnologías del entorno.
- Alertas, escalaciones y ventanas de mantenimiento incluidas.
- Adecuado para comenzar con infraestructura productiva.

**Desventajas**

- Menor flexibilidad que PromQL para análisis dinámicos.
- Menos natural para instrumentar código y métricas por endpoint.
- Los dashboards integrados son menos flexibles que Grafana.

### Prometheus

**Ventajas**

- Excelente para métricas de aplicaciones, Docker y Kubernetes.
- Modelo de etiquetas flexible.
- PromQL permite análisis detallados.
- Muy apropiado para instrumentación Java y APIs.
- Integración nativa con Grafana.

**Desventajas**

- Requiere varios componentes para cubrir todo el entorno.
- Oracle y MongoDB pueden depender de exporters externos.
- Mayor curva de aprendizaje.
- Más esfuerzo de configuración y mantenimiento.

## 7. Resultado preliminar

| Necesidad principal | Alternativa con ventaja |
|---|---|
| Empezar rápidamente | Zabbix |
| Menor esfuerzo operativo | Zabbix |
| Monitorear Oracle y MongoDB | Zabbix |
| Administración centralizada | Zabbix |
| Métricas detalladas de APIs | Prometheus |
| Análisis P95 y P99 | Prometheus |
| Instrumentación de Java | Prometheus |
| Docker dinámico y Kubernetes | Prometheus |
| Consultas avanzadas de métricas | Prometheus |

## 8. Conclusión inicial

Para comenzar con el monitoreo de infraestructura productiva actual, **Zabbix presenta una ventaja por integración y facilidad operativa**.

Prometheus presenta una ventaja cuando el proyecto evoluciona hacia **observabilidad de aplicaciones, métricas personalizadas, Docker a mayor escala o Kubernetes**.

La selección definitiva debe realizarse después de conocer el inventario productivo y ejecutar una prueba de concepto controlada.

## Referencias oficiales

- [Documentación de Zabbix](https://www.zabbix.com/documentation/current/en/)
- [Zabbix Agent 2](https://www.zabbix.com/documentation/current/en/manual/concepts/agent2)
- [Documentación de Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Exporters e integraciones de Prometheus](https://prometheus.io/docs/instrumenting/exporters/)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Grafana con Prometheus](https://grafana.com/docs/grafana/latest/datasources/prometheus/)