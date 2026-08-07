# Configuración del monitoreo de Oracle Database con Zabbix Agent 2

> Alcance: esta guía parte de un Agent 2 ya instalado y funcionando en Oracle Linux. No incluye interpretación detallada de métricas ni ajuste de umbrales.

## 1. Objetivo

Al finalizar:

- La plantilla `Oracle by Zabbix agent 2` estará vinculada directamente al host.
- La plantilla Oracle no estará agregada dentro de la plantilla Linux.
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
- Acceso a la interfaz web de Zabbix con permisos para modificar hosts y plantillas.

## 3. Confirmar que Agent 2 funciona

Este paso se ejecuta en Oracle Linux.

```bash
sudo systemctl is-active zabbix-agent2
sudo ss -lntp | grep ':10050'
```

Resultado esperado:

```text
active
LISTEN ... *:10050 ... zabbix_agent2
```

En Zabbix:

```text
Monitoreo → Últimos datos → Zabbix agent ping
```

Resultado esperado:

```text
Up (1)
```

No continuar si el servicio está detenido o si el agente no escucha en `10050/TCP`.

## 4. Vincular correctamente las plantillas

Las dos plantillas deben vincularse directamente al host Oracle Linux:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

La estructura esperada es:

```text
Host Oracle Linux
├── Linux by Zabbix agent active
└── Oracle by Zabbix agent 2
```

No debe quedar así:

```text
Host Oracle Linux
└── Linux by Zabbix agent active
    └── Oracle by Zabbix agent 2
```

### 4.1. Identificar el host correcto

1. Abrir la interfaz web de Zabbix.
2. Ir a **Recopilación de datos → Equipos**.
3. Localizar el servidor Oracle Linux.
4. Confirmar el valor de **Nombre del equipo**.
5. Abrir el host y comprobar que la IP corresponde al servidor donde se ejecuta Oracle Database.

No aplicar el cambio sobre otro host.

### 4.2. Revisar si la plantilla Oracle está dentro de la plantilla Linux

1. Ir a **Recopilación de datos → Plantillas**.
2. Buscar:

   ```text
   Linux by Zabbix agent active
   ```

3. Hacer clic en el nombre de la plantilla.
4. Abrir la sección **Plantillas**.
5. Revisar las plantillas vinculadas.
6. Buscar:

   ```text
   Oracle by Zabbix agent 2
   ```

Si aparece en esa sección, la plantilla Oracle fue agregada dentro de la plantilla Linux y debe retirarse.

### 4.3. Retirar la relación incorrecta

Dentro de `Linux by Zabbix agent active`:

1. Localizar `Oracle by Zabbix agent 2`.
2. Seleccionar **Desvincular y limpiar**.
3. No seleccionar únicamente **Desvincular**.
4. Presionar **Actualizar**.
5. Volver a abrir la plantilla Linux.
6. Confirmar que Oracle ya no aparece como plantilla vinculada.

Diferencia entre las opciones:

| Opción | Resultado |
|---|---|
| **Desvincular** | Elimina la relación, pero conserva las entidades heredadas |
| **Desvincular y limpiar** | Elimina la relación y también los elementos, triggers, gráficas y reglas heredadas |

Usar **Desvincular y limpiar** solamente después de confirmar que la relación fue accidental.

### 4.4. Vincular la plantilla Linux directamente al host

1. Ir a **Recopilación de datos → Equipos**.
2. Abrir el host Oracle Linux.
3. Entrar a la pestaña **Plantillas**.
4. En el campo para agregar plantillas, escribir:

   ```text
   Linux by Zabbix agent active
   ```

5. Seleccionar la coincidencia correcta.
6. Confirmar que aparece en el bloque de plantillas vinculadas.

Si ya estaba vinculada directamente, no agregarla otra vez.

### 4.5. Vincular la plantilla Oracle directamente al host

En la misma pestaña **Plantillas**:

1. Escribir:

   ```text
   Oracle by Zabbix agent 2
   ```

2. Seleccionar la coincidencia correcta.
3. Confirmar que aparece junto con la plantilla Linux.
4. No utilizar **Reemplazar**, porque esa opción puede retirar otras plantillas ya vinculadas.
5. Presionar **Actualizar**.

### 4.6. Verificar la relación final

Volver a abrir:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Plantillas
```

Deben aparecer directamente:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

Después abrir:

```text
Recopilación de datos → Plantillas → Linux by Zabbix agent active
```

La plantilla Oracle no debe aparecer dentro de la plantilla Linux.

### 4.7. Validar los datos heredados

1. Ir a **Monitoreo → Últimos datos**.
2. Seleccionar el host Oracle Linux.
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

6. Si todavía no tiene valor, continuar con la configuración de interfaz, macros y Oracle Client descrita en los siguientes pasos.

### 4.8. Revisar errores de plantillas

Ir a:

```text
Monitoreo → Problemas
```

Filtrar por el host y comprobar que no existan errores relacionados con:

- claves de elementos duplicadas;
- dependencias faltantes;
- elementos Oracle heredados por la plantilla Linux;
- ausencia de datos del agente;
- ausencia de datos de Oracle.

### 4.9. Corrección cuando se utilizó solamente “Desvincular”

Si se retiró Oracle de la plantilla Linux utilizando solo **Desvincular**, sus entidades pueden permanecer.

Para limpiarlas:

1. Volver a vincular temporalmente `Oracle by Zabbix agent 2` dentro de `Linux by Zabbix agent active`.
2. Presionar **Actualizar**.
3. Volver a abrir la plantilla Linux.
4. Seleccionar **Desvincular y limpiar**.
5. Presionar **Actualizar**.
6. Confirmar que las entidades heredadas ya no permanecen.

Consulta detallada de la incidencia:

```text
../base-conocimiento/oracle.md
```

## 5. Agregar interfaz pasiva

En Zabbix:

1. Ir a **Recopilación de datos → Equipos**.
2. Abrir el host Oracle Linux.
3. Localizar el bloque **Interfaces**.
4. Agregar una interfaz de tipo **Agente**.
5. Configurar:

   ```text
   IP: <IP_ORACLE_LINUX>
   Conectar mediante: IP
   Puerto: 10050
   Predeterminada: Sí
   ```

6. Presionar **Actualizar**.
7. Esperar una nueva comprobación.
8. Confirmar que la interfaz pase a estado **Disponible**.

Si permanece no disponible, revisar:

- que Agent 2 esté activo;
- que escuche en `10050/TCP`;
- que la IP real del Zabbix Server esté incluida en `Server=`;
- que `firewalld` permita el origen real;
- que exista ruta entre Zabbix Server y Oracle Linux.

## 6. Confirmar listener y servicio Oracle

Este paso se ejecuta en Oracle Linux como usuario Oracle o con un usuario autorizado.

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

Salir:

```sql
EXIT
```

## 7. Crear el usuario de monitoreo

Entrar a SQL*Plus como usuario con privilegios:

```bash
sqlplus / as sysdba
```

Crear el usuario:

```sql
CREATE USER ZABBIX_MON IDENTIFIED BY "<CONTRASENA_SEGURA>";
GRANT CREATE SESSION TO ZABBIX_MON;
```

Usar una contraseña exclusiva y no publicarla en documentación.

Confirmar el estado:

```sql
SELECT USERNAME, ACCOUNT_STATUS
FROM DBA_USERS
WHERE USERNAME = 'ZABBIX_MON';
```

Resultado esperado:

```text
ZABBIX_MON  OPEN
```

## 8. Privilegios utilizados en el laboratorio

Ejecutar como usuario con privilegios:

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

Confirmar los privilegios:

```sql
SELECT OWNER, TABLE_NAME, PRIVILEGE
FROM DBA_TAB_PRIVS
WHERE GRANTEE = 'ZABBIX_MON'
ORDER BY OWNER, TABLE_NAME;
```

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
4. Vincular la copia al host de prueba.
5. Validar la recopilación.
6. Desvincular la plantilla oficial únicamente cuando la copia funcione.
7. No modificar directamente la plantilla oficial.

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

Procedimiento:

1. Abrir **Macros**.
2. Seleccionar **Macros del equipo**.
3. Agregar cada macro una sola vez.
4. Revisar que no existan valores diferentes heredados o duplicados.
5. Marcar la contraseña como **Texto secreto**.
6. Presionar **Actualizar**.

## 11. Validar credenciales desde Oracle Linux

Ejecutar:

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/<ORACLE_SERVICE>
```

Una conexión exitosa confirma:

- listener disponible;
- servicio correcto;
- usuario válido;
- contraseña válida;
- cuenta habilitada.

Ejecutar una prueba mínima:

```sql
SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
```

Salir con:

```sql
EXIT
```

Si esta prueba falla, corregir Oracle antes de revisar Zabbix.

## 12. Localizar las bibliotecas Oracle Client

Buscar:

```bash
sudo find / -name 'libclntsh.so*' 2>/dev/null
```

Validar permisos y enlaces:

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

### 13.1. Crear el directorio del override

```bash
sudo mkdir -p /etc/systemd/system/zabbix-agent2.service.d
```

### 13.2. Comprobar si el archivo existe

Ruta:

```text
/etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Ejecutar:

```bash
sudo test -f \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf \
  && echo 'EXISTE' || echo 'NO EXISTE'
```

### 13.3. Respaldar cuando ya existe

```bash
sudo cp -a \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf.respaldo
```

### 13.4. Abrir el archivo

```bash
sudo vi /etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Contenido:

```ini
[Service]
Environment="ORACLE_HOME=<ORACLE_HOME>"
Environment="LD_LIBRARY_PATH=<ORACLE_HOME>/lib"
```

Guardar:

```text
Esc
:wq
Enter
```

### 13.5. Validar sintaxis

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Sin salida significa que no se detectó un error de sintaxis.

### 13.6. Aplicar

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

Resultado esperado:

```text
active
```

### 13.7. Validar el entorno efectivo

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

### 13.8. Revisar logs

```bash
sudo tail -n 100 /var/log/zabbix/zabbix_agent2.log
```

O bien:

```bash
sudo journalctl -u zabbix-agent2 \
  --since '10 minutes ago' --no-pager
```

El error `DPI-1047` no debe aparecer nuevamente después del reinicio.

## 14. Validar Oracle Ping

En Zabbix:

```text
Monitoreo → Últimos datos
```

Procedimiento:

1. Filtrar por el host Oracle Linux.
2. Buscar:

   ```text
   Oracle by Zabbix agent 2: Ping
   ```

3. Confirmar:

   ```text
   Up (1)
   ```

4. Revisar que la antigüedad sea reciente.
5. Confirmar que no aparezca un error en la columna de información.

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

No otorgar privilegios adicionales únicamente para eliminar un mensaje sin revisar antes el propósito y el licenciamiento de la consulta.

## 16. Checklist

- [ ] Agent 2 activo.
- [ ] Agent 2 escuchando en `10050/TCP`.
- [ ] Host Oracle Linux identificado correctamente.
- [ ] Plantilla Oracle retirada de la plantilla Linux.
- [ ] Entidades heredadas incorrectas limpiadas.
- [ ] Plantilla Linux vinculada directamente al host.
- [ ] Plantilla Oracle vinculada directamente al host.
- [ ] Interfaz pasiva disponible.
- [ ] `SERVICE_NAME` identificado mediante Oracle y listener.
- [ ] Usuario `ZABBIX_MON` creado y abierto.
- [ ] Privilegios revisados.
- [ ] Restricciones de licencia documentadas.
- [ ] Macros configuradas.
- [ ] Contraseña almacenada como texto secreto.
- [ ] SQL*Plus directo exitoso.
- [ ] Oracle Client localizado.
- [ ] Override `systemd` configurado si era necesario.
- [ ] Agent 2 activo después del cambio.
- [ ] `Zabbix agent ping = Up (1)`.
- [ ] `Oracle Ping = Up (1)`.
- [ ] Elementos no soportados revisados.

## 17. Referencias

- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Vincular y desvincular plantillas](https://www.zabbix.com/documentation/7.4/en/manual/config/templates/linking)
- [Configurar un host](https://www.zabbix.com/documentation/7.4/es/manual/config/hosts/host)
- [Plugin Oracle para Agent 2](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin)
- [Incidencias de Oracle](../base-conocimiento/oracle.md)
