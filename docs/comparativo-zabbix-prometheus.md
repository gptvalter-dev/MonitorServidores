# Comparativo histórico: Zabbix vs. Prometheus

> Estado: **decisión inicial cerrada**. Este documento se conserva como antecedente de selección; la plataforma elegida para la fase actual es **Zabbix**.

El objetivo original fue comparar ambas alternativas para monitorear Linux, Docker, Tomcat/Java, Oracle, MongoDB y aplicaciones web.

## 1. Diferencia de enfoque

| Zabbix | Prometheus |
|---|---|
| Plataforma integrada de monitoreo de infraestructura y servicios. | Motor de métricas orientado a series de tiempo y entornos dinámicos. |
| Incluye recolección, almacenamiento, alertas, inventario y frontend. | Habitualmente se complementa con Grafana y Alertmanager. |
| Hosts, plantillas, items y triggers. | Métricas, labels, exporters y PromQL. |

## 2. Cobertura para este proyecto

| Área | Zabbix | Prometheus |
|---|---|---|
| Linux | Agent 2 + plantillas oficiales | Node Exporter |
| Docker | Plugin/plantilla Docker | cAdvisor/métricas del motor |
| Tomcat/JVM | Java Gateway/JMX | JMX Exporter |
| Oracle | Integración oficial | Habitualmente exporter externo |
| MongoDB | Plugin e integración oficial | Habitualmente exporter externo |
| HTTP/HTTPS | Integrado | Blackbox Exporter |
| Alertas | Integradas | Alertmanager |
| Dashboards | Integrados; Grafana opcional | Grafana habitual |
| Métricas de aplicación | Posibles | Fortalezas de PromQL/instrumentación |
| Kubernetes | Adecuado | Muy fuerte |

## 3. Razones para iniciar con Zabbix

Para el alcance actual se priorizaron:

- menor cantidad de componentes que administrar;
- monitoreo centralizado de infraestructura;
- Agent 2 para Linux/Windows;
- integraciones oficiales para Oracle y MongoDB;
- descubrimiento/monitoreo Docker;
- alertas y frontend integrados;
- curva de adopción adecuada para comenzar con infraestructura productiva.

La prueba de concepto confirmó que Zabbix puede cubrir las capas previstas, aunque también mostró que **una plantilla oficial no garantiza compatibilidad perfecta con todas las versiones nuevas de la aplicación**; MongoDB 8.x es el ejemplo documentado en la base de conocimiento.

## 4. Dónde Prometheus sigue siendo relevante

Prometheus continúa siendo una alternativa a evaluar si el proyecto evoluciona principalmente hacia:

- Kubernetes a gran escala;
- métricas instrumentadas directamente desde aplicaciones;
- labels de alta cardinalidad controlada;
- análisis avanzado con PromQL;
- percentiles/SLI/SLO derivados de métricas de aplicación;
- una arquitectura de observabilidad donde Grafana/Alertmanager ya formen parte del estándar.

Esto no obliga a reemplazar Zabbix. Ambos enfoques pueden coexistir si en el futuro existen necesidades claramente diferentes.

## 5. Decisión del proyecto

```text
Fase actual: Zabbix 7.4
Arquitectura objetivo: Zabbix Server sobre Linux
Alcance: Linux + Docker + Oracle + MongoDB + disponibilidad de aplicaciones
```

Por tanto, este comparativo ya no representa una decisión pendiente. Se mantiene para documentar **por qué se inició con Zabbix** y qué condiciones podrían justificar reevaluar Prometheus más adelante.

## 6. Consideraciones que siguen vigentes

- Oracle: revisar siempre privilegios y licenciamiento antes de habilitar métricas avanzadas.
- MongoDB: validar la versión exacta contra los campos que espera la plantilla.
- Docker: `running` no equivale a aplicación saludable.
- Métricas propias de aplicaciones: pueden requerir instrumentación adicional independientemente de la plataforma elegida.

## Referencias

- Zabbix: https://www.zabbix.com/documentation/current/en/
- Zabbix Agent 2: https://www.zabbix.com/documentation/current/en/manual/concepts/agent2
- Prometheus: https://prometheus.io/docs/introduction/overview/
- Prometheus exporters: https://prometheus.io/docs/instrumenting/exporters/
- Alertmanager: https://prometheus.io/docs/alerting/latest/alertmanager/
