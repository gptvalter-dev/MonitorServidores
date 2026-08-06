# Incidencias: monitoreo de Oracle

## Estado de la prueba

- Oracle Linux ya está registrado en Zabbix.
- Están vinculadas directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

- La interfaz del agente está configurada en `192.0.0.73:10050`.
- La interfaz pasiva aparece como **Disponible** en Zabbix.
- La base es no-CDB.
- El `SERVICE_NAME` confirmado es `SIAL`.
- El usuario `ZABBIX_MON` ya fue creado.
- `SELECT_CATALOG_ROLE` y el permiso directo sobre `V_$ACTIVE_SESSION_HISTORY` fueron revocados.
- No se cuenta con Oracle Diagnostics Pack.
- La conectividad TCP al agente pasivo fue validada desde `192.20.0.12`.
- Agent 2 autoriza consultas pasivas desde `192.20.0.10` y `192.20.0.12`.
- Las macros Oracle están configuradas en el host.
- La conexión directa por SQL*Plus con `ZABBIX_MON@//127.0.0.1:1521/SIAL` fue exitosa.
- La incidencia `DPI-1047` fue resuelta configurando el entorno Oracle del servicio `zabbix-agent2`.
- `oracle.ping` devuelve actualmente `Up (1)`.
- No fue necesario instalar Oracle Instant Client porque se reutilizaron las bibliotecas del `ORACLE_HOME` existente.

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

**Estado:** resuelta. La regla de `firewalld` para `192.20.0.12/32` sigue pendiente de hacerse permanente.

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

La estructura de las macros quedó validada funcionalmente mediante `oracle.ping`.

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

Esto confirmó:

- Listener disponible en `127.0.0.1:1521`.
- `SERVICE_NAME` `SIAL` válido.
- Usuario `ZABBIX_MON` válido.
- Contraseña válida.
- Cuenta habilitada para iniciar sesión.

---

## 7. `oracle.ping` devolvía `Down (0)` por `DPI-1047`

**Síntoma en Zabbix**

```text
Oracle by Zabbix agent 2: Ping
Último valor: Down (0)
```

**Error en el log**

Archivo:

```text
/var/log/zabbix/zabbix_agent2.log
```

Mensaje:

```text
DPI-1047: Cannot locate a 64-bit Oracle Client library:
"libclntsh.so: cannot open shared object file: No such file or directory"
```

**Causa**

La biblioteca Oracle Client sí existía, pero el servicio `zabbix-agent2`, iniciado por `systemd`, no recibía `ORACLE_HOME` ni `LD_LIBRARY_PATH`.

SQL*Plus funcionaba bajo el usuario `oracle` porque esa sesión sí tenía cargado el entorno Oracle.

**Biblioteca utilizada**

```text
/u01/app/oracle/product/19.3.0/dbhome_1/lib/libclntsh.so.19.1
```

Enlace confirmado:

```text
/u01/app/oracle/product/19.3.0/dbhome_1/lib/libclntsh.so -> libclntsh.so.19.1
```

La copia localizada bajo `/home/oracle/Downloads/...` pertenece a archivos de parche y no se utilizó.

---

## 8. Configuración del entorno Oracle para Agent 2

Se creó el directorio de sobrescritura de `systemd`:

```bash
sudo mkdir -p /etc/systemd/system/zabbix-agent2.service.d
```

Se creó:

```text
/etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Contenido:

```ini
[Service]
Environment="ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1"
Environment="LD_LIBRARY_PATH=/u01/app/oracle/product/19.3.0/dbhome_1/lib"
```

Después se aplicó:

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
```

Validación:

```bash
sudo systemctl show zabbix-agent2 -p Environment --value | tr ' ' '\n'
```

Resultado:

```text
CONFFILE=/etc/zabbix/zabbix_agent2.conf
ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
LD_LIBRARY_PATH=/u01/app/oracle/product/19.3.0/dbhome_1/lib
```

**Resultado final en Zabbix**

```text
Oracle by Zabbix agent 2: Ping
Último valor: Up (1)
Cambio: +1
```

**Estado:** resuelta.

---

## Validación alcanzada

Quedaron comprobados los siguientes niveles:

1. Zabbix Server puede consultar pasivamente Agent 2.
2. Agent 2 puede cargar las bibliotecas Oracle Client.
3. Agent 2 puede conectarse al listener local.
4. El servicio `SIAL` responde.
5. El usuario `ZABBIX_MON` puede autenticarse.
6. Las macros Oracle se expanden correctamente.
7. `oracle.ping` devuelve `Up (1)`.

---

## Pendientes

1. Hacer permanente la regla de `firewalld` para `192.20.0.12/32`.
2. Validar la conectividad después de recargar `firewalld`.
3. Clonar la plantilla Oracle para excluir consultas relacionadas con ASH por no contar con Diagnostics Pack.
4. Auditar los permisos exactos de `ZABBIX_MON` contra la plantilla usada.
5. Revisar las métricas Oracle no soportadas.
6. Definir criterios de aceptación y checklist final.
