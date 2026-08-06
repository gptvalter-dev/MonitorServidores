# Configuración del monitoreo de Oracle Database con Zabbix Agent 2

> Alcance: esta guía parte de un Agent 2 ya instalado y funcionando en Oracle Linux. No incluye interpretación detallada de métricas ni ajuste de umbrales.

## 1. Objetivo

Al finalizar:

- La plantilla `Oracle by Zabbix agent 2` estará vinculada directamente al host.
- Agent 2 podrá cargar las bibliotecas Oracle Client.
- El usuario de monitoreo podrá conectarse al servicio Oracle.
- Las macros estarán configuradas.
- `Oracle Ping` aparecerá como `Up (1)`.

## 2. Requisitos

- Oracle Linux con Agent 2 activo.
- Oracle Database accesible localmente.
- Listener iniciado.
- Usuario con privilegios para crear el usuario de monitoreo.
- Confirmación del licenciamiento disponible.
- Interfaz pasiva del agente accesible en `10050/TCP`.

## 3. Confirmar que Agent 2 funciona

En Oracle Linux:

```bash
sudo systemctl is-active zabbix-agent2
sudo ss -lntp | grep ':10050'
```

En Zabbix:

```text
Monitoreo → Últimos datos → Zabbix agent ping
```

Resultado esperado:

```text
Up (1)
```

## 4. Vincular la plantilla Oracle

En Zabbix:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Plantillas
```

Vincular directamente:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

No agregar la plantilla Oracle dentro de la plantilla Linux y no modificar directamente una plantilla oficial.

## 5. Agregar interfaz pasiva

En el host, agregar una interfaz tipo **Agente**:

```text
IP: <IP_ORACLE_LINUX>
Conectar mediante: IP
Puerto: 10050
Predeterminada: Sí
```

Confirmar que la interfaz pase a estado **Disponible**.

## 6. Confirmar listener y servicio Oracle

Entrar al servidor como usuario Oracle o con un usuario autorizado.

Comprobar listener:

```bash
lsnrctl status
```

Abrir SQL*Plus como administrador:

```bash
sqlplus / as sysdba
```

Consultar:

```sql
SHOW PARAMETER service_names;
SELECT CDB FROM V$DATABASE;
SELECT SYS_CONTEXT('USERENV','CON_NAME') AS CON_NAME FROM DUAL;
```

La macro `{$ORACLE.SERVICE}` debe usar el `SERVICE_NAME` que responde por el listener. No asumir que es igual al SID.

## 7. Crear el usuario de monitoreo

Ejemplo:

```sql
CREATE USER ZABBIX_MON IDENTIFIED BY "<CONTRASENA_SEGURA>";
GRANT CREATE SESSION TO ZABBIX_MON;
```

Usar una contraseña exclusiva y no publicarla en documentación.

## 8. Privilegios utilizados en el laboratorio

```sql
GRANT SELECT ON SYS.V_$INSTANCE TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$DATABASE TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SYSMETRIC TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SYSTEM_PARAMETER TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SESSION TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$RECOVERY_FILE_DEST TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$OSSTAT TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$PROCESS TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$DATAFILE TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$PGASTAT TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$SGASTAT TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$LOG TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$ARCHIVE_DEST TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$ASM_DISKGROUP TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$ASM_DISKGROUP_STAT TO ZABBIX_MON;
GRANT SELECT ON SYS.DBA_USERS TO ZABBIX_MON;
```

Esta lista corresponde al laboratorio y debe auditarse contra la versión exacta de la plantilla antes de producción.

## 9. Restricción de licenciamiento

El ambiente evaluado no cuenta con Oracle Diagnostics Pack. Por ello se revocaron:

```sql
REVOKE SELECT_CATALOG_ROLE FROM ZABBIX_MON;
REVOKE SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY FROM ZABBIX_MON;
```

Antes de producción:

1. Clonar la plantilla Oracle.
2. Identificar elementos que consultan ASH u otras funciones licenciadas.
3. Deshabilitar esos elementos, dependencias, triggers y gráficas en la copia.
4. No modificar la plantilla oficial.

## 10. Configurar macros Oracle

En Zabbix:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Macros
```

Configurar:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = <ORACLE_SERVICE>
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <CONTRASENA_SEGURA>
```

La contraseña debe configurarse como **Texto secreto**.

## 11. Validar credenciales desde Oracle Linux

Ejecutar:

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/<ORACLE_SERVICE>
```

Una conexión exitosa confirma:

- Listener disponible.
- Servicio correcto.
- Usuario válido.
- Contraseña válida.
- Cuenta habilitada.

Salir con:

```sql
EXIT
```

## 12. Localizar las bibliotecas Oracle Client

Buscar:

```bash
find <ORACLE_HOME> -name 'libclntsh.so*' -type f -o -type l
```

Validar permisos:

```bash
ls -l <ORACLE_HOME>/lib/libclntsh.so*
```

Debe existir una biblioteca de 64 bits y, normalmente, un enlace `libclntsh.so` hacia la versión instalada.

## 13. Resolver `DPI-1047`

Síntoma:

```text
DPI-1047: Cannot locate a 64-bit Oracle Client library:
libclntsh.so: cannot open shared object file
```

Causa encontrada: el servicio `zabbix-agent2` iniciado por `systemd` no heredaba el entorno del usuario Oracle.

### Crear el directorio del override

```bash
sudo mkdir -p /etc/systemd/system/zabbix-agent2.service.d
```

### Crear el archivo

Ruta:

```text
/etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Respaldar si ya existe:

```bash
sudo cp -a \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf.respaldo
```

El comando anterior solo aplica cuando el archivo ya existe.

Abrir:

```bash
sudo vi /etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Contenido:

```ini
[Service]
Environment="ORACLE_HOME=<ORACLE_HOME>"
Environment="LD_LIBRARY_PATH=<ORACLE_HOME>/lib"
```

### Aplicar

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
```

### Validar

```bash
sudo systemctl show zabbix-agent2 \
  -p Environment --value | tr ' ' '\n'
```

Resultado esperado:

```text
ORACLE_HOME=<ORACLE_HOME>
LD_LIBRARY_PATH=<ORACLE_HOME>/lib
```

Este cambio afecta solo al servicio Agent 2.

## 14. Validar Oracle Ping

En Zabbix:

```text
Monitoreo → Últimos datos
```

Buscar:

```text
Oracle by Zabbix agent 2: Ping
```

Resultado esperado:

```text
Up (1)
```

Esto confirma funcionalmente:

1. Comunicación pasiva Zabbix Server → Agent 2.
2. Carga de Oracle Client.
3. Listener accesible.
4. Servicio correcto.
5. Autenticación del usuario.
6. Macros correctas.

## 15. Revisar métricas no soportadas

En:

```text
Monitoreo → Últimos datos
```

Revisar la columna de información y los elementos sin valor.

Para cada elemento no soportado registrar:

```text
Nombre del elemento:
Clave:
Error:
Permiso requerido:
Relación con licenciamiento:
Decisión: habilitar, corregir o excluir
```

## 16. Checklist

- [ ] Agent 2 activo.
- [ ] Interfaz pasiva disponible.
- [ ] Plantilla Oracle vinculada directamente.
- [ ] `SERVICE_NAME` identificado.
- [ ] Usuario `ZABBIX_MON` creado.
- [ ] Privilegios revisados.
- [ ] Restricciones de licencia documentadas.
- [ ] Macros configuradas.
- [ ] SQL*Plus directo exitoso.
- [ ] Oracle Client localizado.
- [ ] Override `systemd` configurado si era necesario.
- [ ] `Oracle Ping = Up (1)`.
- [ ] Elementos no soportados revisados.

## 17. Referencias

- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
- [Incidencias de Oracle](../base-conocimiento/oracle.md)
