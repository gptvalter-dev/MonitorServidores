# Checklist preventivo para despliegues Zabbix sobre Linux

> Ejecutar esta revisión antes de instalar Zabbix Server o incorporar un host Linux, Oracle Database, Docker o MongoDB. No marcar un punto por suposición: cada control debe tener evidencia.

## A. Sistema operativo e identidad

- [ ] Distribución y versión exactas documentadas.
- [ ] Kernel documentado.
- [ ] Hostname definitivo.
- [ ] IP fija/reserva y gateway confirmados.
- [ ] DNS revisado cuando aplique.
- [ ] Zona horaria correcta.
- [ ] NTP/chrony sincronizado.
- [ ] CPU, RAM, discos y filesystems inventariados.
- [ ] SELinux en estado conocido.
- [ ] `firewalld` en estado conocido.

```bash
cat /etc/os-release
uname -r
hostnamectl
ip addr
ip route
timedatectl
chronyc tracking
lscpu
free -m
lsblk
df -h
getenforce
firewall-cmd --state
```

## B. Red Zabbix

- [ ] IP/DNS del Zabbix Server o Proxy confirmada desde el host monitoreado.
- [ ] Ruta de red confirmada.
- [ ] `10051/TCP` accesible cuando se usen checks activos.
- [ ] `10050/TCP` accesible desde Server/Proxy cuando se usen checks pasivos.
- [ ] IP real de origen de checks pasivos conocida.
- [ ] NAT/balanceadores documentados.
- [ ] Puertos alternos documentados y justificados.
- [ ] Firewall perimetral y local alineados.

```bash
ip route get <IP_ZABBIX_SERVER>
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/10051'
```

Desde Zabbix Server/Proxy cuando aplique:

```bash
nc -vz <IP_HOST> 10050
```

Si el origen real es dudoso:

```bash
sudo tcpdump -nni <INTERFAZ> tcp port 10050
```

## C. Zabbix Agent 2

- [ ] Versión del Zabbix Server documentada.
- [ ] Agent 2 de rama compatible.
- [ ] Instalación previa revisada antes de instalar otra.
- [ ] Archivo de configuración respaldado.
- [ ] `Hostname` coincide con el nombre técnico del host.
- [ ] `ServerActive=` apunta al Server/Proxy correcto.
- [ ] `Server=` contiene solo orígenes autorizados.
- [ ] `ListenPort` confirmado cuando se usen checks pasivos.
- [ ] Solo están instalados/activos los plugins necesarios.
- [ ] Configuración validada antes de reiniciar.
- [ ] Servicio habilitado al arranque.
- [ ] Logs revisados después del reinicio.

```bash
zabbix_agent2 -V
sudo zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
sudo systemctl enable --now zabbix-agent2
sudo systemctl is-active zabbix-agent2
sudo systemctl is-enabled zabbix-agent2
sudo journalctl -u zabbix-agent2 -n 100 --no-pager
```

No continuar con una integración de aplicación si el agente base todavía falla.

## D. Plantillas y elementos

Antes de vincular una plantilla:

- [ ] Producto y versión reales identificados.
- [ ] Versión de Zabbix compatible.
- [ ] Versiones probadas declaradas por la plantilla revisadas.
- [ ] Tipo de checks (activo/pasivo/dependiente) entendido.
- [ ] Interfaz requerida configurada.
- [ ] Macros heredadas revisadas.
- [ ] Reglas LLD revisadas.
- [ ] Dependencias y permisos identificados.
- [ ] Implicaciones de licenciamiento revisadas.
- [ ] Plantilla oficial conservada sin cambios en producción.
- [ ] Adaptaciones realizadas en una copia controlada.
- [ ] Todo item `No soportada` tiene causa y decisión documentadas.

Estructura preferida:

```text
Host
├── Linux by Zabbix agent active
├── Docker by Zabbix agent 2       (si aplica)
├── Oracle by Zabbix agent 2       (si aplica)
├── MongoDB ...                    (si aplica)
└── Plantillas propias
```

## E. Oracle Database

- [ ] Listener funcionando.
- [ ] `SERVICE_NAME` confirmado.
- [ ] CDB/no-CDB confirmado.
- [ ] Usuario dedicado de monitoreo.
- [ ] Privilegios mínimos revisados.
- [ ] Licenciamiento revisado antes de habilitar métricas.
- [ ] SQL*Plus directo probado.
- [ ] `ORACLE_HOME` confirmado.
- [ ] `libclntsh.so` localizado.
- [ ] Arquitectura de Oracle Client correcta.
- [ ] Entorno efectivo de `zabbix-agent2` revisado.
- [ ] Macros sensibles protegidas.
- [ ] Plantilla Oracle vinculada directamente al host.

```bash
lsnrctl status
sqlplus -L <USUARIO>@//127.0.0.1:1521/<SERVICE_NAME>
find <ORACLE_HOME> -name 'libclntsh.so*'
systemctl show zabbix-agent2 -p Environment
```

Criterio mínimo:

```text
Oracle Ping = Up (1)
```

## F. Docker Engine

- [ ] Docker Engine y versión confirmados.
- [ ] Inventario de contenedores generado.
- [ ] Aplicación/función de cada contenedor identificada.
- [ ] Imagen y versión documentadas.
- [ ] Puertos publicados documentados.
- [ ] Redes Docker documentadas.
- [ ] Volúmenes documentados.
- [ ] Política de reinicio documentada.
- [ ] Límites CPU/RAM documentados.
- [ ] Agent 2 instalado en el host Linux.
- [ ] Acceso de Agent 2 al socket Docker revisado.
- [ ] Contenedores a excluir identificados.
- [ ] Healthcheck/endpoint funcional definido para servicios críticos.

```bash
docker version
docker info
docker ps -a
docker network ls
docker volume ls
ls -l /var/run/docker.sock
id zabbix
```

Regla:

```text
container running != aplicación saludable
```

## G. MongoDB en Docker

Antes de vincular `MongoDB node by Zabbix agent 2`:

### Inventario por instancia

- [ ] Nombre lógico de la instancia.
- [ ] Contenedor e imagen/versión.
- [ ] Endpoint/puerto estable y único.
- [ ] Volumen de datos exclusivo y persistente.
- [ ] Límite de RAM/CPU por contenedor.
- [ ] Replica Set / standalone identificado.
- [ ] Política de reinicio.
- [ ] Estrategia de backup/restore.

### Host Linux para MongoDB

- [ ] Filesystem de datos XFS o EXT4; preferir XFS para WiredTiger cuando sea viable.
- [ ] NTP activo.
- [ ] `vm.swappiness` revisado (`0` o `1` según diseño).
- [ ] THP revisado según versión de MongoDB (MongoDB 8.x tiene recomendaciones distintas a 7.x y anteriores).
- [ ] `ulimit`/open files revisados.
- [ ] `vm.max_map_count` revisado si MongoDB genera advertencia para la versión instalada.
- [ ] RAM total del host suficiente para todas las instancias.
- [ ] Caché WiredTiger dimensionada considerando límites del contenedor y múltiples `mongod`.

### Seguridad y permisos

- [ ] Autenticación MongoDB habilitada.
- [ ] Usuario `zabbix_monitor` creado en `admin`.
- [ ] `clusterMonitor` validado.
- [ ] `readAnyDatabase` evaluado/requerido para LLD y estadísticas.
- [ ] No se usa cuenta `admin` para monitoreo.
- [ ] MongoDB no se publica a `0.0.0.0` sin necesidad.
- [ ] Credenciales no están versionadas en GitHub.

### Plugin/plantilla

- [ ] Plugin MongoDB instalado y versión compatible.
- [ ] Configuración Agent 2 pasa `-T`.
- [ ] `mongodb.ping = 1`.
- [ ] `mongodb.db.discovery` probado.
- [ ] `mongodb.collections.discovery` probado cuando corresponda.
- [ ] `mongodb.collections.usage` probado cuando corresponda.
- [ ] `mongodb.server.status` devuelve datos.
- [ ] Plantilla oficial revisada contra la versión exacta de MongoDB.
- [ ] Para MongoDB 8.x se usa una copia controlada si hay adaptaciones.
- [ ] Triggers legacy de tickets WiredTiger no se usan sin validar semántica actual.

### Varias instancias en un host

- [ ] Cada Mongo tiene puerto/endpoint distinto.
- [ ] Cada Mongo tiene volumen distinto.
- [ ] Cada Mongo tiene identidad Zabbix independiente cuando se requiera.
- [ ] Sesiones con nombre o macros de host claramente separadas.
- [ ] Una caída de `mongo01` no mezcla ni invalida `mongo02`/`mongo03`.
- [ ] Se probó consumo agregado de RAM/CPU/I/O.

Ver procedimiento: [Monitoreo de MongoDB en Docker](guias/10-monitoreo-mongodb-docker.md).

## H. Firewall y SELinux

- [ ] Reglas necesarias son permanentes.
- [ ] No existen aperturas generales innecesarias.
- [ ] Solo se autorizan orígenes requeridos.
- [ ] `firewalld --reload` ejecutado después de cambios.
- [ ] Conectividad revalidada después del reload.
- [ ] SELinux permanece habilitado cuando sea posible.
- [ ] Cualquier excepción SELinux está documentada.

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules
getenforce
```

## I. Métricas y alertas

Antes de modificar un umbral:

- [ ] Nombre exacto de métrica.
- [ ] Unidad.
- [ ] Intervalo.
- [ ] Expresión del trigger.
- [ ] Macro del umbral.
- [ ] Valor normal observado.
- [ ] Pico aislado vs condición sostenida diferenciados.
- [ ] Métricas relacionadas revisadas.
- [ ] Semántica confirmada para la versión actual de la aplicación.
- [ ] Cambio documentado.

No cambiar infraestructura solo para cerrar una alerta genérica.

## J. Seguridad de operación

- [ ] Sin secretos reales en GitHub.
- [ ] Macros sensibles como secreto cuando aplique.
- [ ] Privilegio mínimo.
- [ ] Sin `sudo NOPASSWD: ALL` para `zabbix`.
- [ ] Sin `AllowKey=system.run[*]` general sin justificación.
- [ ] Acceso al socket Docker tratado como privilegio elevado.
- [ ] TLS evaluado para producción.
- [ ] Credenciales predeterminadas sustituidas.

## K. Validación final de cada integración

- [ ] Host visible.
- [ ] Agent 2 con datos recientes.
- [ ] CPU/RAM/filesystems/red con datos recientes.
- [ ] Integración específica con datos.
- [ ] Sin elementos no soportados sin explicación.
- [ ] Problemas activos revisados.
- [ ] Falla controlada probada cuando sea seguro.
- [ ] Recuperación probada.
- [ ] Reinicio del agente probado.
- [ ] Reinicio del host probado cuando corresponda.
- [ ] Persistencia de firewall/servicios/volúmenes confirmada.
- [ ] Backup/restore probado para componentes con datos.
- [ ] Evidencia y cambios documentados.

## L. Secuencia de diagnóstico

```text
1. Sistema operativo
2. IP/ruta
3. Firewall/SELinux
4. Puerto
5. Servicio
6. Agent 2
7. Plugin/permisos
8. Dependencia externa
9. Plantilla/macros
10. Item/preprocesamiento
11. LLD
12. Trigger
```

Hacer un cambio por vez y validar antes del siguiente.
