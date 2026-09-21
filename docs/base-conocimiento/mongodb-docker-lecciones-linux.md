# MongoDB Docker: incidencias, compatibilidad y lecciones específicas

> Alcance: hallazgos **específicos de MongoDB + Zabbix Agent 2 + Docker** obtenidos en el laboratorio con Zabbix 7.4.12 y MongoDB 8.2.12.

Los controles generales de Linux, red, Agent 2, firewall y seguridad ya no se repiten aquí. Consultar:

- [Lecciones aprendidas y prevención Linux](../lecciones-aprendidas-y-prevencion-linux.md)
- [Checklist preventivo Linux](../checklist-preventivo-linux.md)
- [Guía normal de monitoreo MongoDB Docker](../guias/10-monitoreo-mongodb-docker.md)

## 1. Ambiente donde se obtuvieron los hallazgos

```text
Windows host
├── Zabbix Agent 2 7.4.12
├── plugin MongoDB
├── Docker Desktop / WSL
└── MongoDB 8.2.12 (mongo:8)

Zabbix Server 7.4.12
└── Docker
```

Este laboratorio sirvió para validar el flujo funcional, pero **no debe copiarse literalmente a Linux**. En Linux no se debe depender de `host.docker.internal`, WSL ni particularidades de puertos reservados de Windows.

## 2. Usuario de monitoreo creado en la base incorrecta

### Síntoma

El usuario `zabbix_monitor` se creó mientras el prompt de `mongosh` estaba en:

```text
test>
```

Por lo tanto, el usuario quedó asociado a `test`.

### Causa

`db.createUser()` crea el usuario en la base seleccionada en ese momento.

### Solución

Seleccionar primero `admin`:

```javascript
use admin

db.createUser({
  user: "zabbix_monitor",
  pwd: passwordPrompt(),
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "readAnyDatabase", db: "admin" }
  ]
})
```

Autenticar con `authenticationDatabase=admin`.

**Estado:** resuelta.

## 3. `clusterMonitor` permitió ping, pero no todo el descubrimiento

### Síntoma

Con solamente `clusterMonitor`:

- `mongodb.ping` funcionaba;
- `serverStatus` funcionaba;
- `Collection discovery` quedó `No soportada`.

### Solución aplicada

Agregar:

```javascript
{ role: "readAnyDatabase", db: "admin" }
```

Después el descubrimiento de colecciones pudo ejecutarse.

### Lección

`mongodb.ping = 1` demuestra conectividad/autenticación, pero **no demuestra que LLD tenga todos los permisos necesarios**.

Validar por separado:

```text
mongodb.ping
mongodb.db.discovery
mongodb.collections.discovery
mongodb.collections.usage
```

**Estado:** resuelta.

## 4. Sin bases de aplicación no aparecen métricas de tamaño por DB

### Síntoma

`Database discovery` estaba activado, pero no aparecían métricas como `Size, data`.

### Diagnóstico

El plugin devolvía únicamente:

```text
admin
config
local
```

La plantilla excluye normalmente esas bases internas mediante filtros LLD.

### Lección

Antes de modificar la regla de descubrimiento, confirmar que realmente existan bases de aplicación.

**Estado:** comportamiento esperado.

## 5. Compatibilidad parcial: plantilla Zabbix 7.4 vs MongoDB 8.2.12

La integración `MongoDB node by Zabbix agent 2` de Zabbix 7.4 declara pruebas con MongoDB 4.0.21 y 4.4.3. En MongoDB 8.2.12 se observaron cambios de `serverStatus()` que afectan varios JSONPath de la plantilla.

### Decisión para producción

No modificar la plantilla oficial directamente. Crear una copia controlada, por ejemplo:

```text
MongoDB 8.x by Zabbix agent 2 - Custom
```

Registrar en esa copia cada adaptación y volver a evaluarla después de actualizar Zabbix o MongoDB.

## 6. Métricas `mem.mapped` y `mem.mappedWithJournal`

### Síntoma

Quedaron `No soportada`:

```text
Memory: mapped
Memory: mapped with journal
```

### Salida observada en MongoDB 8.2.12

```javascript
{
  bits: 64,
  resident: ...,
  virtual: ...,
  supported: true,
  secureAllocByteCount: ...,
  secureAllocBytesInPages: ...
}
```

No estaban presentes los campos esperados por esos dos elementos.

### Acción del laboratorio

Se desactivaron esos elementos.

**Estado:** incompatibilidad confirmada en el laboratorio.

## 7. WiredTiger: `maximum page size at eviction`

### Error

```text
cannot extract value from json by path
"$.wiredTiger.cache['maximum page size at eviction']"
```

### Campo real observado

MongoDB 8.2.12 expuso:

```text
maximum page size seen at eviction
```

### Adaptación validada

Anterior:

```text
$.wiredTiger.cache['maximum page size at eviction']
```

Nuevo:

```text
$.wiredTiger.cache['maximum page size seen at eviction']
```

Después de ejecutar nuevamente `Get server status`, el elemento dejó de aparecer como `No soportada`.

**Estado:** resuelta y validada.

## 8. WiredTiger: `pages evicted by application threads, rate`

### Error

```text
cannot extract value from json by path
"$.wiredTiger.cache.['pages evicted by application threads']"
```

### Campo candidato observado en MongoDB 8.2.12

```text
page evict attempts by application threads
```

MongoDB 8.x documenta actualmente ese campo dentro de `serverStatus().wiredTiger.cache`.

### Adaptación candidata

```text
$.wiredTiger.cache['page evict attempts by application threads']
```

### Importante

El nombre nuevo habla de **intentos de eviction**, no necesariamente de páginas efectivamente desalojadas. Por ello no basta con que el JSONPath deje de fallar: hay que validar que la métrica `rate` y su interpretación sigan siendo correctas.

### Criterios de cierre

- [ ] Confirmar que el JSONPath nuevo quedó guardado en la copia/plantilla correcta.
- [ ] Confirmar propagación al elemento descubierto.
- [ ] Ejecutar nuevamente `Get server status`.
- [ ] Confirmar que el elemento deja de estar `No soportada`.
- [ ] Confirmar el valor obtenido.
- [ ] Definir si el nombre de la métrica debe cambiar de `pages evicted` a `page eviction attempts`.
- [ ] Revisar cualquier trigger o gráfica que use esa serie.

**Estado:** pendiente de validación final.

## 9. Tickets WiredTiger no disponibles como espera la plantilla

Quedaron `No soportada` los elementos relacionados con:

```text
concurrent transactions: read/write available
concurrent transactions: read/write out
concurrent transactions: read/write total tickets
```

Los triggers asociados quedaron `Desconocido` y fueron desactivados en el laboratorio.

MongoDB 7.0+ utiliza ajuste dinámico de tickets y MongoDB 8.x expone información actual de concurrencia en `queues.execution`.

### Decisión

No trasladar esos triggers a producción sin rediseñarlos contra métricas actuales de MongoDB 8.x.

**Estado:** triggers legacy desactivados en laboratorio; rediseño pendiente.

## 10. Prueba controlada de disponibilidad

Se ejecutó:

```text
docker stop <MONGO_CONTAINER>
```

Zabbix generó correctamente:

```text
MongoDB node: Connection to MongoDB is unavailable
Severidad: Alta
```

Después de:

```text
docker start <MONGO_CONTAINER>
```

el evento se resolvió automáticamente.

### Lección

La configuración no se considera validada solo porque haya gráficas. Debe comprobarse generación del problema y recuperación.

**Estado:** validada.

## 11. Diseño objetivo: varios MongoDB en un mismo Linux

```text
Servidor Linux
├── Zabbix Agent 2
├── plugin MongoDB
├── plugin Docker
└── Docker Engine
    ├── mongo01
    ├── mongo02
    ├── mongo03
    └── mongoNN
```

Criterios adoptados:

1. Un Agent 2 en el host Linux.
2. Un endpoint estable por instancia MongoDB.
3. Un volumen persistente independiente por instancia.
4. Un host lógico Zabbix por MongoDB cuando se necesiten conexiones/macros independientes.
5. Usar sesiones con nombre o macros a nivel de host para distinguir instancias.
6. No depender de IP interna de contenedor.
7. Separar monitoreo del host, Docker y MongoDB.
8. Dimensionar memoria/caché por instancia; no asumir que la caché predeterminada es adecuada cuando varias instancias comparten el mismo host.
9. Probar caída de una instancia y confirmar que las demás continúan `Up`.
10. Probar reinicio del host Linux y persistencia de volúmenes/contenedores/Agent 2.

## 12. Riesgos específicos que deben prevenirse en Linux multi-Mongo

- Reutilizar el mismo puerto para dos contenedores.
- Publicar `27017` a `0.0.0.0` sin necesidad.
- Compartir por error el mismo volumen de datos entre instancias.
- Usar la cuenta `admin` para monitoreo.
- Usar una sola identidad Zabbix para varias instancias y mezclar alertas.
- Sobredimensionar WiredTiger cache y agotar RAM del host.
- Ignorar límites de memoria/cgroup de cada contenedor.
- No considerar XFS/EXT4, `vm.swappiness`, THP y `ulimit` antes de producción.
- Aplicar triggers de MongoDB 4.x a MongoDB 8.x sin revisar semántica.
- Hacer LLD de miles de colecciones con intervalos agresivos.
- No probar backup/restore por instancia.

Los controles ejecutables están centralizados en [Checklist preventivo Linux](../checklist-preventivo-linux.md).

## 13. Referencias oficiales

- Zabbix MongoDB integration: https://www.zabbix.com/integrations/mongodb
- Zabbix MongoDB plugin: https://www.zabbix.com/documentation/current/es/manual/appendix/config/zabbix_agent2_plugins/mongodb_plugin
- MongoDB built-in roles: https://www.mongodb.com/docs/manual/reference/built-in-roles/
- MongoDB `serverStatus`: https://www.mongodb.com/docs/manual/reference/command/serverStatus/
- MongoDB WiredTiger 8.2: https://www.mongodb.com/docs/v8.2/core/wiredtiger/
- MongoDB production notes: https://www.mongodb.com/docs/manual/administration/production-notes/
