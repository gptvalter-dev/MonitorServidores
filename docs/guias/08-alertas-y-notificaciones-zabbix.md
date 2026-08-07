# Configuración de alertas y notificaciones en Zabbix

> Estado: estructura preparada. Esta parte todavía debe validarse en el laboratorio con una prueba real de problema y recuperación.

## 1. Objetivo

Configurar un flujo completo:

```text
Métrica → Trigger → Problema → Acción → Notificación → Recuperación
```

El laboratorio se considerará validado cuando se reciba:

1. Un aviso de problema.
2. Un aviso de recuperación.

## 2. Conceptos

### Métrica

Valor recopilado, por ejemplo CPU, espacio libre o disponibilidad.

### Trigger

Condición que transforma un valor en problema.

### Problema

Evento visible en **Monitoreo → Problemas**.

### Medio

Canal de entrega, por ejemplo correo electrónico.

### Usuario

Persona que recibirá avisos.

### Grupo de usuarios

Conjunto de destinatarios.

### Acción

Regla que decide cuándo, a quién y cómo enviar la notificación.

### Operación de recuperación

Mensaje enviado cuando la condición vuelve a la normalidad.

## 3. Datos que deben definirse

```text
Medio:
Servidor SMTP o integración:
Puerto:
Cifrado:
Cuenta remitente:
Destinatarios:
Severidades:
Horario:
Tiempo de reintento:
Escalamiento:
Responsable:
```

No almacenar contraseñas reales en este repositorio.

## 4. Crear o validar un usuario

Ruta general:

```text
Usuarios → Usuarios
```

1. Crear o editar un usuario.
2. Asignarlo al grupo correspondiente.
3. Confirmar permisos de lectura sobre los grupos de hosts.
4. Guardar.

Un usuario sin permisos sobre el host puede no recibir información útil del problema.

## 5. Configurar un medio

Ruta general:

```text
Alertas → Tipos de medios
```

Para correo electrónico se necesita:

```text
Servidor SMTP
Puerto
HELO
Correo remitente
Seguridad de conexión
Autenticación
Usuario
Contraseña
```

Antes de utilizarlo en una acción, ejecutar la prueba disponible en la interfaz y conservar evidencia del resultado.

## 6. Asociar el medio al usuario

En el usuario:

1. Abrir la pestaña **Medios**.
2. Agregar el tipo de medio.
3. Escribir el destino.
4. Seleccionar severidades.
5. Definir el periodo de actividad.
6. Habilitarlo.
7. Guardar.

Ejemplo de periodo permanente:

```text
1-7,00:00-24:00
```

Debe adaptarse a las guardias y horarios reales.

## 7. Crear una acción

Ruta general:

```text
Alertas → Acciones → Acciones de triggers
```

Definir:

```text
Nombre:
Condiciones:
Operaciones:
Operaciones de recuperación:
Operaciones de actualización:
```

### Condiciones iniciales sugeridas para laboratorio

- Grupo de hosts igual al grupo de prueba.
- Severidad mayor o igual a advertencia.
- No incluir producción durante la primera prueba.

## 8. Configurar la operación de problema

Seleccionar:

- Usuario o grupo destinatario.
- Medio.
- Paso inicial.
- Duración de pasos.
- Mensaje.

Mensaje mínimo:

```text
Problema: {EVENT.NAME}
Equipo: {HOST.NAME}
Severidad: {EVENT.SEVERITY}
Inicio: {EVENT.DATE} {EVENT.TIME}
Valor: {ITEM.LASTVALUE}
Evento: {EVENT.ID}
```

## 9. Configurar recuperación

Agregar una operación de recuperación con un mensaje distinto:

```text
Recuperado: {EVENT.NAME}
Equipo: {HOST.NAME}
Duración: {EVENT.DURATION}
Recuperación: {EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}
Evento: {EVENT.ID}
```

La recuperación es obligatoria para cerrar el ciclo operativo.

## 10. Severidades

Zabbix maneja niveles como:

```text
No clasificado
Información
Advertencia
Promedio
Alta
Desastre
```

Antes de notificar, definir qué significa cada severidad para la organización.

Ejemplo conceptual:

| Severidad | Tratamiento |
|---|---|
| Información | Registro sin guardia inmediata |
| Advertencia | Revisar dentro del horario operativo |
| Promedio | Atención prioritaria |
| Alta | Escalamiento inmediato |
| Desastre | Incidente crítico |

## 11. Evitar ruido

Antes de habilitar una acción amplia:

- Revisar triggers heredados.
- Corregir zona horaria.
- Ajustar umbrales genéricos.
- Confirmar dependencias.
- Evitar avisos por cada fluctuación.
- Usar periodos mínimos cuando corresponda.
- Configurar recuperación.

## 12. Prueba controlada sugerida

Usar un host de laboratorio y una condición reversible.

Ejemplo:

1. Crear una métrica personalizada simple.
2. Crear trigger cuando el valor sea `0`.
3. Cambiar la condición para producir `0`.
4. Confirmar problema en Zabbix.
5. Confirmar recepción de aviso.
6. Restaurar la condición a `1`.
7. Confirmar recuperación.
8. Confirmar aviso de recuperación.

No detener servicios productivos para probar notificaciones.

## 13. Evidencia requerida

```text
Fecha y hora:
Host:
Trigger:
Severidad:
Evento:
Destinatario:
Medio:
Hora de envío:
Hora de recepción:
Mensaje de problema recibido: Sí/No
Mensaje de recuperación recibido: Sí/No
Tiempo total:
Errores:
Resultado:
```

## 14. Escalamiento

Después de validar el aviso básico, definir:

```text
Paso 1: operador
Paso 2: responsable técnico
Paso 3: coordinación o guardia
```

Cada paso debe tener:

- Tiempo de espera.
- Destinatario.
- Condición de detención.
- Horario.
- Mensaje.

No escalar automáticamente sin validar primero que la alerta sea accionable.

## 15. Diagnóstico

| Síntoma | Revisión |
|---|---|
| No llega correo | Probar tipo de medio y revisar SMTP |
| Prueba de medio funciona, acción no | Revisar condiciones y operaciones |
| Usuario no recibe | Revisar medio activo, severidad, horario y permisos |
| Llega problema pero no recuperación | Agregar operación de recuperación |
| Demasiados mensajes | Revisar trigger, umbral, dependencias y escalamiento |
| Hora incorrecta | Revisar zona horaria del perfil y frontend |

## 16. Checklist

- [ ] Tipo de medio configurado.
- [ ] Prueba del medio exitosa.
- [ ] Usuario con permisos.
- [ ] Destino asociado al usuario.
- [ ] Severidades definidas.
- [ ] Horario definido.
- [ ] Acción limitada al laboratorio.
- [ ] Mensaje de problema configurado.
- [ ] Mensaje de recuperación configurado.
- [ ] Problema controlado generado.
- [ ] Aviso de problema recibido.
- [ ] Recuperación generada.
- [ ] Aviso de recuperación recibido.
- [ ] Evidencia documentada.
- [ ] Escalamiento pendiente o validado.

## 17. Referencias

- [Tipos de medios](https://www.zabbix.com/documentation/7.4/en/manual/config/notifications/media)
- [Acciones](https://www.zabbix.com/documentation/7.4/en/manual/config/notifications/action)
- [Usuarios y medios](https://www.zabbix.com/documentation/7.4/en/manual/config/users_and_usergroups/user)
