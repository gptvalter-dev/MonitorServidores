# MongoDB en Docker con Zabbix Agent 2: lecciones aprendidas y prevención para Linux

## Objetivo

Documentar los hallazgos del laboratorio realizado con Zabbix 7.4.12, Zabbix Agent 2 y MongoDB 8.2.12 en Docker, y convertir los tropiezos encontrados en una lista preventiva para la implementación final sobre Linux.

> **Objetivo final:** el diseño definitivo debe ejecutarse en Linux, con varios contenedores Docker que ejecuten MongoDB y un esquema de monitoreo reproducible, seguro y documentado.

## Ambiente del laboratorio

El laboratorio que permitió obtener estas lecciones utilizó:

- Zabbix Server 7.4.12 ejecutándose en Docker.
- Zabbix Agent 2 7.4.12 instalado en el host Windows.
- Plugin MongoDB para Zabbix Agent 2.
- Docker Desktop/WSL.
- MongoDB 8.2.12 en un contenedor `mongo:8`.
- Plantilla `MongoDB node by Zabbix agent 2`.

Este ambiente fue útil para validar el flujo completo, pero **no debe copiarse literalmente a Linux**. En Linux se eliminan varias particularidades de Docker Desktop/WSL y se deben usar las rutas, permisos, red y servicios propios del sistema operativo.

---

## Arquitectura objetivo en Linux

```text
Zabbix Server / Proxy
        |
        | checks Zabbix
        v
Servidor Linux
├── Zabbix Agent 2
├── Plugin MongoDB
├── Plugin Docker
└── Docker Engine
    ├── mongo01  -> 127.0.0.1:27017
    ├── mongo02  -> 127.0.0.1:27018
    ├── mongo03  -> 127.0.0.1:27019
    └── mongoNN  -> 127.0.0.1:27xxx
```

Criterios:

1. **Zabbix Agent 2 se instala en el host Linux**, no dentro de cada contenedor MongoDB.
2. Cada MongoDB se monitorea de forma independiente.
3. Los datos MongoDB deben residir en volúmenes persistentes del host.
4. Cuando MongoDB solo debe ser consumido localmente, publicar el puerto hacia `127.0.0.1` en lugar de `0.0.0.0`.
5. No depender de IPs internas de contenedores: pueden cambiar al recrearlos.
6. Para varios MongoDB en el mismo servidor, se recomienda representar cada instancia como un **host lógico independiente en Zabbix**, manteniendo un único Agent 2 físico en Linux.

Ejemplo lógico en Zabbix:

```text
LINUX-MONGO-01                 -> sistema operativo / Docker
LINUX-MONGO-01-MONGO01         -> MongoDB puerto 27017
LINUX-MONGO-01-MONGO02         -> MongoDB puerto 27018
LINUX-MONGO-01-MONGO03         -> MongoDB puerto 27019
```

Esto evita mezclar conexiones, alertas y macros de instancias diferentes.

---

# Lecciones aprendidas

## 1. Confirmar siempre dónde se está ejecutando el comando

Uno de los primeros errores del laboratorio fue confundir:

- PowerShell del host.
- shell Linux dentro del contenedor.
- `mongosh` dentro de MongoDB.

Ejemplo de síntoma:

```text
test> notepad ...
SyntaxError
```

`test>` era el prompt de `mongosh`, no PowerShell.

### Prevención

Antes de ejecutar cualquier instrucción documentar explícitamente:

```text
DÓNDE: host Linux
DÓNDE: contenedor MongoDB
DÓNDE: mongosh
DÓNDE: interfaz web Zabbix
```

Nunca asumir el contexto por el comando anterior.

---

## 2. No reinstalar el Agent 2 sin revisar lo existente

El host ya tenía Zabbix Agent 2 7.4.12 instalado y ejecutándose.

La primera validación correcta fue:

```powershell
Get-Service *zabbix*
```

seguida de:

```powershell
zabbix_agent2.exe -V
```

### Prevención en Linux

Antes de instalar o modificar:

```bash
systemctl status zabbix-agent2
zabbix_agent2 -V
rpm -qa | grep -i zabbix
```

Registrar versión de Zabbix Server y Agent 2 y mantenerlos en una rama compatible.

---

## 3. Validar configuración antes de reiniciar el agente

Durante el laboratorio, al reiniciar Agent 2 el servicio dejó de iniciar.

La causa **no era MongoDB**. Un plugin NVIDIA instalado junto con otros plugins generaba:

```text
Cannot register plugins ... NVIDIA ... is not a valid Win32 application
```

La validación del archivo permitió encontrar el problema:

```text
zabbix_agent2 -T
```

### Prevención en Linux

Después de cualquier cambio y **antes de `systemctl restart`**:

```bash
zabbix_agent2 -T
```

Después:

```bash
systemctl restart zabbix-agent2
systemctl status zabbix-agent2 --no-pager
journalctl -u zabbix-agent2 -n 100 --no-pager
```

Instalar solamente los plugins necesarios. Un plugin ajeno al objetivo puede impedir que inicie todo el Agent 2.

---

## 4. No asumir puertos predeterminados

En el laboratorio el Agent 2 usaba:

```text
ListenPort=11050
```

y Zabbix Server Docker publicaba:

```text
11051 -> 10051
```

Inicialmente se intentó consultar el Agent 2 por `10050`, lo cual falló aunque el servicio estaba operativo.

### Prevención

Levantar siempre el inventario real:

```bash
ss -lntp
```

Y documentar:

| Servicio | Puerto real |
|---|---:|
| Zabbix Agent 2 | definir y registrar |
| Zabbix Server | definir y registrar |
| MongoDB 1 | 27017 o asignado |
| MongoDB 2 | 27018 o asignado |
| MongoDB N | único por instancia |

Para Linux se recomienda conservar puertos estándar cuando no exista una razón para cambiarlos.

---

## 5. `127.0.0.1` cambia de significado dentro de Docker

El Zabbix Server estaba en un contenedor. Desde ese contenedor:

```text
127.0.0.1
```

representaba el propio contenedor Zabbix, no Windows.

En el laboratorio Docker Desktop se resolvió usando `host.docker.internal`.

### Prevención para Linux

**No copiar `host.docker.internal` como requisito del diseño Linux.**

En Linux definir explícitamente la ruta de red:

- IP/DNS real del host Linux, o
- red Docker diseñada para el propósito, o
- `host-gateway` solo si deliberadamente se requiere ese patrón.

Probar conectividad desde el Zabbix Server/Proxy real antes de crear la interfaz del host.

---

## 6. `Server=` del Agent 2 controla quién puede hacer checks pasivos

El laboratorio tenía inicialmente:

```text
Server=127.0.0.1
```

Esto debe revisarse en producción. En Linux `Server=` debe permitir únicamente las IP/DNS reales de los Zabbix Server/Proxy autorizados.

### Prevención

No utilizar de forma permanente:

```text
0.0.0.0/0
```

si no es estrictamente necesario.

Permitir solo las fuentes requeridas y validar firewall.

---

## 7. El plugin MongoDB se valida antes de tocar la plantilla

Una prueba crítica fue:

```text
mongodb.ping -> 1
```

Esto demostró que Agent 2 + plugin + credenciales + MongoDB funcionaban antes de diagnosticar la interfaz de Zabbix.

### Prevención

Separar el diagnóstico en capas:

```text
MongoDB responde
   ↓
Plugin MongoDB responde
   ↓
Agent 2 responde
   ↓
Zabbix Server llega al Agent 2
   ↓
Plantilla recibe datos
   ↓
LLD descubre DB/colecciones
   ↓
Triggers generan y recuperan eventos
```

No saltar directamente a editar plantillas si falla una capa inferior.

---

## 8. Crear el usuario de monitoreo en la base correcta

El usuario se creó inicialmente mientras el prompt era:

```text
test>
```

por lo que quedó asociado a `test`.

Se corrigió entrando primero a:

```javascript
use admin
```

### Usuario recomendado para monitoreo

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

### Motivo de los roles

- `clusterMonitor`: métricas de servidor/cluster.
- `readAnyDatabase`: necesario para descubrimiento y estadísticas de bases/colecciones usadas por la plantilla.

### Prevención

Validar siempre:

```text
authenticationDatabase = admin
```

y nunca usar la cuenta `admin` de MongoDB para monitoreo continuo.

---

## 9. `clusterMonitor` solo no fue suficiente para el descubrimiento completo

Con solamente `clusterMonitor`:

- `mongodb.ping` funcionaba.
- `serverStatus` funcionaba.
- `Collection discovery` quedó `No soportada`.

Al agregar `readAnyDatabase`, el descubrimiento pudo ejecutarse.

### Prevención

La prueba de `ping` **no demuestra que LLD tenga todos los permisos requeridos**.

Validar por separado:

```text
mongodb.ping
mongodb.db.discovery
mongodb.collections.discovery
mongodb.collections.usage
```

---

## 10. No confundir "sin datos" con "fallo de descubrimiento"

El MongoDB del laboratorio no tenía bases de aplicación.

`mongodb.db.discovery` devolvía únicamente:

```text
admin
config
local
```

La plantilla las excluye normalmente del descubrimiento de bases de aplicación, por lo que no aparecían métricas como `Size, data`.

### Prevención

Antes de modificar LLD confirmar que realmente existan bases de aplicación.

---

## 11. La plantilla oficial puede no coincidir con MongoDB 8.x

Se encontraron elementos de la plantilla que esperaban campos antiguos de `serverStatus`.

### Métricas de memoria no disponibles en MongoDB 8.2.12

No se encontraron:

```text
mem.mapped
mem.mappedWithJournal
```

La salida observada contenía, entre otros:

```text
mem.bits
mem.resident
mem.virtual
mem.supported
```

Por lo tanto se desactivaron en el laboratorio:

```text
Memory: mapped
Memory: mapped with journal
```

### Prevención

Nunca asumir que una plantilla oficial soporta sin cambios todas las versiones nuevas de MongoDB.

Antes de producción:

1. Capturar `serverStatus()` real de la versión instalada.
2. Comparar JSONPath de cada elemento problemático.
3. No modificar directamente la plantilla del fabricante en producción.
4. Crear una copia, por ejemplo:

```text
MongoDB 8.x by Zabbix agent 2 - Custom
```

Esto evita perder cambios cuando se actualicen las plantillas oficiales.

---

## 12. Cambio confirmado de WiredTiger en MongoDB 8.2.12

La plantilla buscaba:

```text
$.wiredTiger.cache['maximum page size at eviction']
```

MongoDB 8.2.12 entregó:

```text
maximum page size seen at eviction
```

Se corrigió el prototipo a:

```text
$.wiredTiger.cache['maximum page size seen at eviction']
```

y el elemento dejó de aparecer como `No soportada`.

**Estado: validado.**

---

## 13. WiredTiger: `pages evicted by application threads, rate`

La plantilla esperaba:

```text
$.wiredTiger.cache['pages evicted by application threads']
```

Ese campo no apareció en MongoDB 8.2.12.

En la salida real sí apareció:

```text
page evict attempts by application threads
```

La adaptación propuesta es:

```text
$.wiredTiger.cache['page evict attempts by application threads']
```

### Estado

**PENDIENTE DE VALIDACIÓN FINAL.**

Se modificó el prototipo, pero antes de cerrar la incidencia se debe comprobar que:

1. El JSONPath nuevo esté guardado en la **plantilla**, no solo en el host.
2. El cambio se propague al elemento descubierto.
3. Se ejecute nuevamente `Get server status`.
4. El elemento desaparezca de `No soportada`.
5. El valor generado tenga semántica correcta para una métrica `rate`.

Esta es la **última tarea técnica pendiente del laboratorio** antes de continuar con el escenario multi-Mongo en Linux.

---

## 14. Los tickets de WiredTiger no deben usarse ciegamente en MongoDB 8

Los elementos de la plantilla relacionados con:

```text
WiredTiger concurrent transactions: read, available
WiredTiger concurrent transactions: read, out
WiredTiger concurrent transactions: read, total tickets
WiredTiger concurrent transactions: write, available
WiredTiger concurrent transactions: write, out
WiredTiger concurrent transactions: write, total tickets
```

quedaron `No soportada`.

Los iniciadores:

```text
Available WiredTiger read tickets is low
Available WiredTiger write tickets is low
```

quedaron `Desconocido` y fueron desactivados durante el laboratorio.

### Prevención

Para MongoDB 8.x no trasladar estos triggers a producción sin validar la semántica actual de las métricas. Diseñar alertas usando los campos actuales de MongoDB 8.x.

---

## 15. Probar siempre una caída real controlada

Se detuvo el contenedor:

```text
docker stop <mongo_container>
```

Zabbix detectó:

```text
MongoDB node: Connection to MongoDB is unavailable
Severidad: Alta
```

Después de iniciar nuevamente el contenedor, el problema se resolvió automáticamente.

### Lección

Un monitoreo no está terminado cuando aparecen gráficas. Debe probarse:

- creación del evento;
- severidad;
- tiempo de detección;
- recuperación;
- cierre automático.

---

# Checklist preventivo para la implementación final en Linux

## A. Sistema operativo

- [ ] Registrar distribución y versión exacta de Linux.
- [ ] Registrar kernel.
- [ ] Sincronización horaria activa (NTP/chrony).
- [ ] Revisar CPU/RAM disponible para número de MongoDB previstos.
- [ ] Revisar límites de procesos y archivos.
- [ ] Revisar `vm.max_map_count` según la versión exacta de MongoDB.
- [ ] Revisar `vm.swappiness`; MongoDB recomienda minimizar swap y en Linux suele recomendar `0` o `1` según el escenario.
- [ ] Para MongoDB 8.x revisar THP según documentación de esa versión: la recomendación cambió respecto a MongoDB 7 y anteriores.
- [ ] Documentar cualquier perfil `tuned` utilizado en RHEL/Oracle Linux.

## B. Filesystem y almacenamiento

- [ ] Preferir XFS para volúmenes de datos WiredTiger.
- [ ] Verificar que los datos no residan en el filesystem efímero del contenedor.
- [ ] Crear un volumen/directorio persistente por MongoDB.
- [ ] Definir capacidad y crecimiento esperado por instancia.
- [ ] Monitorear espacio, IOPS y latencia del filesystem real del host.
- [ ] Probar backup y restore, no solo backup.

## C. Docker

- [ ] Docker Engine inicia automáticamente.
- [ ] Cada contenedor tiene nombre estable.
- [ ] Cada Mongo tiene volumen persistente propio.
- [ ] Cada Mongo tiene puerto único si se publica al host.
- [ ] Si el acceso solo es local, publicar a `127.0.0.1`, no a todas las interfaces.
- [ ] Definir `restart` policy.
- [ ] No utilizar IP interna efímera del contenedor como dependencia permanente.
- [ ] Registrar versión exacta de la imagen, evitando depender indefinidamente de etiquetas genéricas como `latest`.

## D. Zabbix Agent 2

- [ ] Confirmar versión antes de instalar.
- [ ] Confirmar compatibilidad con Zabbix Server.
- [ ] Instalar únicamente plugins requeridos.
- [ ] Validar configuración con `zabbix_agent2 -T` antes de reiniciar.
- [ ] Confirmar `Server=` con IP/DNS reales de Zabbix Server/Proxy.
- [ ] Confirmar `ServerActive=` si se utilizan checks activos.
- [ ] Confirmar `ListenPort` real.
- [ ] Validar firewall.
- [ ] Revisar logs con `journalctl` ante cualquier fallo.

## E. Plugin Docker en Linux

El plugin Docker de Agent 2 utiliza el socket Unix de Docker, normalmente:

```text
unix:///var/run/docker.sock
```

Prevención:

- [ ] Verificar existencia del socket.
- [ ] Verificar permisos reales del usuario que ejecuta Zabbix Agent 2.
- [ ] No dar acceso al socket sin evaluar seguridad: el acceso al daemon Docker implica privilegios muy elevados.
- [ ] Validar las claves Docker antes de asociar triggers masivos.

## F. MongoDB

- [ ] Autenticación habilitada.
- [ ] Usuario exclusivo `zabbix_monitor`.
- [ ] Usuario creado en `admin`.
- [ ] Roles mínimos requeridos: `clusterMonitor` y, si se requiere descubrimiento de DB/colecciones, `readAnyDatabase`.
- [ ] No usar el usuario administrador en Zabbix.
- [ ] No guardar contraseñas en GitHub.
- [ ] En Zabbix preferir macros secretas para credenciales cuando sea viable.
- [ ] En archivos Linux restringir permisos, por ejemplo `root:zabbix` y modo mínimo requerido.
- [ ] Probar `serverStatus`, descubrimiento de DB y colecciones.

## G. Plantillas Zabbix

- [ ] No modificar directamente la plantilla oficial en producción.
- [ ] Clonar una plantilla específica para MongoDB 8.x.
- [ ] Documentar cada JSONPath cambiado.
- [ ] Desactivar solamente métricas confirmadas como incompatibles.
- [ ] Mantener registro de triggers desactivados y motivo.
- [ ] Después de actualizar Zabbix, comparar la plantilla oficial nueva contra la plantilla custom.

## H. Varios MongoDB en el mismo host

Por cada instancia registrar:

| Campo | Ejemplo |
|---|---|
| Host físico | `LINUX-MONGO-01` |
| Host lógico Zabbix | `LINUX-MONGO-01-MONGO01` |
| Contenedor | `mongo01` |
| Imagen | `mongo:8.x.y` |
| Puerto host | `27017` |
| Puerto contenedor | `27017` |
| Volumen | `/srv/mongodb/mongo01/data` |
| Authentication DB | `admin` |
| Usuario monitoreo | `zabbix_monitor` |
| Plantilla | `MongoDB 8.x by Zabbix agent 2 - Custom` |

Repetir para `mongo02`, `mongo03`, etc.

---

# Comandos de levantamiento recomendados para Linux

Ejecutar antes de modificar el servidor:

```bash
cat /etc/os-release
uname -r
hostname -f
ip addr
ss -lntp

docker version
docker info
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'
docker volume ls

df -Th
lsblk -f

systemctl status zabbix-agent2 --no-pager
zabbix_agent2 -V
zabbix_agent2 -T

sysctl vm.swappiness
sysctl vm.max_map_count
cat /sys/kernel/mm/transparent_hugepage/enabled 2>/dev/null
```

Para cada MongoDB:

```bash
docker inspect <mongo_container>
docker logs --tail 100 <mongo_container>
```

No ejecutar cambios de sysctl, filesystem o THP hasta confirmar la versión exacta de MongoDB y la recomendación correspondiente.

---

# Secuencia preventiva antes de producción

1. Levantar inventario Linux y Docker sin modificar nada.
2. Definir nombres y puertos de todos los MongoDB.
3. Definir almacenamiento persistente y filesystem.
4. Revisar requisitos Linux específicos de la versión MongoDB.
5. Instalar/validar Agent 2.
6. Instalar únicamente plugins MongoDB y Docker requeridos.
7. Validar `zabbix_agent2 -T`.
8. Crear usuario de monitoreo MongoDB.
9. Validar MongoDB localmente.
10. Validar plugin desde Agent 2.
11. Validar conectividad Zabbix Server/Proxy -> Agent 2.
12. Crear host físico Linux en Zabbix.
13. Crear un host lógico por instancia MongoDB.
14. Aplicar plantilla custom MongoDB 8.x.
15. Revisar elementos `No soportada` antes de habilitar alertas.
16. Probar parada/arranque controlado de cada contenedor.
17. Probar crecimiento de disco y umbrales.
18. Probar backup/restore.
19. Documentar evidencia final.

---

# Cosas que NO debemos asumir

- Que el puerto es `10050` porque sea el predeterminado.
- Que `127.0.0.1` representa el host cuando Zabbix está dentro de un contenedor.
- Que `mongodb.ping=1` implica que LLD tiene permisos suficientes.
- Que la plantilla oficial de Zabbix soporta sin cambios MongoDB 8.x.
- Que un trigger `Desconocido` significa que MongoDB está fallando.
- Que una base no visible en Zabbix significa fallo si solo existen `admin`, `config` y `local`.
- Que un plugin ajeno no puede impedir que Agent 2 inicie.
- Que Docker Desktop y Docker Engine en Linux tienen exactamente la misma red.
- Que una IP interna de contenedor permanecerá estable.
- Que el backup es válido sin haber probado restauración.
- Que una recomendación de MongoDB 7.x para THP aplica a MongoDB 8.x.

---

# Pendientes registrados

## Pendiente inmediato del laboratorio

Validar definitivamente el elemento:

```text
WiredTiger cache: pages evicted by application threads, rate
```

con el JSONPath candidato para MongoDB 8.2.12:

```text
$.wiredTiger.cache['page evict attempts by application threads']
```

La tarea solo se considera cerrada cuando el elemento deja de estar `No soportada` y el valor/rate obtenido sea coherente.

## Próxima etapa

Diseñar e implementar el escenario definitivo:

```text
Linux
└── Docker Engine
    ├── MongoDB 1
    ├── MongoDB 2
    ├── MongoDB 3
    └── MongoDB N
```

con:

- un Agent 2 en el host Linux;
- monitoreo Docker por socket Unix;
- un host lógico Zabbix por instancia MongoDB;
- plantilla custom compatible con MongoDB 8.x;
- alertas independientes por contenedor/instancia;
- almacenamiento y backups validados;
- documentación reproducible.

---

## Referencias técnicas

- Zabbix 7.4 Agent 2 UNIX: https://www.zabbix.com/documentation/7.4/en/manual/appendix/config/zabbix_agent2
- Zabbix 7.4 Docker plugin: https://www.zabbix.com/documentation/7.4/en/manual/appendix/config/zabbix_agent2_plugins/d_plugin
- MongoDB 8.0 Production Notes: https://www.mongodb.com/docs/v8.0/administration/production-notes/
- MongoDB 8.0 TCMalloc/THP: https://www.mongodb.com/docs/v8.0/administration/tcmalloc-performance/

## Seguridad documental

No incluir en GitHub:

- contraseñas reales;
- cadenas de conexión con secretos;
- IPs productivas privadas;
- datos de clientes;
- tokens;
- archivos de configuración con credenciales.

Usar marcadores como `<PASSWORD_MONITOREO>`, `<IP_ZABBIX_SERVER>` y `<HOST_LINUX>`.
