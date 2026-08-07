# Parametrización de métricas en Zabbix

> Alcance: esta guía explica cómo ajustar métricas, intervalos y umbrales sin modificar directamente las plantillas oficiales. También define el método para crear y validar métricas personalizadas.

## 1. Objetivo

Al finalizar será posible:

- Identificar qué elemento genera un valor.
- Identificar qué trigger genera un problema.
- Localizar macros heredadas.
- Sobrescribir umbrales a nivel host.
- Ajustar intervalos de actualización de forma controlada.
- Documentar y revertir cambios.
- Diseñar una métrica personalizada de prueba.

## 2. Conceptos necesarios

### Elemento

Es la definición de una métrica individual.

Ejemplos:

```text
CPU utilization
Zabbix agent ping
Oracle Ping
PGA used
Disk waiting time
```

### Clave

Identificador técnico que indica cómo se obtiene el valor.

Ejemplos:

```text
agent.ping
system.cpu.util
vfs.fs.size
oracle.ping
```

### Trigger

Expresión que determina cuándo una métrica representa un problema.

### Macro

Variable reutilizable empleada para umbrales, direcciones, usuarios y otros parámetros.

Ejemplo:

```text
{$ORACLE.PGA.USE.MAX.WARN}
```

### Intervalo de actualización

Frecuencia con la que Zabbix obtiene el valor.

### Histórico

Valores detallados conservados durante un periodo.

### Tendencias

Resúmenes por hora utilizados para conservar información durante más tiempo con menos espacio.

## 3. Regla principal

No cambiar una métrica o umbral sin conocer:

- Qué mide.
- Unidad.
- Origen.
- Frecuencia.
- Comportamiento normal.
- Periodo evaluado por el trigger.
- Consecuencia de alertar tarde.
- Consecuencia de alertar demasiado.

## 4. No modificar plantillas oficiales

Las plantillas oficiales pueden actualizarse con nuevas versiones de Zabbix. Una modificación directa puede perderse o complicar futuras actualizaciones.

Opciones recomendadas:

1. Sobrescribir macros a nivel host.
2. Clonar la plantilla cuando se requieran cambios estructurales.
3. Crear una plantilla propia para métricas personalizadas.

## 5. Localizar la métrica

En Zabbix:

```text
Monitoreo → Últimos datos
```

1. Seleccionar el equipo.
2. Buscar el nombre de la métrica.
3. Anotar nombre, último valor, unidad y antigüedad.
4. Abrir el elemento cuando se requiera revisar la configuración.

También puede localizarse en:

```text
Recopilación de datos → Equipos → <HOST> → Elementos
```

## 6. Identificar el trigger relacionado

Ruta:

```text
Recopilación de datos → Equipos → <HOST> → Triggers
```

Buscar por el nombre del problema.

Registrar:

```text
Nombre del trigger:
Expresión:
Severidad:
Dependencias:
Periodo evaluado:
Macro utilizada:
Condición de recuperación:
```

No cambiar el trigger antes de entender la expresión completa.

## 7. Identificar una macro heredada

En el host:

```text
Recopilación de datos → Equipos → <HOST> → Macros
```

Activar la visualización de macros heredadas cuando esté disponible.

Buscar la macro mencionada por el trigger.

Ejemplo:

```text
{$ORACLE.PGA.USE.MAX.WARN}
```

Registrar:

```text
Valor heredado:
Plantilla de origen:
Valor propuesto:
Motivo:
```

## 8. Sobrescribir una macro a nivel host

1. Abrir **Macros** del host.
2. Agregar la misma macro heredada.
3. Escribir el valor específico del servidor.
4. Agregar una descripción con motivo y fecha.
5. Guardar.

Ejemplo conceptual:

```text
Macro: {$EJEMPLO.UMBRAL}
Valor: 80
Descripción: Ajustado después de observar comportamiento normal durante 14 días.
```

El valor a nivel host tiene prioridad para ese servidor sin modificar la plantilla.

## 9. Validar el cambio

Después de guardar:

1. Esperar al menos un nuevo intervalo de recopilación.
2. Revisar **Últimos datos**.
3. Revisar **Monitoreo → Problemas**.
4. Confirmar que el trigger evalúa el nuevo valor.
5. Generar una prueba controlada cuando sea seguro.
6. Confirmar recuperación.

Registrar evidencia antes y después.

## 10. Revertir una sobrescritura

Para volver al valor heredado:

1. Abrir macros del host.
2. Eliminar la macro local o restablecer herencia, según la interfaz.
3. Guardar.
4. Confirmar que vuelve a mostrarse el valor de la plantilla.

No borrar una macro sin documentar la razón de la reversión.

## 11. Ajustar intervalos de actualización

Un intervalo menor genera más datos y más carga en:

- Agente.
- Zabbix Server.
- Base de datos de Zabbix.
- Red.
- Sistema monitoreado.

Antes de reducirlo, responder:

```text
¿Con qué rapidez puede cambiar la condición?
¿Cuánto retraso es aceptable?
¿La métrica requiere detalle de segundos o minutos?
¿Cuántos hosts tendrán el mismo intervalo?
```

Para cambios estructurales de intervalos, clonar la plantilla o crear una propia.

## 12. Crear una plantilla propia

Ruta general:

```text
Recopilación de datos → Plantillas → Crear plantilla
```

Definir:

```text
Nombre: Plantilla personalizada - <TECNOLOGIA>
Grupo: Templates/Custom
Descripción: Propósito, propietario y fecha
```

Vincularla al host de prueba antes de usarla en otros servidores.

## 13. Clonar una plantilla oficial

Usar cuando se necesita:

- Deshabilitar elementos no aplicables.
- Excluir consultas por licenciamiento.
- Cambiar descubrimiento.
- Modificar triggers o gráficas.
- Ajustar intervalos para un grupo de servidores.

Nombre recomendado:

```text
Oracle by Zabbix agent 2 - Sin Diagnostics Pack
```

Documentar versión y fecha de la plantilla original.

## 14. Crear una métrica personalizada con `UserParameter`

### Ejemplo de objetivo

Medir si existe un archivo de aplicación.

### Paso 1. Crear archivo de configuración

En Oracle Linux:

```text
/etc/zabbix/zabbix_agent2.d/app_personalizada.conf
```

Comprobar el directorio:

```bash
sudo ls -ld /etc/zabbix/zabbix_agent2.d
```

Crear respaldo si el archivo ya existe:

```bash
sudo cp -a \
  /etc/zabbix/zabbix_agent2.d/app_personalizada.conf \
  /etc/zabbix/zabbix_agent2.d/app_personalizada.conf.respaldo
```

Abrir:

```bash
sudo vi /etc/zabbix/zabbix_agent2.d/app_personalizada.conf
```

Contenido de ejemplo:

```ini
UserParameter=app.archivo.existe,test -f /ruta/controlada/archivo && echo 1 || echo 0
```

### Paso 2. Validar configuración

```bash
sudo /usr/sbin/zabbix_agent2 \
  -c /etc/zabbix/zabbix_agent2.conf -T
```

### Paso 3. Reiniciar

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

### Paso 4. Probar localmente

```bash
zabbix_agent2 -t app.archivo.existe
```

Resultado esperado:

```text
1
```

o:

```text
0
```

### Paso 5. Crear el elemento en Zabbix

En la plantilla personalizada:

```text
Nombre: Aplicación: archivo de control existe
Tipo: Zabbix agent active
Clave: app.archivo.existe
Tipo de información: Numérico sin signo
Intervalo: 1m para prueba
```

### Paso 6. Crear trigger

Ejemplo conceptual:

```text
Problema cuando el último valor sea 0
```

No usar comandos que expongan contraseñas o permitan ejecución arbitraria.

## 15. Métrica personalizada mediante script

Para lógica mayor:

1. Crear un script en una ruta controlada.
2. Asignar propietario y permisos mínimos.
3. Probarlo con el usuario del agente.
4. Hacer que devuelva un único valor fácil de interpretar.
5. Llamarlo desde un `UserParameter`.

Ejemplo de ruta:

```text
/usr/local/lib/zabbix/checks/
```

Evitar:

- Salida extensa.
- Datos sensibles.
- Consultas sin límite.
- Comandos que puedan bloquearse indefinidamente.

## 16. Consultas SQL personalizadas

Antes de crear una consulta:

- Definir propósito.
- Usar usuario de solo lectura.
- Verificar licenciamiento.
- Limitar filas.
- Evitar consultas costosas.
- Probar tiempo de ejecución.
- Documentar plan de reversión.

No reutilizar credenciales de aplicación.

## 17. Elementos calculados

Se usan para obtener un valor a partir de métricas ya almacenadas.

Ejemplos:

```text
Porcentaje calculado
Relación entre usados y disponibles
Suma de varias métricas
```

Ventaja: no realiza una consulta adicional al servidor monitoreado.

## 18. Elementos dependientes

Se usan cuando un elemento maestro obtiene varios datos y los dependientes extraen valores individuales.

Ventajas:

- Menos conexiones.
- Menor carga.
- Reutilización de una respuesta JSON o texto.

## 19. Descubrimiento de bajo nivel

Permite crear automáticamente elementos y triggers para recursos detectados, como:

- Sistemas de archivos.
- Interfaces de red.
- Discos.
- Tablespaces.
- Servicios.

Antes de modificarlo, revisar filtros y macros para evitar crear miles de elementos innecesarios.

## 20. Ficha de parametrización

```text
Fecha:
Responsable:
Equipo:
Ambiente:
Plantilla:
Métrica:
Clave:
Unidad:
Intervalo original:
Intervalo nuevo:
Valor normal observado:
Umbral heredado:
Umbral propuesto:
Periodo del trigger:
Macro:
Motivo:
Riesgo:
Prueba realizada:
Resultado:
Plan de reversión:
Aprobación:
```

## 21. Criterios de aceptación

- La métrica recibe valores recientes.
- La unidad es correcta.
- El valor fue comparado con una fuente independiente.
- El trigger se activa en una prueba controlada.
- El trigger se recupera al normalizar la condición.
- No genera alertas repetitivas sin acción.
- El cambio está documentado.
- Existe plan de reversión.

## 22. Checklist

- [ ] Métrica y unidad comprendidas.
- [ ] Trigger relacionado identificado.
- [ ] Macro heredada localizada.
- [ ] Comportamiento normal observado.
- [ ] Cambio aplicado a nivel host o plantilla propia.
- [ ] Plantilla oficial sin modificaciones directas.
- [ ] Evidencia antes y después.
- [ ] Prueba de problema realizada.
- [ ] Prueba de recuperación realizada.
- [ ] Reversión documentada.
- [ ] Ficha de parametrización completada.

## 23. Referencias

- [Macros definidas por usuario](https://www.zabbix.com/documentation/7.4/en/manual/config/macros/user_macros)
- [Elementos](https://www.zabbix.com/documentation/7.4/en/manual/config/items)
- [Triggers](https://www.zabbix.com/documentation/7.4/en/manual/config/triggers)
- [User parameters](https://www.zabbix.com/documentation/7.4/en/manual/config/items/userparameters)
