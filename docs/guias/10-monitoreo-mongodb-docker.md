# Monitoreo de MongoDB en Docker con Zabbix Agent 2

> Estado: **procedimiento funcional validado en laboratorio Windows/Docker Desktop; adaptación Linux multi-Mongo pendiente de ejecución completa**.

Esta guía contiene el procedimiento normal. Los errores, incompatibilidades de MongoDB 8.x y soluciones encontradas se documentan en [MongoDB Docker: incidencias y compatibilidad](../base-conocimiento/mongodb-docker-lecciones-linux.md).

## 1. Arquitectura objetivo

Para producción se busca:

```text
Servidor Linux
├── Zabbix Agent 2
├── plugin MongoDB
├── plugin Docker
└── Docker Engine
    ├── mongo01 -> puerto/endpoint estable
    ├── mongo02 -> puerto/endpoint estable
    └── mongoNN -> puerto/endpoint estable
```

Reglas:

- Instalar **un Agent 2 en el host Linux**, no un agente dentro de cada contenedor MongoDB.
- Cada instancia MongoDB debe tener identidad, puerto/endpoint, volumen y alerta independientes.
- No depender de la IP interna de un contenedor recreable.
- En Zabbix, representar cada MongoDB como host lógico independiente cuando se requieran macros/credenciales/conexiones distintas.
- Monitorear por separado el host Linux, Docker y MongoDB.

## 2. Prerrequisitos

Antes de comenzar debe estar validado:

- Agent 2 compatible con la versión del Zabbix Server.
- Conectividad del Zabbix Server/Proxy hacia el Agent 2.
- MongoDB accesible desde el host donde corre Agent 2.
- Plugin MongoDB instalado.
- Usuario dedicado de monitoreo.
- Inventario de contenedores, puertos, volúmenes y redes.

Para Linux, ejecutar primero el [Checklist preventivo Linux](../checklist-preventivo-linux.md).

## 3. Usuario MongoDB de monitoreo

Crear el usuario en `admin`, no en una base de aplicación:

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

Motivo:

- `clusterMonitor`: acceso de lectura para herramientas de monitoreo.
- `readAnyDatabase`: permite listar bases y leer colecciones no-sistema; en el laboratorio fue necesario para descubrimiento/estadísticas de DB y colecciones.

No utilizar la cuenta administrativa para el monitoreo continuo.

## 4. Configuración del plugin MongoDB

Zabbix permite valores predeterminados y sesiones con nombre.

Ejemplo de una sola instancia:

```ini
Plugins.MongoDB.Default.Uri=tcp://127.0.0.1:27017
Plugins.MongoDB.Default.User=zabbix_monitor
Plugins.MongoDB.Default.Password=<MONGODB_PASSWORD>
```

Para varias instancias, preferir sesiones con nombre o macros por host lógico:

```ini
Plugins.MongoDB.Sessions.mongo01.Uri=tcp://127.0.0.1:27017
Plugins.MongoDB.Sessions.mongo01.User=zabbix_monitor
Plugins.MongoDB.Sessions.mongo01.Password=<MONGODB_PASSWORD_01>

Plugins.MongoDB.Sessions.mongo02.Uri=tcp://127.0.0.1:27018
Plugins.MongoDB.Sessions.mongo02.User=zabbix_monitor
Plugins.MongoDB.Sessions.mongo02.Password=<MONGODB_PASSWORD_02>
```

> No almacenar contraseñas reales en GitHub. En producción evaluar macros de tipo secreto y/o mecanismos de secretos disponibles en el entorno.

## 5. Validar Agent 2 antes de reiniciar

Linux:

```bash
sudo zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
sudo journalctl -u zabbix-agent2 -n 100 --no-pager
```

No continuar si la validación de configuración falla.

## 6. Validar MongoDB desde Agent 2

Prueba mínima:

```text
mongodb.ping -> 1
```

Cuando exista `zabbix_get`, probar desde el Zabbix Server/Proxy hacia el Agent 2:

```bash
zabbix_get -s <HOST_AGENT> -p 10050 -k 'mongodb.ping'
```

Para diagnóstico de descubrimiento también son útiles:

```text
mongodb.db.discovery
mongodb.collections.discovery
mongodb.collections.usage
mongodb.server.status
```

## 7. Configuración en Zabbix

Para cada instancia MongoDB:

1. Crear o utilizar un host lógico con interfaz **Agente** apuntando al Agent 2 del servidor Linux.
2. Vincular `MongoDB node by Zabbix agent 2`.
3. Configurar `{$MONGODB.CONNSTRING}` con el endpoint o nombre de sesión correspondiente.
4. Definir `{$MONGODB.USER}` y `{$MONGODB.PASSWORD}` solo si se desea sobrescribir lo configurado en el plugin.
5. Revisar reglas de descubrimiento de bases y colecciones.
6. Revisar elementos `No soportada` antes de habilitar alertas en producción.

Ejemplo conceptual:

```text
LINUX-MONGO-01              -> Linux + Docker
LINUX-MONGO-01-MONGO01      -> sesión/endpoint mongo01
LINUX-MONGO-01-MONGO02      -> sesión/endpoint mongo02
LINUX-MONGO-01-MONGO03      -> sesión/endpoint mongo03
```

## 8. Validaciones obligatorias

Confirmar:

- `MongoDB version` con valor reciente.
- `Ping = Up (1)`.
- conexiones actuales/disponibles.
- operaciones por segundo.
- métricas WiredTiger compatibles.
- descubrimiento de bases cuando existan bases de aplicación.
- descubrimiento de colecciones cuando aplique.
- ausencia de elementos no soportados sin explicación.

Las bases internas `admin`, `config` y `local` pueden ser excluidas por filtros LLD de la plantilla; si no existen bases de aplicación, no debe interpretarse como fallo de descubrimiento.

## 9. Prueba controlada de disponibilidad

En ambiente autorizado:

```bash
docker stop <MONGO_CONTAINER>
```

Zabbix debe generar un evento equivalente a:

```text
MongoDB node: Connection to MongoDB is unavailable
```

Después:

```bash
docker start <MONGO_CONTAINER>
```

Validar cierre automático del problema.

## 10. Compatibilidad MongoDB 8.x

La plantilla `MongoDB node by Zabbix agent 2` de Zabbix 7.4 declara pruebas con MongoDB 4.0.21 y 4.4.3. En el laboratorio con MongoDB 8.2.x se encontraron campos de `serverStatus` que cambiaron o desaparecieron.

Por ello, en producción:

- no modificar directamente la plantilla oficial;
- crear una copia controlada para MongoDB 8.x;
- comparar cada JSONPath no soportado contra la salida real de `serverStatus()`;
- validar la semántica antes de cambiar un nombre de campo;
- no conservar triggers basados en métricas que MongoDB 8.x ya no expone de la misma forma.

Detalles: [MongoDB Docker: incidencias y compatibilidad](../base-conocimiento/mongodb-docker-lecciones-linux.md).

## 11. Consideraciones especiales para varios MongoDB en un mismo Linux

MongoDB/WiredTiger calcula memoria pensando normalmente en una instancia por máquina. Cuando varias instancias comparten host o se ejecutan en contenedores con límites, se debe dimensionar explícitamente memoria y caché por instancia.

Antes de producción documentar por cada contenedor:

```text
Nombre lógico:
Nombre del contenedor:
Imagen/versión:
Puerto/endpoint:
Volumen de datos:
Límite de memoria:
Límite CPU:
Replica Set:
Usuario de monitoreo:
Sesión/macros Zabbix:
Política de reinicio:
Backup:
```

No desplegar N instancias con la configuración de caché predeterminada sin revisar la memoria total del host y los límites del contenedor.

## 12. Referencias oficiales

- Zabbix 7.4 MongoDB integration: https://www.zabbix.com/integrations/mongodb
- Zabbix Agent 2 template operation: https://www.zabbix.com/documentation/7.4/en/manual/config/templates_out_of_the_box/zabbix_agent2
- MongoDB plugin: https://www.zabbix.com/documentation/current/es/manual/appendix/config/zabbix_agent2_plugins/mongodb_plugin
- MongoDB built-in roles: https://www.mongodb.com/docs/manual/reference/built-in-roles/
- MongoDB production notes: https://www.mongodb.com/docs/manual/administration/production-notes/
- WiredTiger: https://www.mongodb.com/docs/v8.2/core/wiredtiger/
