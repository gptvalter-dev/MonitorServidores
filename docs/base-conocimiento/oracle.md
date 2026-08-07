# Incidencias: monitoreo de Oracle

Este documento conserva los problemas encontrados durante la integración de Oracle Database con Zabbix Agent 2. Cada incidencia incluye el síntoma, la causa, el procedimiento completo de corrección, la validación y el estado.

> Nota de seguridad: este repositorio es público. Las direcciones, nombres de servidor, servicios y contraseñas se representan con valores genéricos.

## Estado funcional alcanzado

- Oracle Linux está registrado como host en Zabbix.
- Las plantillas `Linux by Zabbix agent active` y `Oracle by Zabbix agent 2` están vinculadas directamente al host.
- La interfaz pasiva del agente está disponible en `<IP_ORACLE_LINUX>:10050`.
- Las comprobaciones activas de Linux funcionan.
- La conexión directa con el usuario de monitoreo mediante SQL*Plus funciona.
- La incidencia `DPI-1047` fue resuelta configurando el entorno Oracle del servicio `zabbix-agent2`.
- `Zabbix agent ping` devuelve `Up (1)`.
- `Oracle Ping` devuelve `Up (1)`.
- La base evaluada opera en modo `NOARCHIVELOG`.
- El ambiente no cuenta con Oracle Diagnostics Pack.

---

# 1. La plantilla Oracle se agregó dentro de la plantilla Linux

## Síntoma

Al intentar utilizar `Oracle by Zabbix agent 2`, se presentaron errores de interfaz, dependencias o elementos duplicados. También se observó que cualquier host que utilizara la plantilla Linux podía heredar elementos de Oracle aunque no tuviera una base de datos.

## Causa

La plantilla:

```text
Oracle by Zabbix agent 2
```

se vinculó dentro de:

```text
Linux by Zabbix agent active
```

en lugar de vincularse directamente al host que ejecuta Oracle Database.

El resultado fue una jerarquía incorrecta:

```text
Host Oracle Linux
└── Linux by Zabbix agent active
    └── Oracle by Zabbix agent 2
```

La estructura correcta es:

```text
Host Oracle Linux
├── Linux by Zabbix agent active
└── Oracle by Zabbix agent 2
```

De esta forma, la plantilla del sistema operativo y la plantilla de Oracle permanecen independientes.

## Corrección completa

### Paso 1. Identificar el host que debe monitorear Oracle

En la interfaz web de Zabbix:

1. Abrir **Recopilación de datos**.
2. Entrar a **Equipos**.
3. Localizar el host Oracle Linux.
4. Anotar su **Nombre del equipo**.
5. Abrir el host y confirmar que corresponde al servidor donde se ejecuta Oracle Database.

No realizar el cambio sobre un host distinto.

### Paso 2. Revisar si Oracle está vinculada dentro de la plantilla Linux

1. Ir a **Recopilación de datos → Plantillas**.
2. Buscar:

   ```text
   Linux by Zabbix agent active
   ```

3. Hacer clic en el nombre de la plantilla.
4. Abrir la sección o pestaña **Plantillas**.
5. Revisar el bloque de plantillas vinculadas.
6. Confirmar si aparece:

   ```text
   Oracle by Zabbix agent 2
   ```

Si Oracle aparece dentro de la plantilla Linux, la relación es incorrecta para este diseño.

### Paso 3. Desvincular y limpiar la plantilla Oracle de la plantilla Linux

Dentro de la configuración de `Linux by Zabbix agent active`:

1. Localizar `Oracle by Zabbix agent 2` en las plantillas vinculadas.
2. Elegir **Desvincular y limpiar**.
3. No elegir únicamente **Desvincular**.

La diferencia es importante:

- **Desvincular** elimina la relación, pero conserva los elementos heredados.
- **Desvincular y limpiar** elimina la relación y también retira los elementos, triggers, gráficas y reglas heredadas de Oracle.

4. Presionar **Actualizar** para guardar.
5. Volver a abrir `Linux by Zabbix agent active`.
6. Confirmar que `Oracle by Zabbix agent 2` ya no aparece en la lista de plantillas vinculadas.

> Usar **Desvincular y limpiar** solamente cuando se haya confirmado que la relación con Oracle fue accidental y que no se necesita conservar ninguna entidad heredada.

### Paso 4. Vincular ambas plantillas directamente al host

1. Ir a **Recopilación de datos → Equipos**.
2. Hacer clic en el host Oracle Linux.
3. Abrir la pestaña **Plantillas**.
4. En el campo para vincular plantillas, escribir:

   ```text
   Linux by Zabbix agent active
   ```

5. Seleccionar la coincidencia correcta.
6. En el mismo campo, escribir:

   ```text
   Oracle by Zabbix agent 2
   ```

7. Seleccionar la coincidencia correcta.
8. Confirmar que ambas aparecen en el bloque de plantillas vinculadas.
9. No utilizar la opción **Reemplazar**, porque podría retirar otras plantillas ya vinculadas al host.
10. Presionar **Actualizar**.

### Paso 5. Verificar que la estructura quedó correcta

Volver a abrir el host:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Plantillas
```

Deben aparecer directamente:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

Después revisar nuevamente:

```text
Recopilación de datos → Plantillas → Linux by Zabbix agent active
```

La plantilla Oracle ya no debe aparecer dentro de la plantilla Linux.

### Paso 6. Validar la recopilación de datos

1. Ir a **Monitoreo → Últimos datos**.
2. Filtrar por el host Oracle Linux.
3. Buscar:

   ```text
   Zabbix agent ping
   ```

4. Confirmar:

   ```text
   Up (1)
   ```

5. Buscar:

   ```text
   Oracle Ping
   ```

6. Confirmar:

   ```text
   Up (1)
   ```

7. Revisar que la antigüedad de ambos valores sea reciente.

### Paso 7. Revisar problemas después del cambio

1. Ir a **Monitoreo → Problemas**.
2. Filtrar por el host.
3. Verificar que no existan problemas nuevos relacionados con:

   - claves de elementos duplicadas;
   - dependencias faltantes;
   - elementos heredados de una plantilla incorrecta;
   - ausencia de datos de Linux;
   - ausencia de datos de Oracle.

## Si se eligió solamente “Desvincular”

Cuando se usa **Desvincular** sin limpiar, las entidades heredadas pueden permanecer.

Para corregirlo:

1. Volver a vincular temporalmente `Oracle by Zabbix agent 2` dentro de `Linux by Zabbix agent active`.
2. Guardar con **Actualizar**.
3. Volver a abrir la plantilla Linux.
4. Seleccionar **Desvincular y limpiar** para Oracle.
5. Guardar nuevamente.
6. Validar que los elementos heredados ya no permanezcan.

## Resultado esperado

```text
Host Oracle Linux
├── Linux by Zabbix agent active
└── Oracle by Zabbix agent 2
```

**Estado:** resuelta.

---

# 2. La plantilla Oracle requiere una interfaz pasiva del agente

## Síntoma

La plantilla Oracle estaba vinculada al host, pero `Oracle Ping` no recibía datos o la interfaz del agente aparecía como no disponible.

## Causa

La integración Oracle mediante Zabbix Agent 2 utiliza comprobaciones pasivas. Zabbix Server debe poder conectarse al Agent 2 del servidor Oracle Linux por el puerto `10050/TCP`.

## Corrección completa

### Paso 1. Confirmar que Agent 2 escucha en Oracle Linux

Ejecutar en Oracle Linux:

```bash
sudo systemctl is-active zabbix-agent2
sudo ss -lntp | grep ':10050'
```

Resultado esperado:

```text
active
LISTEN ... *:10050 ... zabbix_agent2
```

### Paso 2. Revisar la configuración del agente

Archivo:

```text
/etc/zabbix/zabbix_agent2.conf
```

Crear respaldo:

```bash
sudo cp -a /etc/zabbix/zabbix_agent2.conf \
  /etc/zabbix/zabbix_agent2.conf.respaldo
```

Abrir:

```bash
sudo vi /etc/zabbix/zabbix_agent2.conf
```

Localizar y configurar:

```ini
Server=<IP_ORIGEN_REAL_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>:<PUERTO_ZABBIX_SERVER>
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

Cuando existan dos orígenes autorizados:

```ini
Server=<IP_ORIGEN_1>,<IP_ORIGEN_2>
```

Guardar y validar:

```bash
sudo /usr/sbin/zabbix_agent2 \
  -c /etc/zabbix/zabbix_agent2.conf -T
```

Reiniciar:

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

### Paso 3. Identificar la IP real desde la que llega la consulta

En Oracle Linux:

```bash
sudo timeout 60 tcpdump -nni any tcp port 10050 -c 5
```

Mientras `tcpdump` está activo, desde Zabbix abrir **Monitoreo → Últimos datos** y actualizar el host.

La dirección origen observada en `tcpdump` debe estar incluida en `Server=`.

### Paso 4. Abrir el firewall de forma permanente

Identificar la zona activa:

```bash
sudo firewall-cmd --get-active-zones
```

Agregar la regla:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="<IP_ORIGEN_REAL>/32" port port="10050" protocol="tcp" accept'
```

Aplicar:

```bash
sudo firewall-cmd --reload
```

Validar:

```bash
sudo firewall-cmd --zone=public --list-rich-rules
```

### Paso 5. Crear o revisar la interfaz del host

En Zabbix:

1. Ir a **Recopilación de datos → Equipos**.
2. Abrir el host Oracle Linux.
3. En **Interfaces**, agregar o revisar una interfaz de tipo **Agente**.
4. Configurar:

   ```text
   IP: <IP_ORACLE_LINUX>
   Conectar mediante: IP
   Puerto: 10050
   Predeterminada: Sí
   ```

5. Presionar **Actualizar**.

### Paso 6. Validar

1. Esperar una nueva comprobación.
2. Confirmar que la interfaz aparezca como **Disponible**.
3. Ir a **Monitoreo → Últimos datos**.
4. Buscar `Oracle Ping`.
5. Confirmar `Up (1)`.

**Estado:** resuelta y persistida.

---

# 3. Se utilizó SID en lugar de SERVICE_NAME

## Síntoma

La conexión de SQL*Plus podía funcionar con una configuración local, pero la plantilla Oracle no lograba conectarse o `Oracle Ping` permanecía en `Down (0)`.

## Causa

La macro `{$ORACLE.SERVICE}` debe contener el servicio publicado por el listener. No debe asumirse que el SID y el `SERVICE_NAME` son iguales.

## Corrección completa

### Paso 1. Consultar los servicios configurados

Entrar a SQL*Plus como usuario autorizado:

```bash
sqlplus / as sysdba
```

Ejecutar:

```sql
SHOW PARAMETER service_names;
SELECT SYS_CONTEXT('USERENV','CON_NAME') AS CON_NAME FROM DUAL;
SELECT CDB FROM V$DATABASE;
```

### Paso 2. Confirmar lo publicado por el listener

Salir de SQL*Plus:

```sql
EXIT
```

Ejecutar:

```bash
lsnrctl status
```

Localizar el nombre del servicio que aparece como disponible.

### Paso 3. Probar el servicio con SQL*Plus

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/<ORACLE_SERVICE>
```

La prueba debe conectarse sin solicitar un identificador diferente.

### Paso 4. Corregir la macro

En Zabbix:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Macros
```

Configurar:

```text
{$ORACLE.SERVICE} = <ORACLE_SERVICE>
```

Presionar **Actualizar**.

### Paso 5. Validar

Ir a **Monitoreo → Últimos datos**, buscar `Oracle Ping` y confirmar:

```text
Up (1)
```

**Estado:** resuelta.

---

# 4. Privilegios incompatibles con el licenciamiento disponible

## Síntoma

La configuración inicial otorgaba privilegios amplios, incluyendo acceso a vistas relacionadas con Active Session History.

## Causa

El ambiente evaluado no cuenta con Oracle Diagnostics Pack. Por ello no debe habilitarse ni utilizarse monitoreo que dependa de ASH sin una revisión de licenciamiento.

## Corrección aplicada

### Paso 1. Entrar como usuario con privilegios

```bash
sqlplus / as sysdba
```

### Paso 2. Revocar privilegios no permitidos para este ambiente

```sql
REVOKE SELECT_CATALOG_ROLE FROM ZABBIX_MON;
REVOKE SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY FROM ZABBIX_MON;
```

### Paso 3. Confirmar los roles del usuario

```sql
SELECT GRANTED_ROLE
FROM DBA_ROLE_PRIVS
WHERE GRANTEE = 'ZABBIX_MON'
ORDER BY GRANTED_ROLE;
```

### Paso 4. Confirmar privilegios directos

```sql
SELECT OWNER, TABLE_NAME, PRIVILEGE
FROM DBA_TAB_PRIVS
WHERE GRANTEE = 'ZABBIX_MON'
ORDER BY OWNER, TABLE_NAME, PRIVILEGE;
```

### Paso 5. Preparar una plantilla personalizada

Antes de producción:

1. Ir a **Recopilación de datos → Plantillas**.
2. Localizar `Oracle by Zabbix agent 2`.
3. Clonar la plantilla oficial.
4. Asignar un nombre que identifique la restricción, por ejemplo:

   ```text
   Oracle by Zabbix agent 2 - Sin Diagnostics Pack
   ```

5. Revisar elementos, elementos dependientes, triggers, gráficas y reglas de descubrimiento.
6. Deshabilitar los componentes que consulten ASH u otra funcionalidad no licenciada.
7. Vincular la copia al host.
8. Desvincular la plantilla oficial solamente después de comprobar que la copia recopila los valores requeridos.

No modificar directamente la plantilla oficial.

**Estado:** privilegios revocados; clonación y auditoría de plantilla pendientes.

---

# 5. Configuración de macros Oracle

## Objetivo

Configurar los parámetros utilizados por la plantilla sin escribir credenciales directamente en cada elemento.

## Procedimiento completo

1. Ir a **Recopilación de datos → Equipos**.
2. Abrir el host Oracle Linux.
3. Entrar a **Macros**.
4. Seleccionar **Macros del equipo**.
5. Agregar o sobrescribir:

   ```text
   {$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
   {$ORACLE.SERVICE}    = <ORACLE_SERVICE>
   {$ORACLE.USER}       = ZABBIX_MON
   {$ORACLE.PASSWORD}   = <CONTRASENA_SEGURA>
   ```

6. Para `{$ORACLE.PASSWORD}`, seleccionar el tipo **Texto secreto**.
7. Confirmar que no existan macros duplicadas con valores diferentes.
8. Presionar **Actualizar**.
9. Esperar una nueva comprobación.
10. Validar `Oracle Ping = Up (1)`.

**Estado:** resuelta.

---

# 6. Validación directa de credenciales

## Objetivo

Separar los problemas de Oracle de los problemas de Zabbix.

## Procedimiento

En Oracle Linux ejecutar:

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/<ORACLE_SERVICE>
```

Ingresar la contraseña cuando se solicite.

Resultado esperado:

```text
Connected to:
Oracle Database 19c ...
```

Ejecutar una prueba mínima:

```sql
SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
```

Salir:

```sql
EXIT
```

Una conexión exitosa confirma:

- listener disponible;
- servicio correcto;
- usuario válido;
- contraseña válida;
- cuenta habilitada;
- privilegio de inicio de sesión.

Si esta prueba falla, corregir Oracle antes de continuar con Zabbix.

**Estado:** validada.

---

# 7. `Oracle Ping` devolvía `Down (0)` por `DPI-1047`

## Síntoma

```text
DPI-1047: Cannot locate a 64-bit Oracle Client library:
libclntsh.so: cannot open shared object file: No such file or directory
```

## Causa

Las bibliotecas Oracle existían en el servidor, pero el servicio `zabbix-agent2`, iniciado por `systemd`, no heredaba `ORACLE_HOME` ni `LD_LIBRARY_PATH` del usuario Oracle.

## Diagnóstico completo

### Paso 1. Confirmar que SQL*Plus funciona

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/<ORACLE_SERVICE>
```

Si SQL*Plus falla, primero resolver credenciales, listener o servicio.

### Paso 2. Localizar `libclntsh.so`

```bash
sudo find / -name 'libclntsh.so*' 2>/dev/null
```

Validar enlaces y permisos:

```bash
ls -l <ORACLE_HOME>/lib/libclntsh.so*
```

Debe existir una biblioteca de 64 bits y normalmente un enlace:

```text
libclntsh.so -> libclntsh.so.<VERSION>
```

### Paso 3. Revisar el entorno del servicio

```bash
sudo systemctl show zabbix-agent2 \
  -p Environment --value | tr ' ' '\n'
```

Si no aparecen `ORACLE_HOME` y `LD_LIBRARY_PATH`, el servicio no conoce la ruta de las bibliotecas.

### Paso 4. Crear un override de systemd

Crear el directorio:

```bash
sudo mkdir -p /etc/systemd/system/zabbix-agent2.service.d
```

Comprobar si ya existe el archivo:

```bash
sudo test -f \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf \
  && echo 'EXISTE' || echo 'NO EXISTE'
```

Si existe, crear respaldo:

```bash
sudo cp -a \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf.respaldo
```

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

Guardar en `vi`:

```text
Esc
:wq
Enter
```

### Paso 5. Validar el archivo de systemd

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Sin salida significa que no se detectó un error de sintaxis.

### Paso 6. Aplicar el cambio

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

Resultado esperado:

```text
active
```

### Paso 7. Confirmar el entorno efectivo

```bash
sudo systemctl show zabbix-agent2 \
  -p Environment --value | tr ' ' '\n'
```

Deben aparecer:

```text
ORACLE_HOME=<ORACLE_HOME>
LD_LIBRARY_PATH=<ORACLE_HOME>/lib
```

### Paso 8. Revisar el log

```bash
sudo tail -n 100 /var/log/zabbix/zabbix_agent2.log
```

También puede utilizarse:

```bash
sudo journalctl -u zabbix-agent2 \
  --since '10 minutes ago' --no-pager
```

El error `DPI-1047` no debe volver a aparecer después del reinicio.

### Paso 9. Validar en Zabbix

1. Ir a **Monitoreo → Últimos datos**.
2. Filtrar por el host.
3. Buscar `Oracle Ping`.
4. Confirmar:

   ```text
   Up (1)
   ```

**Estado:** resuelta.

---

# 8. Alerta de REDO logs disponibles

## Síntoma

Zabbix reportó:

```text
Redo logs available to switch = 0
```

## Diagnóstico realizado

Entrar a SQL*Plus:

```bash
sqlplus / as sysdba
```

Consultar:

```sql
SET LINESIZE 200
COL STATUS FORMAT A12

SELECT GROUP#,
       THREAD#,
       SEQUENCE#,
       ROUND(BYTES/1024/1024) AS MB,
       MEMBERS,
       ARCHIVED,
       STATUS
FROM V$LOG
ORDER BY THREAD#, GROUP#;
```

También confirmar el modo de la base:

```sql
SELECT LOG_MODE FROM V$DATABASE;
```

Resultado observado durante el laboratorio:

```text
1 grupo CURRENT
2 grupos ACTIVE
0 grupos INACTIVE o UNUSED
LOG_MODE = NOARCHIVELOG
```

## Interpretación provisional

Con tres grupos totales y uno siempre en estado `CURRENT`, un umbral que espera tres grupos disponibles no puede cumplirse. Sin embargo, el valor cero también puede indicar que todavía no hay un grupo reutilizable y debe revisarse junto con la frecuencia de cambios de REDO y el avance de checkpoints.

## Siguiente validación pendiente

Consultar cambios de REDO de las últimas 24 horas:

```sql
SET LINESIZE 200
SELECT SEQUENCE#,
       TO_CHAR(FIRST_TIME,'YYYY-MM-DD HH24:MI:SS') AS FECHA_CAMBIO
FROM V$LOG_HISTORY
WHERE FIRST_TIME >= SYSDATE - 1
ORDER BY FIRST_TIME DESC
FETCH FIRST 30 ROWS ONLY;
```

No realizar todavía ninguna de estas acciones únicamente para cerrar la alerta:

- agregar grupos de REDO;
- eliminar grupos;
- cambiar tamaño;
- forzar checkpoints;
- modificar el umbral.

Primero debe documentarse el comportamiento normal de la base.

**Estado:** en análisis.

---

# Validación alcanzada

1. Zabbix Server puede consultar pasivamente Agent 2.
2. Agent 2 puede enviar comprobaciones activas de Linux.
3. Agent 2 puede cargar las bibliotecas Oracle Client.
4. Agent 2 puede conectarse al listener local.
5. El servicio Oracle responde.
6. El usuario de monitoreo puede autenticarse.
7. Las macros se expanden correctamente.
8. `Zabbix agent ping` devuelve `Up (1)`.
9. `Oracle Ping` devuelve `Up (1)`.
10. La regla de firewall permanece después de recargar `firewalld`.

---

# Pendientes

1. Revisar la frecuencia de cambios REDO y definir el umbral apropiado.
2. Clonar la plantilla Oracle para excluir consultas relacionadas con funcionalidades no licenciadas.
3. Auditar los privilegios exactos del usuario de monitoreo contra la revisión de plantilla utilizada.
4. Revisar los elementos Oracle no soportados.
5. Completar los criterios de aceptación y el checklist de producción.

---

# Referencias

- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Vincular y desvincular plantillas](https://www.zabbix.com/documentation/7.4/en/manual/config/templates/linking)
- [Configurar un host](https://www.zabbix.com/documentation/7.4/es/manual/config/hosts/host)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
