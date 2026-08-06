# Instalación de Zabbix Agent 2 en Oracle Linux

> Alcance: esta guía instala y configura Zabbix Agent 2 en Oracle Linux. No incluye todavía la integración con Oracle Database.

## 1. Objetivo

Al finalizar:

- Agent 2 estará instalado como servicio.
- El agente podrá enviar comprobaciones activas al Zabbix Server.
- El agente podrá aceptar comprobaciones pasivas autorizadas.
- El firewall permitirá únicamente los orígenes necesarios.
- `Zabbix agent ping` aparecerá como `Up (1)`.

## 2. Datos requeridos

Antes de iniciar, definir:

```text
<IP_ZABBIX_SERVER>                    Dirección que responde por el puerto del Zabbix Server
<PUERTO_ZABBIX_SERVER>                Puerto publicado; en el laboratorio es 11051
<IP_ORIGEN_COMPROBACION_PASIVA>       Dirección desde la que realmente llegan las consultas
<HOSTNAME_LINUX>                      Nombre técnico del host en Zabbix
<IP_ORACLE_LINUX>                     Dirección del servidor monitoreado
<INTERFAZ_RED>                        Ejemplo: ens192, eth0 o enp0s3
```

## 3. Entrar al servidor

Conectarse por SSH con un usuario autorizado.

Confirmar el sistema operativo:

```bash
cat /etc/os-release
```

Confirmar la versión:

```bash
hostnamectl
```

## 4. Obtener privilegios

Los comandos de instalación requieren `root` o `sudo`.

Comprobar:

```bash
sudo -v
```

Si el comando solicita contraseña y no se cuenta con permisos, detener el procedimiento y solicitar autorización.

## 5. Respaldar instalaciones manuales anteriores

Si existe una instalación anterior bajo `/opt/zabbix`, respaldar la configuración:

```bash
sudo cp -a /opt/zabbix/conf/zabbix_agentd.conf \
  /root/zabbix_agentd.conf.opt.backup
```

Si el archivo no existe, el comando puede omitirse.

## 6. Instalar el repositorio oficial

Ejecutar:

```bash
sudo rpm -Uvh \
https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/zabbix-release-latest-7.4.el8.noarch.rpm
```

Actualizar metadatos:

```bash
sudo dnf clean all
sudo dnf makecache
```

## 7. Instalar Agent 2

```bash
sudo dnf install -y zabbix-agent2
```

Confirmar versión:

```bash
/usr/sbin/zabbix_agent2 -V
```

## 8. Rutas principales

```text
Ejecutable:    /usr/sbin/zabbix_agent2
Configuración: /etc/zabbix/zabbix_agent2.conf
Servicio:      zabbix-agent2
Log:           /var/log/zabbix/zabbix_agent2.log
```

Comprobar el archivo:

```bash
sudo ls -l /etc/zabbix/zabbix_agent2.conf
```

## 9. Crear respaldo

```bash
sudo cp -a /etc/zabbix/zabbix_agent2.conf \
  /etc/zabbix/zabbix_agent2.conf.respaldo
```

Confirmar:

```bash
sudo ls -l /etc/zabbix/zabbix_agent2.conf*
```

## 10. Abrir el archivo

```bash
sudo vi /etc/zabbix/zabbix_agent2.conf
```

### Comandos mínimos de `vi`

```text
i       iniciar edición
Esc     terminar edición
:wq     guardar y salir
:q!     salir sin guardar
/Texto  buscar Texto
n       ir a la siguiente coincidencia
```

## 11. Configuración base

Configurar líneas activas:

```ini
Server=<IP_ZABBIX_SERVER>,<IP_ORIGEN_COMPROBACION_PASIVA>
ServerActive=<IP_ZABBIX_SERVER>:<PUERTO_ZABBIX_SERVER>
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

### Qué controla cada línea

| Parámetro | Función |
|---|---|
| `Server` | Autoriza los orígenes de consultas pasivas |
| `ServerActive` | Define la dirección donde el agente solicita y envía comprobaciones activas |
| `Hostname` | Debe coincidir exactamente con el nombre técnico del host en Zabbix |
| `ListenPort` | Puerto local de consultas pasivas |

Agent 2 puede utilizar comprobaciones activas y pasivas simultáneamente.

## 12. Validar líneas guardadas

```bash
sudo grep -E '^(Server|ServerActive|Hostname|ListenPort)=' \
  /etc/zabbix/zabbix_agent2.conf
```

No deben existir líneas duplicadas activas para el mismo parámetro.

## 13. Validar sintaxis

```bash
sudo /usr/sbin/zabbix_agent2 \
  -c /etc/zabbix/zabbix_agent2.conf -T
```

No continuar si aparece un error.

## 14. Habilitar e iniciar el servicio

```bash
sudo systemctl enable --now zabbix-agent2
```

Validar:

```bash
sudo systemctl status zabbix-agent2 --no-pager
sudo systemctl is-active zabbix-agent2
sudo systemctl is-enabled zabbix-agent2
```

Resultados esperados:

```text
active
enabled
```

## 15. Confirmar puerto pasivo

```bash
sudo ss -lntp | grep ':10050'
```

Resultado esperado: un listener asociado a `zabbix_agent2`.

## 16. Crear el host en Zabbix

En la consola web:

```text
Recopilación de datos → Equipos → Crear equipo
```

Configurar:

```text
Nombre del equipo: <HOSTNAME_LINUX>
Nombre visible: descripción amigable opcional
Grupo: Linux servers
Plantilla: Linux by Zabbix agent active
Estado: Habilitado
```

La plantilla activa no requiere inicialmente una interfaz de agente.

## 17. Probar conectividad activa

Desde Oracle Linux:

```bash
timeout 5 bash -c \
'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/<PUERTO_ZABBIX_SERVER>' \
&& echo 'CONEXION OK' \
|| echo 'SIN CONEXION'
```

Si existen varias direcciones posibles:

```bash
for IP in <IP_1> <IP_2>; do
  echo "Probando $IP:<PUERTO_ZABBIX_SERVER>"
  timeout 5 bash -c \
  "cat < /dev/null > /dev/tcp/$IP/<PUERTO_ZABBIX_SERVER>" \
  && echo 'CONEXION OK' \
  || echo 'SIN CONEXION'
done
```

Usar en `ServerActive` únicamente una dirección realmente accesible.

## 18. Hallazgo del laboratorio: dirección incorrecta

Síntomas:

```text
cannot connect ... i/o timeout
cannot connect ... no route to host
active check configuration update ... started to fail
```

La dirección configurada inicialmente no respondía. Otra dirección del mismo equipo sí aceptaba conexiones al puerto `11051`.

Después de corregir `ServerActive`:

```bash
sudo systemctl restart zabbix-agent2
```

Validar errores recientes:

```bash
sudo journalctl -u zabbix-agent2 \
  --since '10 minutes ago' --no-pager |
grep -Ei \
'active check configuration|cannot connect|host .*not found|no active checks|failed to send|no route'
```

Sin salida significa que no se encontraron esos errores durante el periodo revisado.

## 19. Configurar firewall para consultas pasivas

Comprobar `firewalld`:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
```

Agregar una regla permanente para el origen autorizado:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="<IP_ORIGEN_COMPROBACION_PASIVA>/32" port port="10050" protocol="tcp" accept'
```

Recargar:

```bash
sudo firewall-cmd --reload
```

Validar:

```bash
sudo firewall-cmd --zone=public --list-rich-rules
```

No abrir `10050` para toda la red si no existe una justificación.

## 20. Identificar el origen real de consultas pasivas

Cuando Zabbix muestra timeout o rechazo, capturar tráfico:

```bash
sudo timeout 60 tcpdump -nni <INTERFAZ_RED> tcp port 10050 -c 3
```

La IP observada debe aparecer en:

```ini
Server=<IP_OBSERVADA>
```

También debe estar autorizada en `firewalld`.

## 21. Reiniciar después de cambios

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl is-active zabbix-agent2
```

## 22. Validar en Zabbix

Abrir:

```text
Monitoreo → Últimos datos
```

Buscar:

```text
Zabbix agent ping
```

Resultado esperado:

```text
Up (1)
```

La antigüedad debe ser de segundos o pocos minutos.

## 23. Revisar logs

Últimas líneas:

```bash
sudo tail -n 50 /var/log/zabbix/zabbix_agent2.log
```

Registro del servicio:

```bash
sudo journalctl -u zabbix-agent2 -n 50 --no-pager
```

## 24. Checklist

- [ ] Repositorio Zabbix instalado.
- [ ] Agent 2 instalado.
- [ ] Archivo respaldado.
- [ ] `Server` configurado con orígenes reales.
- [ ] `ServerActive` apunta a una dirección accesible.
- [ ] `Hostname` coincide con Zabbix.
- [ ] Sintaxis validada.
- [ ] Servicio activo y habilitado.
- [ ] Puerto `10050` escuchando.
- [ ] Regla de firewall permanente.
- [ ] Host creado con plantilla activa.
- [ ] `Zabbix agent ping = Up (1)`.
- [ ] No existen errores recientes de conexión activa.

## 25. Referencias

- [Comprobaciones activas](https://www.zabbix.com/documentation/current/en/manual/guides/monitor_active)
- [Configuración de Agent 2](https://www.zabbix.com/documentation/7.4/en/manual/appendix/config/zabbix_agent2)
- [Incidencias de agentes](../base-conocimiento/agentes-zabbix.md)
