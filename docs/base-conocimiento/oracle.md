# Incidencias: monitoreo de Oracle

## Estado de la prueba

- Oracle Linux ya está registrado en Zabbix.
- Están vinculadas directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

- La interfaz del agente está configurada en `<IP_ORACLE_LINUX>:10050`.
- La base es no-CDB.
- El `SERVICE_NAME` está identificado.
- El usuario `ZABBIX_MON` ya fue creado.
- `SELECT_CATALOG_ROLE` y el permiso directo sobre `V_$ACTIVE_SESSION_HISTORY` fueron revocados.
- No se cuenta con Oracle Diagnostics Pack.
- Las macros Oracle todavía no deben configurarse hasta resolver la comunicación pasiva y preparar la plantilla sin ASH.

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
IP: <IP_ORACLE_LINUX>
Puerto: 10050
```

El agente debe tener habilitados:

```ini
Server=<IP_ZABBIX_SERVER>
ListenPort=10050
```

**Estado:** configurada, pero actualmente no disponible por timeout. Consultar [Agentes Zabbix: interfaz pasiva en timeout](agentes-zabbix.md#4-interfaz-pasiva-en-timeout).

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

Configurar `{$ORACLE.SERVICE}` con el `SERVICE_NAME` publicado por el listener, no asumir que coincide con el SID.

**Estado:** identificado.

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

## Punto de reanudación

Antes de configurar las macros o probar la conexión Oracle, debe resolverse el acceso pasivo al agente.

Última validación realizada desde el servidor Windows:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

Siguiente comando, en Oracle Linux:

```bash
sudo ss -lntp | grep ':10050'
```

No avanzar a macros, credenciales ni `oracle.ping` hasta confirmar:

```text
TcpTestSucceeded : True
```

Después de cerrar la comunicación pasiva se continuará con:

1. Copia de la plantilla Oracle sin consultas ASH.
2. Configuración de macros.
3. Prueba de conexión del usuario `ZABBIX_MON`.
4. Validación de `oracle.ping`.
5. Revisión de métricas y elementos no soportados.
