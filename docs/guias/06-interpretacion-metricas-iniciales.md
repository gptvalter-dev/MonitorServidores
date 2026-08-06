# Interpretación inicial de métricas de Zabbix

> Alcance: esta guía documenta las primeras métricas revisadas durante el laboratorio. Los umbrales deben validarse con el comportamiento real de cada servidor antes de usarse en producción.

## 1. Cómo usar esta guía

Para cada métrica revisar:

```text
Qué mide
Unidad
Cómo interpretarla
Qué otras métricas revisar
Posibles causas
Umbral actual
Acción operativa
```

No ajustar Oracle, Linux ni Zabbix usando una sola lectura aislada.

---

# Métricas del sistema operativo

## 2. Zabbix agent ping

### Qué mide

Indica si Agent 2 está enviando correctamente las comprobaciones activas.

### Valores

```text
Up (1)    Agente disponible
Sin datos Comunicación activa interrumpida o configuración incorrecta
```

### Qué revisar si no hay datos

- `ServerActive`.
- Conectividad al puerto del Zabbix Server.
- Coincidencia exacta de `Hostname`.
- Estado del servicio `zabbix-agent2`.
- Logs del agente.

## 3. CPU

### Qué mide

Porcentaje de tiempo que el procesador utiliza en diferentes estados.

Estados comunes:

```text
user     trabajo de aplicaciones
system   trabajo del kernel
idle     tiempo sin uso
iowait   espera relacionada con entrada/salida
steal    tiempo tomado por el hipervisor
```

### Interpretación

- CPU alta durante segundos puede ser normal.
- CPU alta sostenida requiere identificar procesos y carga.
- `iowait` alto puede apuntar a almacenamiento, no a falta de CPU.
- En máquinas virtuales, `steal` alto puede indicar competencia con otras máquinas del host.

### Revisar junto con

```text
Load average
Procesos
Memoria
Disk utilization
Disk waiting time
```

## 4. Memoria

### Qué mide

Uso de memoria RAM del sistema.

En Linux, memoria usada no equivale automáticamente a falta de memoria porque el sistema utiliza RAM como caché.

### Revisar

```text
Memoria disponible
Memoria utilizada
Caché
Swap utilizada
Procesos con mayor consumo
```

La métrica más útil para disponibilidad real suele ser la memoria disponible, no únicamente `used`.

## 5. Swap

### Qué mide

Uso de espacio de intercambio en disco.

### Interpretación

- Un valor pequeño y estable puede ser normal.
- Crecimiento continuo puede indicar presión de memoria.
- Actividad alta de entrada y salida de swap puede afectar el rendimiento.

### Revisar junto con

```text
Memoria disponible
Swap in
Swap out
Latencia de disco
Procesos
```

## 6. Espacio en disco

### Qué mide

Porcentaje o cantidad de espacio utilizado y libre por sistema de archivos.

### Interpretación

No revisar únicamente el porcentaje. También considerar:

- Espacio libre real en GB.
- Velocidad de crecimiento.
- Inodos disponibles.
- Punto de montaje.
- Impacto de quedarse sin espacio.

Para Oracle deben revisarse especialmente los sistemas de archivos donde viven datafiles, REDO, FRA, respaldos y logs.

## 7. Tiempo promedio de espera del disco

Ejemplo:

```text
sda: Disk average waiting time
```

### Qué mide

Tiempo promedio desde que Linux solicita una operación de entrada/salida hasta que termina.

### Unidad

```text
milisegundos (ms)
```

### Series comunes

| Serie | Significado |
|---|---|
| `r_await` | Espera promedio de lecturas |
| `w_await` | Espera promedio de escrituras |

### Interpretación

- Valores bajos y estables indican respuesta rápida.
- Picos breves pueden deberse a respaldos, consultas o checkpoints.
- Valores altos sostenidos pueden indicar saturación, cola creciente, almacenamiento lento o competencia entre procesos.

### Revisar junto con

```text
Disk utilization
Disk average queue size
Disk read rate
Disk write rate
CPU iowait
```

No mide espacio libre; mide latencia.

## 8. Utilización del disco

### Qué mide

Porcentaje del tiempo en que el dispositivo estuvo ocupado atendiendo operaciones.

### Interpretación

Una utilización alta no siempre significa problema si la latencia sigue baja. Debe analizarse junto con cola y tiempos de espera.

## 9. Cola promedio del disco

### Qué mide

Número promedio de operaciones esperando o siendo atendidas.

### Interpretación

Una cola creciente acompañada de latencia alta puede indicar saturación.

## 10. Red

### Métricas comunes

```text
Bytes recibidos
Bytes enviados
Errores
Paquetes descartados
Estado de la interfaz
```

### Interpretación

- Tráfico alto puede ser normal para el servidor.
- Errores y descartes sostenidos requieren revisar interfaz, red y capacidad.
- Comparar con velocidad nominal de la interfaz.

## 11. Uptime

### Qué mide

Tiempo transcurrido desde el último arranque.

Un valor que regresa a cero indica reinicio. Debe correlacionarse con mantenimiento o incidente.

---

# Métricas de Oracle Database

## 12. Oracle Ping

### Qué mide

Confirma que Agent 2 puede conectarse a Oracle con las macros configuradas.

### Valores

```text
Up (1)    Conexión correcta
Down (0)  Fallo de conexión o integración
```

### Qué confirma `Up (1)`

- Comunicación pasiva con Agent 2.
- Oracle Client disponible.
- Listener accesible.
- Servicio correcto.
- Credenciales válidas.
- Macros correctas.

## 13. PGA

PGA significa **Program Global Area**.

### Qué es

Memoria privada utilizada por procesos Oracle para:

- Ordenamientos.
- `GROUP BY`.
- `HASH JOIN`.
- Cursores.
- Información de sesión.
- Operaciones PL/SQL.

### Métricas comunes

| Métrica | Significado |
|---|---|
| `Total in use` | Memoria PGA utilizada actualmente |
| `Total allocated` | Memoria total asignada por Oracle |
| `Total freeable` | Memoria asignada que podría liberarse |
| `Aggregate target` | Objetivo configurado de PGA |
| `Global memory bound` | Límite aproximado de un área de trabajo automática |

### Cálculo inicial

```text
PGA utilizada / PGA_AGGREGATE_TARGET × 100
```

### Interpretación

- Una lectura alta aislada no confirma un problema.
- Revisar tendencia, duración, carga SQL y memoria disponible del sistema operativo.
- Oracle puede superar temporalmente el objetivo bajo determinadas cargas.

## 14. SGA

SGA significa **System Global Area**.

### Qué es

Memoria compartida de la instancia Oracle.

Incluye componentes como:

```text
Buffer cache
Shared pool
Large pool
Java pool
Redo log buffer
```

### Interpretación

No evaluar solo el tamaño. Debe relacionarse con memoria física, PGA, configuración automática y carga de la base.

## 15. Sesiones Oracle

### Qué mide

Cantidad de sesiones abiertas o activas.

### Revisar

```text
Sesiones totales
Sesiones activas
Límite configurado
Procesos Oracle
Tendencia por horario
```

Un número alto puede ser normal si la aplicación usa pool de conexiones. Debe conocerse la arquitectura de la aplicación.

## 16. Procesos Oracle

### Qué mide

Cantidad de procesos utilizados respecto al límite configurado.

Revisar junto con sesiones y parámetros `processes` y `sessions`.

## 17. Tablespaces

### Qué mide

Espacio utilizado y disponible por tablespace.

### Revisar

- Porcentaje utilizado.
- Espacio libre real.
- Autoextend.
- Tamaño máximo.
- Velocidad de crecimiento.
- Espacio disponible en el sistema de archivos o ASM.

No asumir que autoextend elimina el riesgo; puede trasladar el problema al almacenamiento físico.

## 18. REDO logs disponibles

Zabbix reportó durante el laboratorio:

```text
Redo logs available to switch = 0
```

### Consulta utilizada

```sql
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

### Resultado observado

```text
1 grupo CURRENT
2 grupos ACTIVE
0 grupos INACTIVE o UNUSED
```

La base tiene tres grupos de 200 MB y utiliza:

```text
NOARCHIVELOG
```

### Interpretación provisional

- `CURRENT`: grupo donde Oracle escribe actualmente.
- `ACTIVE`: todavía necesario para recuperación de instancia.
- `INACTIVE`: puede reutilizarse.
- `UNUSED`: todavía no utilizado.

El umbral predeterminado de menos de tres grupos disponibles no es compatible con un total de tres grupos porque uno siempre estará `CURRENT`.

Sin embargo, el valor cero requiere revisar:

- Frecuencia de log switches.
- Checkpoints.
- Tamaño de grupos.
- Carga de escritura.
- Mensajes de espera relacionados con REDO.

No agregar grupos ni cambiar el umbral únicamente para cerrar la alerta.

## 19. Fast Recovery Area

### Qué mide

Uso de la zona de recuperación rápida cuando está configurada.

### Revisar

```text
Límite
Espacio utilizado
Espacio recuperable
Archivelogs
Backups
Flashback logs
```

En una base `NOARCHIVELOG`, algunas métricas pueden no aplicar de la misma forma que en una base `ARCHIVELOG`.

---

# Formato para documentar nuevas métricas

## 20. Ficha estándar

```text
Nombre:
Plataforma:
Plantilla:
Clave:
Qué mide:
Unidad:
Intervalo de actualización:
Valor actual:
Rango normal observado:
Umbral predeterminado:
Umbral propuesto:
Duración antes de alertar:
Métricas relacionadas:
Posibles causas:
Acción operativa:
Responsable:
Estado de validación:
```

## 21. Referencias

- [Integración oficial de Linux](https://www.zabbix.com/integrations/linux)
- [Integración oficial de Oracle](https://www.zabbix.com/integrations/oracle)
- [Documentación Oracle PGA](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/tuning-program-global-area.html)
- [Incidencias Oracle](../base-conocimiento/oracle.md)
