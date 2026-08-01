# Incidencias: monitoreo de Oracle

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

Si la interfaz queda en timeout, consultar [Agentes Zabbix: interfaz pasiva en timeout](agentes-zabbix.md#4-interfaz-pasiva-en-timeout).

**Estado:** configurada; pendiente de validación final.

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

La configuración normal, las macros y los pendientes se mantienen únicamente en la [guía de implementación](../implementacion-zabbix-docker-oracle.md#parte-e-preparar-el-monitoreo-de-oracle-database).
