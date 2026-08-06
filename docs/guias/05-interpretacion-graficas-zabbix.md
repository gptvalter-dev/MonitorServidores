# Cómo interpretar gráficas en Zabbix

> Alcance: esta guía explica cómo leer una gráfica de Zabbix. No describe el significado técnico de cada métrica.

## 1. Objetivo

Al finalizar será posible identificar:

- Qué periodo muestra una gráfica.
- Qué representan sus ejes.
- Qué significan `último`, `mínimo`, `media` y `máximo`.
- Cómo distinguir un pico aislado de una condición sostenida.
- Cómo corregir una diferencia de zona horaria.

## 2. Abrir una gráfica

En Zabbix:

```text
Monitoreo → Últimos datos
```

1. Seleccionar el equipo.
2. Buscar la métrica.
3. Hacer clic en **Gráficas**.

También pueden existir gráficas predefinidas en:

```text
Monitoreo → Equipos → Gráficas
```

## 3. Eje horizontal

El eje horizontal representa el tiempo.

Ejemplo:

```text
23:35  23:40  23:45  00:00  00:05
```

Estos valores son horas. El cambio de `23:55` a `00:00` indica el cambio de día.

Una etiqueta como:

```text
08-06 23:30
```

indica mes, día y hora de referencia del periodo mostrado.

La separación entre marcas depende del rango seleccionado. Puede representar minutos, horas, días o meses.

## 4. Eje vertical

El eje vertical representa el valor de la métrica.

La unidad depende de la métrica:

```text
%
B, KB, MB, GB
ms
operaciones por segundo
sesiones
procesos
0 o 1
```

Antes de interpretar una línea, confirmar la unidad del elemento en **Últimos datos**.

## 5. Leyenda inferior

Las gráficas suelen mostrar:

```text
último | mínimo | media | máximo
```

| Campo | Significado |
|---|---|
| `último` | Valor más reciente del periodo |
| `mínimo` | Menor valor observado |
| `media` | Promedio de los valores |
| `máximo` | Mayor valor observado |

El valor máximo no significa que toda la gráfica estuvo en ese nivel. Puede corresponder a un solo pico.

## 6. Pico aislado y valor sostenido

### Pico aislado

Una subida breve seguida de regreso al nivel habitual.

Puede deberse a:

- Inicio de un proceso.
- Respaldo.
- Consulta pesada.
- Checkpoint.
- Actividad temporal de usuarios.

No debe declararse incidente solo por un pico sin revisar duración y contexto.

### Valor sostenido

El valor permanece alto durante varios intervalos.

Tiene mayor relevancia porque puede indicar:

- Saturación.
- Falta de capacidad.
- Proceso detenido o bloqueado.
- Carga continua.
- Umbral mal definido.

## 7. Cambiar el periodo

En la parte superior de la gráfica se puede seleccionar un periodo, por ejemplo:

```text
Última hora
Últimas 6 horas
Último día
Última semana
Periodo personalizado
```

Para investigar:

1. Comenzar con un periodo corto.
2. Identificar el momento del cambio.
3. Ampliar el periodo para comparar con el comportamiento habitual.
4. Revisar si el patrón se repite diariamente o semanalmente.

## 8. Uso del cursor y zoom

Dependiendo del tipo de gráfica:

- Colocar el cursor sobre la línea muestra el valor y hora exactos.
- Arrastrar sobre un intervalo permite ampliar ese periodo.
- Los controles de navegación permiten avanzar o retroceder.

Registrar siempre la fecha, hora y zona horaria cuando se documente una incidencia.

## 9. Varias líneas en una gráfica

Una gráfica puede incluir varias series.

Ejemplo de disco:

```text
Lectura
Escritura
Utilización
Cola
```

Cada línea tiene una leyenda. No asumir que todas usan la misma unidad; revisar la definición de la gráfica y de cada elemento.

## 10. Zona horaria incorrecta

Síntoma: la hora del eje horizontal está adelantada o atrasada respecto al horario local.

Ejemplo:

```text
Hora local:  17:30
Gráfica:     23:30
Diferencia:  6 horas
```

### Corregir en el perfil del usuario

1. Hacer clic en el icono del usuario, en la parte inferior izquierda.
2. Abrir **Configuración de usuario → Perfil**.
3. En **Zona horaria**, seleccionar:

```text
America/Mexico_City
```

4. Guardar.
5. Recargar la gráfica.

La documentación oficial indica que la zona horaria del perfil sobrescribe la configuración global para ese usuario.

## 11. Zona horaria global del contenedor web

En una instalación Docker, la interfaz web puede usar una variable en:

```text
C:\docker\zabbix-docker\env_vars\.env_web
```

Crear respaldo:

```powershell
Copy-Item `
  C:\docker\zabbix-docker\env_vars\.env_web `
  C:\docker\zabbix-docker\env_vars\.env_web.respaldo
```

Abrir:

```powershell
notepad C:\docker\zabbix-docker\env_vars\.env_web
```

Configurar:

```dotenv
PHP_TZ=America/Mexico_City
```

Aplicar:

```powershell
Set-Location C:\docker\zabbix-docker
$env:OS="alpine"
docker compose up -d
```

Después, revisar nuevamente el perfil del usuario.

## 12. Preguntas antes de concluir que existe un problema

1. ¿Cuál es la unidad?
2. ¿Cuál es el valor actual?
3. ¿Cuánto duró el cambio?
4. ¿Es un pico o una condición sostenida?
5. ¿Cuál es el comportamiento normal del servidor?
6. ¿Qué otras métricas cambiaron al mismo tiempo?
7. ¿El horario de la gráfica es correcto?
8. ¿El trigger usa un umbral genérico?
9. ¿Hubo respaldo, mantenimiento o carga programada?

## 13. Evidencia mínima de una gráfica

Al documentar un hallazgo registrar:

```text
Equipo:
Métrica:
Periodo mostrado:
Zona horaria:
Unidad:
Último:
Mínimo:
Media:
Máximo:
Hora del pico:
Duración:
Métricas relacionadas:
Conclusión provisional:
```

## 14. Errores comunes

| Error | Corrección |
|---|---|
| Leer las horas como valores | Recordar que pertenecen al eje horizontal |
| Interpretar el máximo como valor constante | Revisar la forma y duración del pico |
| Comparar líneas con unidades distintas | Revisar la unidad de cada elemento |
| Ignorar la zona horaria | Configurar perfil y frontend |
| Analizar una sola métrica | Comparar con métricas relacionadas |
| Cambiar un umbral por una sola gráfica | Reunir tendencia y contexto operativo |

## 15. Referencias

- [Configuración del perfil de usuario](https://www.zabbix.com/documentation/7.4/en/manual/web_interface/user_profile)
- [Instalación desde contenedores](https://www.zabbix.com/documentation/7.4/es/manual/installation/containers)
