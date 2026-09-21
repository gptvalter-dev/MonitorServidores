# Incidencias: monitoreo de Oracle Database

> Este archivo contiene únicamente incidencias reales. El procedimiento normal completo está en [Monitoreo de Oracle Database](../guias/04-monitoreo-oracle-database.md).

## Estado funcional

- Linux y Oracle están vinculados como plantillas independientes al host.
- Agent 2 activo y checks activos/pasivos validados.
- Oracle Client accesible para el servicio Agent 2.
- Usuario de monitoreo conecta por SQL*Plus.
- `Zabbix agent ping = Up (1)`.
- `Oracle Ping = Up (1)`.
- Ambiente evaluado: `NOARCHIVELOG` y sin Oracle Diagnostics Pack.

## 1. Plantilla Oracle vinculada dentro de la plantilla Linux

**Síntoma**

Hosts Linux podían heredar elementos Oracle sin ejecutar Oracle Database y aparecían problemas de dependencias/interfaz.

**Causa**

Se creó una jerarquía incorrecta:

```text
Host Oracle Linux
└── Linux by Zabbix agent active
    └── Oracle by Zabbix agent 2
```

**Corrección**

Desvincular y limpiar la relación accidental y vincular ambas plantillas directamente al host:

```text
Host Oracle Linux
├── Linux by Zabbix agent active
└── Oracle by Zabbix agent 2
```

**Validación**

- Linux conserva datos.
- `Oracle Ping` recopila datos.
- No quedan entidades Oracle heredadas por la plantilla Linux.

**Estado:** resuelta.

---

## 2. Plantilla Oracle sin datos por falta de interfaz/check pasivo

**Síntoma**

La plantilla estaba vinculada, pero `Oracle Ping` no recibía datos o la interfaz Agent no estaba disponible.

**Causa**

La integración requiere que Zabbix Server/Proxy pueda consultar Agent 2 por el puerto pasivo configurado. La IP real de origen debía coincidir con `Server=` y firewall.

**Diagnóstico clave**

```bash
sudo ss -lntp | grep ':10050'
sudo tcpdump -nni any tcp port 10050
```

**Corrección**

- Autorizar en `Server=` únicamente el origen real.
- Crear regla persistente en `firewalld`.
- Configurar interfaz Agent del host con IP/DNS alcanzable.

**Validación**

```text
Interfaz Agent = Disponible
Oracle Ping = Up (1)
```

**Estado:** resuelta.

---

## 3. Se utilizó SID en lugar de `SERVICE_NAME`

**Síntoma**

La conexión local podía funcionar, pero la plantilla no conectaba o `Oracle Ping = Down (0)`.

**Causa**

`{$ORACLE.SERVICE}` debe usar el servicio publicado por el listener; SID y `SERVICE_NAME` no deben asumirse equivalentes.

**Diagnóstico**

```sql
SHOW PARAMETER service_names;
```

```bash
lsnrctl status
sqlplus -L <USUARIO_MONITOREO>@//127.0.0.1:1521/<ORACLE_SERVICE>
```

**Corrección**

```text
{$ORACLE.SERVICE} = <ORACLE_SERVICE>
```

**Estado:** resuelta.

---

## 4. Privilegios incompatibles con el licenciamiento disponible

**Síntoma**

La configuración inicial otorgaba privilegios amplios, incluido acceso a vistas relacionadas con Active Session History.

**Causa**

El ambiente evaluado no tiene Oracle Diagnostics Pack. No debe asumirse que todas las consultas de una plantilla pueden habilitarse sin revisar licenciamiento.

**Corrección aplicada**

Se retiraron privilegios amplios/ASH que no correspondían al ambiente y se adoptó esta regla:

```text
Plantilla oficial -> referencia
Plantilla custom   -> adaptaciones por licenciamiento/política
```

Antes de producción queda pendiente auditar la revisión exacta de la plantilla y definir privilegios mínimos.

**Estado:** mitigada; auditoría pendiente.

---

## 5. `Oracle Ping = Down (0)` por `DPI-1047`

**Error**

```text
DPI-1047: Cannot locate a 64-bit Oracle Client library
libclntsh.so: cannot open shared object file
```

**Diagnóstico**

SQL*Plus funcionaba y `libclntsh.so` existía, pero `zabbix-agent2` iniciado por `systemd` no heredaba el entorno del usuario Oracle.

```bash
find <ORACLE_HOME> -name 'libclntsh.so*'
systemctl show zabbix-agent2 -p Environment
```

**Corrección**

Crear un override controlado de `systemd`:

```ini
[Service]
Environment="ORACLE_HOME=<ORACLE_HOME>"
Environment="LD_LIBRARY_PATH=<ORACLE_HOME>/lib"
```

Aplicar:

```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent2
```

**Validación**

- Variables visibles en `systemctl show`.
- Sin nuevos `DPI-1047` en logs.
- `Oracle Ping = Up (1)`.

**Estado:** resuelta.

---

## 6. Alerta `Redo logs available to switch = 0`

**Síntoma**

Zabbix reportó cero grupos disponibles para cambio.

**Datos observados**

```text
3 grupos totales
1 CURRENT
2 ACTIVE
0 INACTIVE/UNUSED
LOG_MODE = NOARCHIVELOG
```

**Interpretación**

Un umbral genérico que espera tres grupos disponibles no encaja con una configuración de tres grupos donde uno siempre es `CURRENT`. Sin embargo, cero grupos reutilizables también exige revisar frecuencia de log switches y checkpoints.

**Decisión**

No agregar/eliminar REDO groups ni cambiar el trigger solo para cerrar la alerta.

Pendiente:

- observar `V$LOG_HISTORY`;
- medir frecuencia de switches;
- correlacionar con checkpoints/carga;
- definir umbral adecuado al ambiente.

**Estado:** en análisis.

---

## Pendientes Oracle

- [ ] Auditar privilegios mínimos contra la versión exacta de la plantilla.
- [ ] Crear/validar plantilla custom sin métricas que impliquen funcionalidades no licenciadas.
- [ ] Resolver elementos `No soportada` restantes con causa documentada.
- [ ] Cerrar análisis de REDO.

## Referencias

- Guía operativa: [Monitoreo de Oracle Database](../guias/04-monitoreo-oracle-database.md)
- Integración oficial: https://www.zabbix.com/integrations/oracle
- Plugin Oracle Agent 2: https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/oracle_plugin
