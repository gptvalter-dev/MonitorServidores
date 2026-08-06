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
- Agent 2 autoriza consultas pasivas desde `192.20.0.10` y `192.20.0.12`.
- La regla de `firewalld` para `192.20.0.12/32` ya es permanente.
- Las macros Oracle están configuradas en el host.
- La conexión directa por SQL*Plus con `ZABBIX_MON@//127.0.0.1:1521/SIAL` fue exitosa.
- La incidencia `DPI-1047` fue resuelta configurando el entorno Oracle del servicio `zabbix-agent2`.
- `oracle.ping` devuelve actualmente `Up (1)`.
- No fue necesario instalar Oracle Instant Client porque se reutilizaron las bibliotecas del `ORACLE_HOME` existente.
- La base opera en modo `NOARCHIVELOG`.

---

## 1. La plantilla Oracle se agregó dentro de la plantilla Linux

**Síntoma**

Al agregar `Oracle by Zabbix agent 2`, Zabbix mostró errores de interfaz o dependencias.

**Causa**

La plantilla Oracle se vinculó dentro de `Linux by Zabbix agent active` en vez de vincularse directamente al host.

**Solución**

Vincular directamente al host:

```text
Linux by Zabbix agent active
Oracle by Zabbix agent 2
```

**Estado:** resuelta.

---

## 2. La plantilla Oracle requiere interfaz pasiva

**Configuración aplicada**

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

La regla permanente quedó configurada así:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="192.20.0.12/32" port port="10050" protocol="tcp" accept'

sudo firewall-cmd --reload
```

Validación:

```text
192.0.0.73:10050
Estado: Disponible
Error: ninguno
```

**Estado:** resuelta y persistida.

---

## 3. Uso de `SERVICE_NAME`

La macro se configuró con:

```text
{$ORACLE.SERVICE} = SIAL
```

No se utilizó SID.

**Estado:** resuelta.

---

## 4. Permisos incompatibles con la licencia disponible

Se revocaron:

```sql
REVOKE SELECT_CATALOG_ROLE FROM ZABBIX_MON;
REVOKE SELECT ON SYS.V_$ACTIVE_SESSION_HISTORY FROM ZABBIX_MON;
```

**Estado:** resuelta. La plantilla debe clonarse posteriormente para excluir consultas relacionadas con ASH.

---

## 5. Macros Oracle

```text
{$ORACLE.CONNSTRING} = tcp://127.0.0.1:1521
{$ORACLE.SERVICE}    = SIAL
{$ORACLE.USER}       = ZABBIX_MON
{$ORACLE.PASSWORD}   = <secreto>
```

La estructura quedó validada mediante `oracle.ping`.

---

## 6. Validación directa de credenciales

```bash
sqlplus -L ZABBIX_MON@//127.0.0.1:1521/SIAL
```

Resultado:

```text
Connected to:
Oracle Database 19c Enterprise Edition
```

Se confirmó listener, servicio, usuario, contraseña y cuenta habilitada.

---

## 7. `oracle.ping` devolvía `Down (0)` por `DPI-1047`

**Error**

```text
DPI-1047: Cannot locate a 64-bit Oracle Client library:
"libclntsh.so: cannot open shared object file: No such file or directory"
```

**Biblioteca utilizada**

```text
/u01/app/oracle/product/19.3.0/dbhome_1/lib/libclntsh.so.19.1
```

Enlace confirmado:

```text
/u01/app/oracle/product/19.3.0/dbhome_1/lib/libclntsh.so -> libclntsh.so.19.1
```

---

## 8. Entorno Oracle para Agent 2

Archivo:

```text
/etc/systemd/system/zabbix-agent2.service.d/oracle.conf
```

Contenido:

```ini
[Service]
Environment="ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1"
Environment="LD_LIBRARY_PATH=/u01/app/oracle/product/19.3.0/dbhome_1/lib"
```

Aplicación:

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
```

Validación:

```text
CONFFILE=/etc/zabbix/zabbix_agent2.conf
ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
LD_LIBRARY_PATH=/u01/app/oracle/product/19.3.0/dbhome_1/lib
```

Resultado:

```text
Oracle by Zabbix agent 2: Ping
Último valor: Up (1)
Cambio: +1
```

**Estado:** resuelta.

---

## 9. Alerta de REDO logs disponibles

Zabbix reportó:

```text
Redo logs available to switch = 0
```

Consulta en `V$LOG`:

```text
Grupo 1: CURRENT
Grupo 2: ACTIVE
Grupo 3: ACTIVE
```

La base tiene tres grupos de 200 MB. Con uno siempre en estado `CURRENT`, el máximo teórico de grupos disponibles es dos, por lo que un umbral de menos de tres siempre permanecerá en problema.

También se confirmó:

```text
LOG_MODE = NOARCHIVELOG
```

En este modo, los grupos `ACTIVE` no están esperando archivado; siguen siendo necesarios para recuperación de instancia hasta que avance el checkpoint. Si la generación de REDO supera la capacidad de checkpoint/DBWR, un siguiente log switch puede quedar esperando un grupo reutilizable.

No se modificará todavía la base ni el umbral. Primero se medirá la frecuencia real de log switches.

**Estado:** en análisis.

---

## Validación alcanzada

1. Zabbix Server puede consultar pasivamente Agent 2.
2. Agent 2 puede cargar las bibliotecas Oracle Client.
3. Agent 2 puede conectarse al listener local.
4. El servicio `SIAL` responde.
5. El usuario `ZABBIX_MON` puede autenticarse.
6. Las macros Oracle se expanden correctamente.
7. `oracle.ping` devuelve `Up (1)`.
8. La regla de firewall permanece después de `firewall-cmd --reload`.

---

## Pendientes

1. Revisar frecuencia de cambios REDO y definir el umbral correcto.
2. Clonar la plantilla Oracle para excluir consultas relacionadas con ASH.
3. Auditar los permisos exactos de `ZABBIX_MON` contra la plantilla usada.
4. Revisar métricas Oracle no soportadas.
5. Definir criterios de aceptación y checklist final.
