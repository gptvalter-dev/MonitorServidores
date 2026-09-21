# Checklist preventivo para despliegues Zabbix sobre Linux

> Propósito: ejecutar esta revisión antes de instalar Zabbix Server o incorporar un nuevo host Linux, Oracle Database o servidor Docker. La intención es detectar problemas de infraestructura antes de comenzar a parametrizar Zabbix.

No marcar un punto como completado por suposición. Cada punto debe tener evidencia: comando, captura, valor observado o responsable que lo confirma.

---

# A. Zabbix Server Linux

## Identidad y sistema operativo

- [ ] Distribución y versión confirmadas.
- [ ] Hostname definitivo confirmado.
- [ ] IP fija o reserva confirmada.
- [ ] Gateway correcto.
- [ ] DNS revisado cuando aplique.
- [ ] Zona horaria correcta.
- [ ] Sincronización NTP/chrony correcta.

Comandos:

```bash
cat /etc/os-release
hostnamectl
ip addr
ip route
timedatectl
chronyc tracking
```

## Capacidad

- [ ] CPU disponible documentada.
- [ ] Memoria disponible documentada.
- [ ] Espacio en filesystem documentado.
- [ ] Espacio para base de datos considerado.
- [ ] Retención de históricos y tendencias considerada.
- [ ] Crecimiento esperado de hosts y métricas considerado.

```bash
lscpu
free -m
df -h
lsblk
```

## Seguridad Linux

- [ ] SELinux revisado y mantenido habilitado cuando sea posible.
- [ ] `firewalld` activo.
- [ ] Puertos necesarios identificados antes de abrirlos.
- [ ] Acceso administrativo restringido.
- [ ] Credenciales predeterminadas serán sustituidas.

```bash
getenforce
firewall-cmd --state
firewall-cmd --list-all
```

## Servicios

- [ ] Motor de base de datos instalado y soportado.
- [ ] Zabbix Server instalado.
- [ ] Frontend instalado.
- [ ] Nginx/Apache instalado según diseño.
- [ ] PHP-FPM instalado cuando aplique.
- [ ] Agent 2 instalado para monitorear el propio servidor.
- [ ] Todos los servicios habilitados al arranque.

Validar después de instalar:

```bash
systemctl is-enabled <servicio>
systemctl is-active <servicio>
```

## Reinicio obligatorio de validación

Antes de considerar terminada la instalación:

- [ ] Reiniciar el servidor Linux.
- [ ] Confirmar que todos los servicios regresan automáticamente.
- [ ] Confirmar que la interfaz web responde.
- [ ] Confirmar que Zabbix Server escucha en `10051/TCP`.
- [ ] Confirmar que Agent 2 funciona después del reinicio.

---

# B. Red antes de instalar cualquier agente

- [ ] IP del Zabbix Server confirmada.
- [ ] Se conoce la red del host monitoreado.
- [ ] Existe ruta entre ambas redes.
- [ ] No existe NAT desconocido.
- [ ] `10051/TCP` es accesible desde el host cuando se usarán comprobaciones activas.
- [ ] `10050/TCP` es accesible desde Zabbix Server cuando se usarán comprobaciones pasivas.
- [ ] Se conoce la IP real que llegará al agente en comprobaciones pasivas.
- [ ] Firewall perimetral y firewall local están alineados.

Desde el host monitoreado:

```bash
ip route get <IP_ZABBIX_SERVER>
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/10051' \
  && echo "CONEXION OK" \
  || echo "SIN CONEXION"
```

Desde Zabbix Server, cuando aplique:

```bash
nc -vz <IP_HOST> 10050
```

Cuando el origen real sea dudoso:

```bash
sudo tcpdump -nni <INTERFAZ> tcp port 10050
```

---

# C. Agent 2 en Linux

Antes de crear el host en Zabbix:

- [ ] Repositorio compatible con la versión de Zabbix.
- [ ] Agent 2 instalado.
- [ ] Archivo original respaldado.
- [ ] `Hostname` definido.
- [ ] `Hostname` coincide exactamente con el nombre técnico del host en Zabbix.
- [ ] `ServerActive` apunta a la IP correcta del Zabbix Server.
- [ ] `Server=` contiene únicamente orígenes autorizados.
- [ ] `ListenPort=10050` cuando se usarán comprobaciones pasivas.
- [ ] Sintaxis validada.
- [ ] Servicio habilitado al arranque.
- [ ] Logs revisados después de reiniciar.

Comandos:

```bash
sudo cp -a /etc/zabbix/zabbix_agent2.conf \
  /etc/zabbix/zabbix_agent2.conf.respaldo

sudo zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
sudo systemctl enable --now zabbix-agent2
sudo systemctl is-active zabbix-agent2
sudo ss -lntp | grep ':10050'
sudo journalctl -u zabbix-agent2 -n 100 --no-pager
```

No continuar con plantillas de aplicación si `Zabbix agent ping` todavía no tiene datos recientes.

---

# D. Plantillas

Antes de vincular una plantilla:

- [ ] Confirmar que corresponde al producto que realmente existe en el host.
- [ ] Confirmar compatibilidad con la versión de Zabbix.
- [ ] Identificar si usa comprobaciones activas, pasivas o ambas.
- [ ] Identificar interfaces requeridas.
- [ ] Revisar macros heredadas.
- [ ] Revisar reglas de descubrimiento.
- [ ] Revisar dependencias externas.
- [ ] Revisar permisos necesarios.
- [ ] Revisar implicaciones de licenciamiento.
- [ ] No modificar una plantilla oficial directamente.
- [ ] No insertar una plantilla de aplicación dentro de una plantilla de sistema operativo sin una decisión de diseño explícita.

Estructura preferida:

```text
Host
├── Linux by Zabbix agent active
├── Docker by Zabbix agent 2        (si aplica)
├── Oracle by Zabbix agent 2        (si aplica)
└── Plantillas propias de aplicación
```

---

# E. Oracle Database

Antes de vincular la plantilla Oracle:

- [ ] Listener funcionando.
- [ ] `SERVICE_NAME` confirmado.
- [ ] CDB/no-CDB confirmado.
- [ ] Usuario de monitoreo dedicado creado.
- [ ] Privilegios mínimos revisados.
- [ ] Licenciamiento Oracle revisado.
- [ ] SQL*Plus directo probado con el usuario de monitoreo.
- [ ] `ORACLE_HOME` confirmado.
- [ ] `libclntsh.so` localizado.
- [ ] Arquitectura 64 bits confirmada.
- [ ] Entorno efectivo de `zabbix-agent2` revisado.
- [ ] Macros configuradas como secreto cuando corresponda.
- [ ] Plantilla Oracle vinculada directamente al host.

Comandos:

```bash
lsnrctl status
sqlplus -L <USUARIO>@//127.0.0.1:1521/<SERVICE_NAME>
find <ORACLE_HOME> -name 'libclntsh.so*'
systemctl show zabbix-agent2 -p Environment
```

Criterio de salida:

```text
Oracle Ping = Up (1)
```

---

# F. Servidor de aplicaciones con Docker

Antes de vincular `Docker by Zabbix agent 2`:

- [ ] Sistema operativo del host identificado.
- [ ] Docker Engine funciona.
- [ ] Inventario de contenedores generado.
- [ ] Aplicación asociada a cada contenedor identificada.
- [ ] Puertos publicados documentados.
- [ ] Redes Docker documentadas.
- [ ] Volúmenes documentados.
- [ ] Política de reinicio documentada.
- [ ] Agent 2 instalado en el host Linux.
- [ ] Acceso de Agent 2 al socket Docker revisado.
- [ ] Contenedores que deben excluirse del monitoreo identificados.
- [ ] Endpoint funcional de cada aplicación crítica identificado.

Comandos:

```bash
docker version
docker info
docker ps -a
docker network ls
docker volume ls
ls -l /var/run/docker.sock
id zabbix
```

Para cada aplicación completar:

```text
Aplicación:
Contenedor:
Imagen:
Puerto interno:
Puerto publicado:
URL de salud:
Endpoint crítico:
Dependencias:
Responsable:
```

Criterio importante:

```text
Contenedor running != aplicación saludable
```

La aplicación debe tener validación propia por puerto, HTTP/HTTPS, código de respuesta o endpoint.

---

# G. Firewall

- [ ] Toda regla necesaria es permanente.
- [ ] No existen aperturas generales innecesarias.
- [ ] Se autoriza únicamente el origen requerido.
- [ ] Se recargó `firewalld` después de crear reglas.
- [ ] Se comprobó que las reglas permanecen después del reload.
- [ ] Se validó de nuevo conectividad después del reload.

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules
```

No considerar resuelta una incidencia de firewall solo porque funcionó antes del `reload`.

---

# H. Métricas y alertas

Antes de modificar un umbral:

- [ ] Nombre exacto de la métrica identificado.
- [ ] Unidad confirmada.
- [ ] Intervalo de actualización confirmado.
- [ ] Expresión del trigger leída.
- [ ] Macro del umbral identificada.
- [ ] Valor normal observado durante un periodo razonable.
- [ ] Pico aislado diferenciado de condición sostenida.
- [ ] Métricas relacionadas revisadas.
- [ ] Arquitectura real comparada contra el supuesto de la plantilla.
- [ ] Motivo de cualquier cambio documentado.

Regla:

```text
No modificar el servidor solo para cerrar una alerta genérica.
Primero comprender la métrica y su contexto.
```

---

# I. Seguridad de operación

- [ ] No guardar secretos reales en GitHub.
- [ ] Usar macros secretas.
- [ ] Aplicar privilegio mínimo.
- [ ] No usar `sudo NOPASSWD: ALL` para el usuario zabbix.
- [ ] No habilitar `AllowKey=system.run[*]` sin justificación.
- [ ] Revisar acceso al socket Docker antes de agregar permisos.
- [ ] Mantener SELinux habilitado y resolver la política correcta.
- [ ] Evaluar TLS entre agentes, proxies y servidor en producción.
- [ ] Sustituir credenciales predeterminadas del frontend.

---

# J. Validación final de cada host

No entregar un host como monitoreado hasta verificar:

- [ ] Host visible en Zabbix.
- [ ] `Zabbix agent ping` reciente.
- [ ] CPU con datos recientes.
- [ ] Memoria con datos recientes.
- [ ] Filesystems con datos recientes.
- [ ] Red con datos recientes.
- [ ] Plantilla específica del servicio funcionando.
- [ ] No existen elementos no soportados sin explicación.
- [ ] Problemas revisados.
- [ ] Zona horaria correcta.
- [ ] Reinicio del agente probado.
- [ ] Reinicio del servidor probado cuando el cambio lo requiere.
- [ ] Evidencia registrada.

---

# K. Secuencia obligatoria de diagnóstico

Cuando algo falle, revisar en este orden:

```text
1. Sistema operativo
2. IP y ruta
3. Firewall
4. Puerto
5. Servicio
6. Configuración de Agent 2
7. Permisos del usuario zabbix
8. Dependencia externa (Oracle, Docker, HTTP, etc.)
9. Plantilla
10. Macros
11. Elemento
12. Trigger
```

Evitar modificar tres componentes a la vez. Hacer un cambio, validar y documentar antes del siguiente.
