# Instalación de Zabbix Agent 2 en Oracle Linux

> Alcance: procedimiento normal para instalar/configurar Agent 2 en Oracle Linux. Las incidencias reales se mantienen en [Agentes Zabbix](../base-conocimiento/agentes-zabbix.md).

## 1. Resultado esperado

- Agent 2 instalado desde repositorio oficial compatible.
- Checks activos funcionales.
- Checks pasivos disponibles solo cuando se requieran.
- Firewall limitado a orígenes autorizados.
- `Zabbix agent ping = Up (1)`.

Antes de iniciar, completar las secciones A-C del [Checklist preventivo Linux](../checklist-preventivo-linux.md).

## 2. Datos requeridos

```text
<IP_ZABBIX_SERVER>                 Destino de checks activos
<PUERTO_ZABBIX_SERVER>             Normalmente 10051
<IP_ORIGEN_COMPROBACION_PASIVA>    Zabbix Server/Proxy que consultará 10050
<HOSTNAME_LINUX>                   Nombre técnico exacto en Zabbix
<IP_HOST_LINUX>                    IP del servidor monitoreado
```

> No copiar los puertos `11050/11051` del laboratorio Windows salvo que exista una razón real. En Linux usar puertos estándar cuando sea posible.

## 3. Revisar el sistema y una posible instalación previa

```bash
cat /etc/os-release
hostnamectl
rpm -qa | grep -i zabbix
systemctl status zabbix-agent2 --no-pager 2>/dev/null || true
```

Si ya existe Agent 2, documentar versión/configuración antes de reinstalar.

## 4. Instalar repositorio y Agent 2

Ejemplo para Oracle Linux 8 y Zabbix 7.4:

```bash
sudo rpm -Uvh \
https://repo.zabbix.com/zabbix/7.4/release/oracle/8/noarch/zabbix-release-latest-7.4.el8.noarch.rpm

sudo dnf clean all
sudo dnf makecache
sudo dnf install -y zabbix-agent2
```

Confirmar:

```bash
/usr/sbin/zabbix_agent2 -V
```

La versión de Agent 2 no debe ser más nueva que la rama del Zabbix Server.

## 5. Rutas principales

```text
Ejecutable:    /usr/sbin/zabbix_agent2
Configuración: /etc/zabbix/zabbix_agent2.conf
Servicio:      zabbix-agent2
Log:           /var/log/zabbix/zabbix_agent2.log
```

Crear respaldo antes de editar:

```bash
sudo cp -a /etc/zabbix/zabbix_agent2.conf \
  /etc/zabbix/zabbix_agent2.conf.respaldo
```

## 6. Configuración base

Editar:

```bash
sudo vi /etc/zabbix/zabbix_agent2.conf
```

### Si se usarán checks activos y pasivos

```ini
Server=<IP_ORIGEN_COMPROBACION_PASIVA>
ServerActive=<IP_ZABBIX_SERVER>:<PUERTO_ZABBIX_SERVER>
Hostname=<HOSTNAME_LINUX>
ListenPort=10050
```

Si existen varios Server/Proxy autorizados para checks pasivos:

```ini
Server=<IP_ORIGEN_1>,<IP_ORIGEN_2>
```

`Server=` debe contener **orígenes reales autorizados**, no direcciones agregadas por costumbre.

### Si solo se utilizarán checks activos

`ServerActive` y `Hostname` son los parámetros principales. No abrir `10050/TCP` si ninguna plantilla/integración requiere checks pasivos.

## 7. Validar antes de reiniciar

```bash
sudo grep -E '^(Server|ServerActive|Hostname|ListenPort)=' \
  /etc/zabbix/zabbix_agent2.conf

sudo /usr/sbin/zabbix_agent2 \
  -c /etc/zabbix/zabbix_agent2.conf -T
```

No reiniciar mientras `-T` muestre errores.

## 8. Habilitar e iniciar

```bash
sudo systemctl enable --now zabbix-agent2
sudo systemctl is-active zabbix-agent2
sudo systemctl is-enabled zabbix-agent2
sudo journalctl -u zabbix-agent2 -n 100 --no-pager
```

Resultados mínimos:

```text
active
enabled
```

## 9. Validar conectividad activa

Desde el host Linux:

```bash
timeout 5 bash -c \
'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/<PUERTO_ZABBIX_SERVER>' \
&& echo 'CONEXION OK' \
|| echo 'SIN CONEXION'
```

No crear el host suponiendo conectividad; probarla primero.

## 10. Checks pasivos: listener y firewall

Solo cuando alguna integración necesite checks pasivos:

```bash
sudo ss -lntp | grep ':10050'
sudo firewall-cmd --get-active-zones
```

Autorizar únicamente el origen requerido:

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="<IP_ORIGEN_COMPROBACION_PASIVA>/32" port port="10050" protocol="tcp" accept'

sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-rich-rules
```

Si el origen real no coincide con lo esperado:

```bash
sudo timeout 60 tcpdump -nni any tcp port 10050 -c 5
```

La IP observada debe estar autorizada tanto en `Server=` como en firewall.

## 11. Crear el host en Zabbix

Ruta:

```text
Recopilación de datos → Equipos → Crear equipo
```

Configuración base:

```text
Nombre del equipo: <HOSTNAME_LINUX>
Grupo: Linux servers
Plantilla: Linux by Zabbix agent active
Estado: Habilitado
```

Una plantilla exclusivamente activa no necesita interfaz Agent para esas métricas. **Agregar interfaz Agent cuando una integración pasiva (por ejemplo Oracle/MongoDB) la requiera.**

## 12. Validación final

En:

```text
Monitoreo → Últimos datos
```

confirmar:

```text
Zabbix agent ping = Up (1)
```

Y revisar:

```bash
sudo journalctl -u zabbix-agent2 --since '10 minutes ago' --no-pager
```

No deben existir errores nuevos de conectividad/configuración sin explicar.

## 13. Siguiente integración

Después de validar Agent 2 base:

- Oracle: [Monitoreo de Oracle Database](04-monitoreo-oracle-database.md)
- Docker/MongoDB: [Monitoreo de MongoDB en Docker](10-monitoreo-mongodb-docker.md)

## 14. Referencias

- Zabbix Agent 2 config: https://www.zabbix.com/documentation/7.4/en/manual/appendix/config/zabbix_agent2
- Checks activos: https://www.zabbix.com/documentation/current/en/manual/guides/monitor_active
- Incidencias: [Agentes Zabbix](../base-conocimiento/agentes-zabbix.md)
