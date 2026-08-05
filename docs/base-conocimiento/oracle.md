# Incidencias: monitoreo de Oracle

## Estado de la prueba

- Oracle Linux ya está registrado en Zabbix.
- Están vinculadas directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

- La interfaz del agente está configurada en `192.0.0.73:10050`.
- La base es no-CDB.
- El `SERVICE_NAME` confirmado es `SIAL`.
- El usuario `ZABBIX_MON` ya fue creado.
- `SELECT_CATALOG_ROLE` y el permiso directo sobre `V_$ACTIVE_SESSION_HISTORY` fueron revocados.
- No se cuenta con Oracle Diagnostics Pack.
- La conectividad TCP al agente pasivo ya fue validada temporalmente desde `192.20.0.12`.
- El agente todavía rechaza la consulta pasiva por la lista de IP autorizadas en `Server=`.
- Las macros Oracle ya están configuradas en el host, pero la contraseña todavía no ha sido validada funcionalmente.

---

## 1. La plantilla Oracle se agregó dentro de la plantilla Linux

**Síntoma**

Al agregar `Oracle by Zabbix agent 2`, Zabbix mostró errores de interfaz o dependencias.

**Causa**

La plantilla Oracle se vinculó dentro de `Linux by Zabbix agent active` en vez de vincularse directamente al host.

**Solución**

No modificar las plantillas oficiales. Vincular ambas directamente al host Oracle Linux:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

**Estado:** resuelta.

---

## 2. La plantilla Oracle requiere interfaz pasiva

**Síntoma**

El host funciona con comprobaciones activas, pero Zabbix solicita una interfaz al agregar la plantilla Oracle.

**Causa**

La plantilla Oracle contiene elementos pasivos que Zabbix Server consulta mediante Agent 2.

**Solución**

Agregar al host una interfaz tipo **Agente**:

```text
IP: 192.0.0.73
Puerto: 10050
```

El agente debe tener habilitados:

```ini
Server=<IP_ZABBIX_SERVER_AUTORIZADA>
ListenPort=10050
```

**Estado:** conectividad TCP disponible con regla temporal; pendiente corregir `Server=` y hacer permanente la regla correcta de `firewalld`.

---

## 3. Se intentó utilizar SID en lugar de `SERVICE_NAME`

**Situación**

La operación habitual identifica la base por SID, pero la plantilla solicita `{$ORACLE.SERVICE}`.

**Diagnóstico**

```sql
SHOW PARAMETER service_names;
```

También puede revisarse:

```bash
lsnrctl status
```

**Solución**

Configurar `{$ORACLE.SERVICE}` con el `SERVICE_NAME` publicado por el listener. Para este servidor:

```text
SIAL
```

**Estado:** resuelta.

---

## 4. Permisos incompatibles con la licencia disponible

**Situación**

Inicialmente se otorgaron al usuario `ZABBIX_MON`:

```sql
GRANT SELECT_CATALOG_ROLE TO ZABBIX_MON;
GRANT SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY TO ZABBIX_MON;
```

La organización no cuenta con Oracle Diagnostics Pack.

**Acción aplicada**

Se revocaron ambos permisos:

```sql
REVOKE SELECT_CATALOG_ROLE FROM ZABBIX_MON;
REVOKE SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY FROM ZABBIX_MON;
```

Los permisos explícitos necesarios sobre las demás vistas de monitoreo se conservan.

**Estado:** resuelta.

---

## 5. Macros Oracle configuradas en el host

En **Recopilación de datos → Equipos → ZAM-SV-073-19C → Macros** se confirmaron:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = SIAL
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <secreto>
```

La estructura de las macros es correcta:

- `CONNSTRING` apunta al listener Oracle local del mismo servidor.
- `SERVICE` usa `SERVICE_NAME`, no SID.
- `USER` corresponde al usuario de monitoreo.
- `PASSWORD` está almacenada como secreto en Zabbix.

La captura solo confirma que existe un valor de contraseña; no demuestra que sea correcto.

**Estado:** configuración visual correcta; validación funcional pendiente.

---

## Punto de reanudación

Antes de concluir que las credenciales Oracle son incorrectas debe resolverse el rechazo del agente:

```text
Received empty response from Zabbix Agent ... Assuming that agent dropped connection because of access permissions.
```

Ese mensaje corresponde al control de acceso del agente, no a una autenticación Oracle fallida.

Después de corregir `Server=` se continuará con:

1. Validación del elemento `oracle.ping`.
2. Confirmación funcional de usuario y contraseña.
3. Copia de la plantilla Oracle para excluir consultas ASH.
4. Revisión de permisos exactos requeridos.
5. Validación de métricas y elementos no soportados.
