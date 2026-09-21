# Configuración de alertas y notificaciones en Zabbix

> Estado: **ciclo Trigger → Problema → Recuperación validado en laboratorio con MongoDB; envío por medio externo (correo/webhook) todavía pendiente**.

## 1. Objetivo

Completar el flujo:

```text
Métrica → Trigger → Problema → Acción → Notificación → Recuperación → Notificación de recuperación
```

Ya se validó que Zabbix crea y cierra automáticamente un problema al detener/iniciar un MongoDB de laboratorio. Falta validar el canal de notificación.

## 2. Datos a definir

```text
Medio:
Servidor/integración:
Puerto:
Cifrado:
Remitente:
Destinatarios:
Severidades:
Horario:
Reintentos:
Escalamiento:
Responsable:
```

No versionar contraseñas ni tokens.

## 3. Configurar el medio

Ruta:

```text
Alertas → Tipos de medios
```

Para correo se requieren normalmente SMTP, puerto, seguridad, autenticación y remitente. Para webhook se requieren los parámetros propios de la integración.

Ejecutar primero la **prueba del medio** desde Zabbix y conservar evidencia.

## 4. Configurar destinatario

En el usuario:

1. Abrir **Medios**.
2. Agregar el medio.
3. Definir destino.
4. Seleccionar severidades.
5. Definir periodo de actividad.
6. Guardar.

También confirmar que el usuario tenga permisos sobre los grupos de hosts involucrados.

## 5. Crear la acción

Ruta:

```text
Alertas → Acciones → Acciones de triggers
```

Para la primera prueba limitar la acción al grupo/host de laboratorio.

Mensaje mínimo de problema:

```text
Problema: {EVENT.NAME}
Equipo: {HOST.NAME}
Severidad: {EVENT.SEVERITY}
Inicio: {EVENT.DATE} {EVENT.TIME}
Valor: {ITEM.LASTVALUE}
Evento: {EVENT.ID}
```

Configurar también una operación de recuperación:

```text
Recuperado: {EVENT.NAME}
Equipo: {HOST.NAME}
Duración: {EVENT.DURATION}
Recuperación: {EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}
Evento: {EVENT.ID}
```

## 6. Severidad y ruido

Antes de ampliar las notificaciones:

- revisar triggers heredados;
- resolver elementos `No soportada` relevantes;
- revisar dependencias;
- validar umbrales;
- diferenciar picos aislados de condiciones sostenidas;
- asegurar que cada alerta tenga una acción operativa definida.

No enviar a una guardia alertas que todavía estén en exploración.

## 7. Prueba controlada

El ciclo de eventos ya fue validado con MongoDB de laboratorio:

```text
docker stop <MONGO_CONTAINER>
→ PROBLEMA: Connection to MongoDB is unavailable

docker start <MONGO_CONTAINER>
→ RESUELTO automáticamente
```

Para validar notificaciones se puede reutilizar una condición reversible de laboratorio, pero **no detener servicios productivos**.

Criterios:

- [ ] aparece el problema;
- [ ] se envía notificación;
- [ ] destinatario la recibe;
- [ ] se restaura la condición;
- [ ] Zabbix cierra el problema;
- [ ] se envía recuperación;
- [ ] destinatario recibe recuperación.

## 8. Evidencia

```text
Fecha/hora:
Host:
Trigger:
Severidad:
Evento:
Medio:
Destinatario:
Hora de envío:
Hora de recepción:
Problema recibido: Sí/No
Recuperación recibida: Sí/No
Duración:
Errores:
Resultado:
```

## 9. Escalamiento

Solo después de validar el aviso básico:

```text
Paso 1: operador
Paso 2: responsable técnico
Paso 3: coordinación/guardia
```

Cada paso debe definir tiempo, destinatario, horario y condición de detención.

## 10. Diagnóstico rápido

| Síntoma | Revisar |
|---|---|
| Prueba de medio falla | SMTP/webhook, credenciales, red, TLS |
| Medio funciona pero acción no | Condiciones y operaciones |
| Usuario no recibe | Medio activo, horario, severidad y permisos |
| Llega problema pero no recuperación | Operación de recuperación |
| Demasiados mensajes | Trigger, umbral, dependencias y escalamiento |
| Hora incorrecta | Perfil/frontend/NTP |

## 11. Estado pendiente

- [ ] Seleccionar medio del laboratorio.
- [ ] Probar medio.
- [ ] Asociar destinatario.
- [ ] Crear acción limitada a laboratorio.
- [ ] Recibir aviso de problema.
- [ ] Recibir aviso de recuperación.
- [ ] Documentar evidencia.
- [ ] Diseñar escalamiento productivo.

## 12. Referencias

- https://www.zabbix.com/documentation/7.4/en/manual/config/notifications/media
- https://www.zabbix.com/documentation/7.4/en/manual/config/notifications/action
- https://www.zabbix.com/documentation/7.4/en/manual/config/users_and_usergroups/user
