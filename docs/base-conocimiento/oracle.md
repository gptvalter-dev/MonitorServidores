# Incidencias: monitoreo de Oracle

## Estado de la prueba

- Oracle Linux ya está registrado en Zabbix.
- Están vinculadas directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

- La interfaz del agente está configurada en `192.0.0.73:10050`.
- La interfaz pasiva ya aparece como **Disponible** en Zabbix.
- La base es no-CDB.
- El `SERVICE_NAME` confirmado es `SIAL`.
- El usuario `ZABBIX_MON` ya fue creado.
- `SELECT_CATALOG_ROLE` y el permiso directo sobre `V_$ACTIVE_SESSION_HISTORY` fueron revocados.
- No se cuenta con Oracle Diagnostics Pack.
- La conectividad TCP al agente pasivo fue validada desde `192.20.0.12`.
- Agent 2 autoriza consultas pasivas desde `192.20.0.10` y `192.20.0.12`.
- Las macros Oracle están configuradas en el host.
- La conexión directa por SQL*Plus con `ZABBIX_MON@//127.0.0.1:1521/SIAL` fue exitosa.
- `oracle.ping` todavía devuelve `Down (0)`.

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

El host funcionaba con comprobaciones activas, pero Zabbix requería una interfaz pasiva para la plantilla Oracle.

**Solución aplicada**

Interfaz tipo **Agente**:

```text
IP: 192.0.0.73
Puerto: 10050
```

En Agent 2:

```ini
Server=192.20.0.10,192.20.0.12
ServerActive=192.20.0.10:11051
Hostname=ZAM-SV-073-19C
```

Después de reiniciar `zabbix-agent2`, la interfaz cambió a:

```text
Estado: Disponible
Error: ninguno
```

**Estado:** resuelta. La regla de `firewalld` para `192.20.0.12/32` sigue pendiente de hacerse permanente al finalizar las validaciones.

---

## 3. Se intentó utilizar SID en lugar de `SERVICE_NAME`

**Situación**

La operación habitual identifica la base por SID, pero la plantilla solicita `{$ORACLE.SERVICE}`.

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

```sql
REVOKE SELECT_CATALOG_ROLE FROM ZABBIX_MON;
REVOKE SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY FROM ZABBIX_MON;
```

Los permisos explícitos necesarios sobre las demás vistas de monitoreo se conservan.

**Estado:** resuelta. La plantilla debe clonarse posteriormente para excluir consultas relacionadas con ASH.

---

## 5. Macros Oracle configuradas en el host

En **Recopilación de datos → Equipos → ZAM-SV-073-19C → Macros** se confirmaron:

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = SIAL
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <secreto>
```

La estructura visual de las macros es correcta.

---

## 6. Validación directa de credenciales

Se ejecutó desde Oracle Linux:

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/SIAL
```

Resultado:

```text
Connected to:
Oracle Database 19c Enterprise Edition
```

Esto confirma, usando la misma contraseña introducida manualmente:

- Listener disponible en `127.0.0.1:1521`.
- `SERVICE_NAME` `SIAL` válido.
- Usuario `ZABBIX_MON` válido.
- Contraseña válida.
- Cuenta habilitada para iniciar sesión.

Sin embargo, en Zabbix:

```text
Oracle by Zabbix agent 2: Ping
Último valor: Down (0)
```

Por lo tanto, el problema restante ya no apunta inicialmente al listener, servicio o credenciales Oracle. Debe revisarse el contexto de ejecución del complemento Oracle de Agent 2 y su registro de errores.

---

## Punto de reanudación

Revisar errores del complemento Oracle inmediatamente después de ejecutar `oracle.ping`:

```bash
sudo grep -iE 'oracle|ORA-|plugin' /var/log/zabbix/zabbix_agent2.log | tail -n 50
```

Después se determinará si existe:

1. Error del complemento Oracle.
2. Diferencia entre la contraseña guardada en la macro y la probada manualmente.
3. Problema de formato de conexión en Agent 2.
4. Dependencia o configuración faltante del complemento.
5. Error de permisos al ejecutar las consultas de la plantilla.
