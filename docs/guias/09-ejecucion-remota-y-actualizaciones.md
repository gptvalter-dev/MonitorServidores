# Ejecución remota y actualizaciones desde Zabbix

> Estado: **capacidad confirmada en documentación oficial; pendiente de validación práctica en el laboratorio**.

## 1. Objetivo

Determinar hasta qué punto Zabbix puede utilizarse no solo para monitorear equipos, sino también para ejecutar acciones remotas sobre ellos.

La conclusión es:

- Zabbix **sí puede ejecutar comandos y scripts remotos** de forma manual o automática.
- Con esos scripts es técnicamente posible iniciar actualizaciones de paquetes, servicios o aplicaciones en los equipos monitoreados.
- Zabbix **no es una plataforma especializada de gestión de parches**. No sustituye herramientas como WSUS, SCCM/MECM, Intune, Ansible, Red Hat Satellite u Oracle Linux Manager.

## 2. Formas de ejecución remota disponibles

Zabbix 7.4 permite definir scripts globales que pueden ejecutarse:

1. Manualmente desde el menú de un host.
2. Manualmente desde el menú de un evento.
3. Automáticamente como operación de una acción cuando se cumple una condición.

Los tipos de script disponibles incluyen:

- Script ejecutado mediante Zabbix Agent.
- SSH.
- Telnet.
- IPMI.
- Webhook.

Para actualizaciones de sistema operativo, las opciones más relevantes son:

```text
Zabbix Agent
SSH
```

## 3. Qué significa esto para actualizaciones

### Oracle Linux

Podría definirse un script controlado que ejecute, por ejemplo:

```bash
/usr/bin/dnf update <PAQUETE> -y
```

También podría utilizarse un script local previamente probado:

```bash
/usr/local/sbin/actualiza_paquete_controlado.sh
```

Zabbix únicamente ordenaría su ejecución. El proceso real de actualización lo realiza el sistema operativo.

### Windows

El mismo concepto puede aplicarse mediante PowerShell, por ejemplo invocando un script previamente preparado y probado:

```powershell
C:\Scripts\Actualizar-Aplicacion.ps1
```

La estrategia recomendada es que Zabbix invoque un script controlado en lugar de colocar una secuencia extensa de comandos directamente en la interfaz.

## 4. Limitación importante

Zabbix no proporciona por sí mismo un ciclo completo de administración de parches como:

```text
Descubrir parches disponibles
→ Aprobar parches
→ Crear grupos o anillos de despliegue
→ Programar ventanas de mantenimiento
→ Descargar desde un catálogo central
→ Instalar por lotes
→ Controlar reinicios
→ Verificar cumplimiento de parches
→ Reintentar equipos fallidos
→ Generar inventario de cumplimiento
```

Estas funciones deben cubrirse mediante scripts propios o herramientas especializadas.

Por lo tanto, Zabbix debe considerarse principalmente como:

```text
Monitoreo + detección + disparo de una acción remota controlada
```

no como un gestor completo de actualizaciones.

## 5. Seguridad: los comandos remotos están deshabilitados por defecto

Cuando el comando debe ejecutarse directamente mediante Zabbix Agent o Agent 2, `system.run[]` está deshabilitado por defecto.

En el equipo monitoreado debe autorizarse explícitamente el comando mediante `AllowKey`.

Ejemplo conceptual:

```ini
AllowKey=system.run[/usr/local/sbin/actualiza_paquete_controlado.sh,*]
```

No se recomienda abrir de forma general:

```ini
AllowKey=system.run[*]
```

porque permitiría ejecutar un rango mucho mayor de comandos remotos.

La recomendación es autorizar solamente scripts o comandos específicos.

## 6. Permisos del sistema operativo

Los comandos se ejecutan con el usuario bajo el cual opera el componente Zabbix.

En Linux normalmente será el usuario:

```text
zabbix
```

Ese usuario no tiene permisos administrativos por defecto.

Si se necesita ejecutar una tarea privilegiada, debe configurarse `sudo` de forma restrictiva.

Ejemplo recomendado:

```text
zabbix ALL=(root) NOPASSWD: /usr/local/sbin/actualiza_paquete_controlado.sh
```

Evitar reglas amplias como:

```text
zabbix ALL=(ALL) NOPASSWD: ALL
```

En Windows debe revisarse la cuenta con la que se ejecuta el servicio Zabbix Agent 2 y los privilegios efectivos del script.

## 7. Ejecución manual desde Zabbix

Según la documentación oficial, un script con alcance **Manual host action** puede aparecer en el menú contextual del host.

Flujo general:

```text
Alertas → Scripts
→ Crear script
→ Scope: Manual host action
→ Type: Script o SSH
→ Configurar comando
→ Restringir grupo de hosts
→ Restringir grupo de usuarios
→ Definir permisos necesarios
→ Guardar
```

Después puede ejecutarse desde el menú del host en secciones como:

```text
Monitoreo → Últimos datos
Monitoreo → Problemas
Monitoreo → Hosts
```

Al abrir el menú del host aparecerá la sección **Scripts** cuando exista un script aplicable.

## 8. Ejecución automática

Zabbix también puede ejecutar el script como operación de una acción.

Ejemplo conceptual:

```text
Problema detectado
→ Se cumple un trigger
→ Zabbix ejecuta una acción
→ La acción llama un script remoto
→ El script intenta corregir la condición
```

La documentación oficial menciona ejemplos de remediación como:

- Reiniciar una aplicación que dejó de responder.
- Reiniciar un servidor mediante IPMI.
- Liberar espacio en disco.
- Ejecutar acciones de infraestructura ante falta de recursos.

Para actualizaciones de software se recomienda inicialmente **ejecución manual**, no automática, hasta disponer de controles, pruebas y recuperación ante fallos.

## 9. Riesgos al utilizar Zabbix para actualizaciones

Antes de ejecutar una actualización deben considerarse:

- La actualización puede reiniciar servicios.
- Puede cortar la comunicación con el propio Zabbix Agent.
- Puede requerir reinicio del sistema operativo.
- Puede romper dependencias de una aplicación.
- El comando puede tardar más que los tiempos de espera habituales.
- El resultado mostrado en Zabbix no sustituye un log detallado del proceso.
- Zabbix no conserva automáticamente toda la salida extendida del script.

Para una operación real conviene que el script genere su propio archivo de log.

Ejemplo:

```text
/var/log/actualizaciones-zabbix.log
```

## 10. Diseño recomendado para este proyecto

Para evaluar esta capacidad sin convertir Zabbix en una herramienta de parchado general, se propone:

```text
Zabbix detecta o el operador decide
        ↓
Script manual desde el host
        ↓
Script local controlado
        ↓
Actualización de un paquete o aplicación de prueba
        ↓
Guardar log
        ↓
Zabbix valida nuevamente servicio, versión y disponibilidad
```

La primera prueba debe realizarse sobre un paquete o aplicación no crítica del laboratorio.

## 11. Criterios de aceptación para la prueba

- [ ] Script creado en Zabbix.
- [ ] Disponible solamente para el grupo de hosts de laboratorio.
- [ ] Disponible solamente para usuarios autorizados.
- [ ] `AllowKey` restringido a un script específico cuando se utilice Agent 2.
- [ ] Permisos `sudo` limitados al script necesario.
- [ ] Ejecución manual confirmada.
- [ ] Log del script generado en el equipo remoto.
- [ ] Código de salida revisado.
- [ ] Servicio continúa operativo después de la actualización.
- [ ] Zabbix vuelve a recopilar métricas después de la ejecución.
- [ ] Procedimiento de reversión documentado.

## 12. Conclusión

Zabbix puede iniciar una actualización remota porque dispone de scripts, comandos remotos y acciones. Sin embargo, esta capacidad debe considerarse una función de **remediación/orquestación básica**.

Para una administración formal y masiva de parches, Zabbix debería integrarse o coexistir con una herramienta especializada.

## 13. Referencias oficiales

- [Scripts globales en Zabbix 7.4](https://www.zabbix.com/documentation/7.4/es/manual/web_interface/frontend_sections/alerts/scripts)
- [Comandos remotos](https://www.zabbix.com/documentation/current/es/manual/config/notifications/action/operation/remote_command)
- [Restricción de comprobaciones y AllowKey](https://www.zabbix.com/documentation/7.4/en/manual/config/items/restrict_checks)
- [Ejecución de comandos](https://www.zabbix.com/documentation/current/es/manual/appendix/command_execution)
- [Menú del host](https://www.zabbix.com/documentation/7.4/en/manual/web_interface/menu/host_menu)
