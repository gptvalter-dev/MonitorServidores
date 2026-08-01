# Incidencias y estado: monitoreo de Oracle

> Estado actual: **pendiente de conexión completa**. El servidor Oracle Linux ya tiene Agent 2 y la plantilla Oracle se está preparando, pero todavía no se reciben métricas válidas de la base de datos.

## 1. La plantilla Oracle se agregó dentro de la plantilla Linux

**Síntoma**

Al intentar agregar `Oracle by Zabbix agent 2`, Zabbix mostró errores relacionados con interfaz o dependencia de elementos.

**Causa identificada**

La plantilla Oracle se estaba vinculando dentro de la plantilla oficial `Linux by Zabbix agent active`, en lugar de vincularse directamente al host.

**Solución**

No modificar las plantillas oficiales. En el host Oracle Linux deben aparecer directamente:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

Ruta:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Plantillas
```

**Estado:** resuelta.

---

## 2. La plantilla Oracle requiere interfaz pasiva

**Síntoma**

El host funcionaba con comprobaciones activas y sin interfaz, pero la plantilla Oracle requería una interfaz de agente.

**Causa**

`Oracle by Zabbix agent 2` contiene elementos pasivos que Zabbix Server consulta a través de Agent 2.

**Solución**

Agregar al host una interfaz tipo **Agente**:

```text
IP: <IP_ORACLE_LINUX>
Conectar mediante: IP
Puerto: 10050
```

Agent 2 puede atender simultáneamente comprobaciones activas y pasivas.

Configuración mínima del agente:

```ini
Server=<IP_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>:11051
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

**Estado:** configurada; validar disponibilidad pasiva.

---

## 3. SID y `SERVICE_NAME` no son lo mismo para la plantilla

**Situación**

La operación cotidiana utiliza un SID, pero la plantilla oficial solicita `{$ORACLE.SERVICE}`.

**Validación en Oracle**

```sql
SHOW PARAMETER service_names;
```

También puede consultarse el listener:

```bash
lsnrctl status
```

**Conclusión**

La macro `{$ORACLE.SERVICE}` debe contener el `SERVICE_NAME` publicado por el listener, no asumir que el SID es válido para esa macro.

Ejemplo conceptual:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = <ORACLE_SERVICE>
```

**Estado:** identificado.

---

## 4. Tipo de base identificado

Consultas realizadas:

```sql
SELECT CDB FROM V$DATABASE;
SELECT SYS_CONTEXT('USERENV', 'CON_NAME') FROM DUAL;
```

Resultado funcional:

```text
CDB = NO
```

La base es **no-CDB**, por lo que el usuario de monitoreo puede ser un usuario normal:

```text
ZABBIX_MON
```

No se requiere el prefijo `C##`.

---

## 5. Usuario de aplicación vs. usuario de monitoreo

En Oracle, normalmente un usuario es propietario de un esquema con el mismo nombre. No se debe utilizar el usuario funcional de la aplicación para Zabbix.

Diseño recomendado:

```text
Usuario de aplicación → operación funcional
ZABBIX_MON            → monitoreo técnico de solo lectura
```

No usar `SYS`, `SYSTEM` ni un usuario de aplicación en las macros de Zabbix.

---

## 6. Macros pendientes de configurar

Ruta:

```text
Recopilación de datos → Equipos → <HOSTNAME_LINUX> → Macros
```

Macros:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = <ORACLE_SERVICE>
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <CONTRASENA_SEGURA>
```

La contraseña debe configurarse como **Texto secreto**.

---

## 7. Pendientes para cerrar la conexión Oracle

- [ ] Confirmar que la interfaz pasiva `10050` aparece disponible.
- [ ] Crear el usuario `ZABBIX_MON` en la base no-CDB.
- [ ] Otorgar los permisos mínimos requeridos por la plantilla oficial.
- [ ] Confirmar si existe licencia de Oracle Diagnostics Pack antes de permitir consultas a `V$ACTIVE_SESSION_HISTORY`.
- [ ] Configurar las macros del host.
- [ ] Validar `oracle.ping`.
- [ ] Confirmar métricas en **Monitoreo → Últimos datos**.
- [ ] Revisar tablespaces, sesiones, procesos, SGA/PGA, redo, archive y FRA.

## Estado

**Pendiente:** hasta este punto no debe afirmarse que Oracle Database está monitoreado correctamente.
