# Lecciones aprendidas y prevención para una arquitectura Zabbix sobre Linux

> Objetivo: concentrar los aprendizajes obtenidos durante el laboratorio y convertirlos en controles preventivos para la siguiente etapa, donde se busca que la solución quede basada en Linux.

Este documento no sustituye las guías de instalación. Su función es responder dos preguntas:

1. ¿Qué aprendimos de los problemas ya encontrados?
2. ¿Qué debemos revisar antes de instalar o integrar un nuevo servidor para evitar repetirlos?

---

# 1. Dirección objetivo de la arquitectura

La meta para la siguiente etapa es reducir dependencias de Windows, WSL 2 y Docker Desktop y operar Zabbix principalmente sobre Linux.

Arquitectura objetivo inicial:

```text
Oracle Linux / Linux dedicado
├── Zabbix Server
├── Zabbix Frontend
├── Base de datos de Zabbix
├── Zabbix Agent 2
└── Servicios administrados por systemd

Servidores monitoreados
├── Oracle Linux con Oracle Database
├── Linux con aplicaciones
├── Linux con Docker
└── Otros servidores según inventario
```

Esto no significa que Windows no pueda monitorearse. Significa que el servidor central de monitoreo y la mayor parte de la operación deben quedar en Linux para reducir complejidad de red, NAT, puertos alternos y dependencias de escritorio.

---

# 2. Lecciones aprendidas

## 2.1. No asumir que una IP es alcanzable solo porque pertenece al servidor correcto

Durante el laboratorio se configuró inicialmente una dirección del equipo Windows que no era accesible desde Oracle Linux.

El agente intentaba conectarse a:

```text
<IP_NO_ACCESIBLE>:11051
```

mientras que otra dirección del mismo equipo sí era alcanzable.

Lección:

```text
No configurar ServerActive únicamente por intuición o por la IP que aparece primero.
```

Antes de parametrizar un agente se debe probar desde el propio servidor monitoreado:

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/10051' \
  && echo "CONEXION OK" \
  || echo "SIN CONEXION"
```

En la instalación Linux objetivo se debe preferir una dirección fija, documentada y directamente enrutable.

---

## 2.2. Activo y pasivo son flujos distintos

Se comprobó que un agente podía funcionar en un sentido y fallar en el otro.

Comprobación activa:

```text
Servidor monitoreado ---> Zabbix Server:10051
```

Comprobación pasiva:

```text
Zabbix Server ---> Agent 2:10050
```

Lección:

- `ServerActive=` no sustituye a `Server=`.
- Abrir `10051/TCP` no garantiza que `10050/TCP` funcione.
- Cada flujo debe validarse por separado.

---

## 2.3. El valor de Hostname debe coincidir exactamente

Las comprobaciones activas dependen del nombre técnico del host configurado en Zabbix.

Lección:

Antes de reiniciar Agent 2 validar:

```ini
Hostname=<NOMBRE_EXACTO_DEL_HOST_EN_ZABBIX>
```

No confundir:

- hostname real del sistema operativo;
- nombre visible en Zabbix;
- nombre técnico del host en Zabbix.

---

## 2.4. El origen real de una conexión puede ser distinto al esperado

Con Docker Desktop, WSL 2, NAT y múltiples interfaces se observó que la IP de origen de la comprobación pasiva no siempre coincidía con la que inicialmente se suponía.

Lección:

Cuando una conexión pasiva falla aunque la red parezca correcta, capturar tráfico:

```bash
sudo tcpdump -nni <INTERFAZ> tcp port 10050
```

La IP observada debe coincidir con:

```ini
Server=<IP_ORIGEN_REAL>
```

y con la regla de firewall.

Una arquitectura Linux directa debe reducir este tipo de traducciones y simplificar el diagnóstico.

---

## 2.5. Las reglas temporales de firewall no son suficientes

Durante las pruebas fue necesario confirmar que las reglas sobrevivieran a `firewall-cmd --reload`.

Lección:

Toda regla que forme parte de la solución debe crearse como permanente:

```bash
sudo firewall-cmd --permanent ...
sudo firewall-cmd --reload
```

Después comprobar:

```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules
```

No considerar una incidencia resuelta hasta validar persistencia.

---

## 2.6. Un puerto abierto no significa que la aplicación esté correctamente configurada

Puede existir conectividad TCP y aun así fallar Zabbix por:

- `Hostname` incorrecto;
- `Server=` incorrecto;
- credenciales;
- macros;
- librerías;
- permisos;
- plantilla equivocada.

Lección:

Validar por capas:

```text
1. Red
2. Puerto
3. Servicio
4. Configuración Zabbix
5. Permisos
6. Aplicación o base de datos
7. Métrica final en Zabbix
```

---

## 2.7. No modificar plantillas oficiales directamente

La plantilla Oracle fue relacionada de forma incorrecta con la plantilla Linux durante la exploración.

Lección:

Las plantillas oficiales deben permanecer independientes y reutilizables.

Estructura correcta:

```text
Host Oracle Linux
├── Linux by Zabbix agent active
└── Oracle by Zabbix agent 2
```

No:

```text
Linux by Zabbix agent active
└── Oracle by Zabbix agent 2
```

Cuando sea necesario adaptar una plantilla por licenciamiento o política, crear una copia controlada.

---

## 2.8. Licenciamiento de Oracle debe revisarse antes de habilitar todas las métricas

La integración oficial puede utilizar vistas o capacidades asociadas a opciones licenciadas de Oracle.

En el ambiente evaluado no se cuenta con Diagnostics Pack.

Lección:

Antes de producción:

1. Revisar la versión exacta de la plantilla.
2. Identificar consultas a ASH u otras vistas sujetas a licencia.
3. Clonar la plantilla.
4. Deshabilitar en la copia los elementos no autorizados.
5. Mantener el principio de privilegios mínimos.

No otorgar roles amplios solo para hacer desaparecer errores.

---

## 2.9. Los servicios systemd no heredan necesariamente el entorno del usuario

El error `DPI-1047` apareció aunque Oracle Client existía correctamente.

Causa real: `zabbix-agent2` iniciado por `systemd` no recibía `ORACLE_HOME` y `LD_LIBRARY_PATH` del usuario Oracle.

Lección:

Para integraciones que dependan de variables de entorno se debe revisar el entorno efectivo del servicio:

```bash
systemctl show zabbix-agent2 -p Environment
```

No asumir que un comando que funciona al usuario `oracle` funcionará igual para un servicio administrado por `systemd`.

---

## 2.10. Probar la dependencia directamente antes de culpar a Zabbix

La conexión SQL*Plus directa permitió separar un problema de credenciales de un problema de librerías.

Lección:

Antes de diagnosticar desde Zabbix, probar el componente directamente.

Ejemplos:

```bash
sqlplus -L usuario@//host:1521/servicio
curl -I http://host:puerto/
docker ps
systemctl status <servicio>
ss -lntp
```

---

## 2.11. Los umbrales estándar son un punto de partida, no una verdad del ambiente

El caso de REDO mostró que un trigger genérico puede ser incompatible con una configuración concreta.

Lección:

Nunca modificar infraestructura solo para cerrar una alerta sin comprender antes:

- qué mide;
- unidad;
- intervalo;
- fórmula del trigger;
- arquitectura real;
- comportamiento histórico.

Primero medir. Después parametrizar.

---

## 2.12. La zona horaria afecta la interpretación operativa

Se observó diferencia entre las horas mostradas por las gráficas y la hora local.

Lección:

La zona horaria debe formar parte del checklist inicial del frontend y de los usuarios.

Para el ambiente actual:

```text
America/Mexico_City
```

---

## 2.13. "Running" en Docker no significa que la aplicación esté sana

Para servidores de aplicaciones con varios contenedores se deben separar tres niveles:

```text
Servidor Linux
Docker Engine
Aplicación dentro del contenedor
```

Lección:

Monitorear únicamente el estado del contenedor es insuficiente.

Para cada aplicación importante se debe validar, además:

- puerto;
- HTTP/HTTPS;
- código de respuesta;
- tiempo de respuesta;
- endpoint crítico;
- proceso o servicio interno cuando aplique.

El Agent 2 debe instalarse normalmente en el host Linux. No instalar un agente en cada contenedor salvo que exista una necesidad concreta.

---

## 2.14. En servidores Docker se debe revisar acceso al socket antes de aplicar la plantilla

Para utilizar `Docker by Zabbix agent 2`, el servicio Agent 2 debe poder consultar Docker.

Antes de configurar el host verificar:

```bash
docker version
docker ps
ls -l /var/run/docker.sock
id zabbix
```

Lección:

El problema puede ser de permisos del socket y no de la plantilla.

No ampliar permisos de forma indiscriminada. Documentar qué usuario o grupo obtiene acceso a Docker.

---

## 2.15. Los cambios deben ser reproducibles

Una solución manual no documentada se pierde en la siguiente instalación.

Lección:

Toda corrección debe dejar registrado:

```text
Síntoma
Causa
Archivo o componente
Cambio exacto
Comando de aplicación
Validación
Cómo revertir
Estado
```

---

# 3. Lista preventiva antes de instalar Zabbix Server en Linux

No comenzar la instalación hasta completar esta revisión.

## Sistema operativo

- [ ] Versión exacta de Oracle Linux/Linux documentada.
- [ ] Hostname definitivo configurado.
- [ ] IP fija o reserva definida.
- [ ] DNS directo y reverso revisado cuando aplique.
- [ ] Zona horaria correcta.
- [ ] NTP/chrony sincronizado.
- [ ] Espacio en disco suficiente.
- [ ] Filesystem para base de datos dimensionado.
- [ ] Memoria y CPU dimensionadas.
- [ ] SELinux en modo conocido y documentado.
- [ ] `firewalld` activo y bajo control.

Comandos mínimos:

```bash
cat /etc/os-release
hostnamectl
ip addr
ip route
timedatectl
chronyc tracking
df -h
free -m
getenforce
firewall-cmd --state
```

---

# 4. Lista preventiva de red

Antes de instalar agentes:

- [ ] IP del Zabbix Server confirmada desde cada segmento.
- [ ] Ruta hacia el servidor confirmada.
- [ ] `10051/TCP` accesible para comprobaciones activas.
- [ ] `10050/TCP` accesible para comprobaciones pasivas cuando se utilicen.
- [ ] No existen NAT o balanceadores desconocidos entre servidor y agente.
- [ ] Se conoce la IP real de origen de las comprobaciones pasivas.
- [ ] No se reutilizan puertos alternos sin documentarlos.
- [ ] Firewall de red y firewall local están alineados.

Pruebas mínimas desde el host monitoreado:

```bash
ip route get <IP_ZABBIX_SERVER>
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/<IP_ZABBIX_SERVER>/10051'
```

Prueba desde Zabbix Server al agente cuando aplique:

```bash
nc -vz <IP_HOST> 10050
```

---

# 5. Lista preventiva para Agent 2 en Linux

Antes de vincular plantillas:

- [ ] Agent 2 instalado desde repositorio compatible.
- [ ] Archivo de configuración respaldado.
- [ ] `Hostname` coincide exactamente con Zabbix.
- [ ] `ServerActive` utiliza la IP correcta.
- [ ] `Server=` contiene únicamente orígenes autorizados.
- [ ] Puerto `10050` escucha cuando se usarán comprobaciones pasivas.
- [ ] Servicio habilitado al arranque.
- [ ] Configuración validada antes de reiniciar.
- [ ] Logs revisados después del reinicio.

Validaciones:

```bash
zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
systemctl enable --now zabbix-agent2
systemctl is-active zabbix-agent2
ss -lntp | grep ':10050'
journalctl -u zabbix-agent2 -n 100 --no-pager
```

---

# 6. Lista preventiva para Oracle Database

Antes de vincular `Oracle by Zabbix agent 2`:

- [ ] Confirmar `SERVICE_NAME`.
- [ ] Confirmar si la base es CDB o no-CDB.
- [ ] Confirmar listener.
- [ ] Crear usuario dedicado de monitoreo.
- [ ] Revisar privilegios mínimos.
- [ ] Revisar licenciamiento antes de otorgar accesos.
- [ ] Probar SQL*Plus directamente con el usuario de monitoreo.
- [ ] Confirmar arquitectura 64 bits de Oracle Client.
- [ ] Confirmar `libclntsh.so`.
- [ ] Revisar entorno efectivo de `zabbix-agent2`.
- [ ] Vincular plantilla Oracle directamente al host.
- [ ] Configurar macros como secretos cuando corresponda.

Validaciones mínimas:

```bash
lsnrctl status
sqlplus -L <USUARIO>@//127.0.0.1:1521/<SERVICE_NAME>
find <ORACLE_HOME> -name 'libclntsh.so*'
systemctl show zabbix-agent2 -p Environment
```

---

# 7. Lista preventiva para servidores de aplicaciones con Docker

Antes de aplicar la plantilla Docker:

- [ ] Identificar sistema operativo del host.
- [ ] Confirmar versión de Docker Engine.
- [ ] Obtener inventario con `docker ps -a`.
- [ ] Identificar nombres de contenedores.
- [ ] Identificar aplicación asociada a cada contenedor.
- [ ] Identificar puertos publicados.
- [ ] Identificar redes Docker.
- [ ] Identificar volúmenes.
- [ ] Identificar política de reinicio.
- [ ] Confirmar acceso de Agent 2 al socket Docker.
- [ ] Definir qué contenedores deben monitorearse y cuáles excluirse.
- [ ] Definir validación funcional por aplicación.

Inventario mínimo:

```bash
docker version
docker info
docker ps -a
docker network ls
docker volume ls
```

Para cada aplicación documentar:

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

No considerar una aplicación saludable solo porque el contenedor esté `running`.

---

# 8. Lista preventiva para plantillas

Antes de vincular una plantilla:

- [ ] Confirmar que corresponde a la versión de Zabbix utilizada.
- [ ] Revisar qué interfaces requiere.
- [ ] Revisar macros heredadas.
- [ ] Revisar elementos y reglas de descubrimiento.
- [ ] Revisar permisos externos necesarios.
- [ ] Revisar implicaciones de licencia.
- [ ] No vincular una plantilla de aplicación dentro de una plantilla de sistema operativo sin una razón de diseño explícita.
- [ ] No modificar una plantilla oficial directamente.

Estructura recomendada:

```text
Host
├── Plantilla del sistema operativo
├── Plantilla de Docker, si aplica
├── Plantilla de base de datos, si aplica
└── Plantillas de aplicación específicas
```

---

# 9. Lista preventiva para métricas y triggers

Antes de aceptar una alerta como válida:

- [ ] Identificar la métrica exacta.
- [ ] Confirmar unidad.
- [ ] Confirmar frecuencia de captura.
- [ ] Leer la expresión del trigger.
- [ ] Identificar macro usada como umbral.
- [ ] Comparar con la arquitectura real.
- [ ] Revisar mínimo, promedio, máximo y tendencia.
- [ ] Correlacionar con otras métricas relacionadas.
- [ ] No ajustar el sistema únicamente para cerrar una alerta genérica.
- [ ] Documentar cualquier sobrescritura de macro.

---

# 10. Lista preventiva para seguridad

- [ ] No almacenar contraseñas en texto visible en el repositorio.
- [ ] Usar macros de tipo secreto cuando estén disponibles.
- [ ] Aplicar privilegio mínimo al usuario `zabbix`.
- [ ] No configurar `sudo NOPASSWD: ALL`.
- [ ] No configurar `AllowKey=system.run[*]` sin justificación.
- [ ] Abrir únicamente puertos necesarios.
- [ ] Limitar `Server=` a orígenes autorizados.
- [ ] Evaluar TLS entre servidor, proxy y agentes para producción.
- [ ] Revisar permisos del socket Docker antes de agregar usuarios a grupos privilegiados.
- [ ] Mantener SELinux habilitado y resolver políticas correctamente.

---

# 11. Orden de validación recomendado para cada nuevo host Linux

Seguir siempre este orden:

```text
1. Inventario del servidor
2. Red y rutas
3. Firewall
4. Instalación Agent 2
5. Configuración Agent 2
6. Prueba activa
7. Prueba pasiva si aplica
8. Vinculación de plantilla del SO
9. Plantilla de Docker / Oracle / aplicación
10. Dependencias externas
11. Últimos datos
12. Problemas
13. Gráficas
14. Umbrales
15. Evidencia y documentación
```

No cambiar simultáneamente varias capas. Si una prueba falla, resolverla antes de continuar.

---

# 12. Criterios de aceptación para la futura plataforma Linux

La migración a una plataforma Linux no debe considerarse concluida solo porque la interfaz web abra.

Debe comprobarse:

- [ ] Zabbix Server inicia automáticamente después de reiniciar Linux.
- [ ] Base de datos inicia automáticamente.
- [ ] Frontend inicia automáticamente.
- [ ] Agent 2 inicia automáticamente.
- [ ] Zona horaria y sincronización son correctas.
- [ ] `10051/TCP` está disponible desde redes autorizadas.
- [ ] `10050/TCP` se abre únicamente cuando sea necesario.
- [ ] Firewall persiste después de reinicio.
- [ ] SELinux permanece habilitado.
- [ ] Se monitorea el propio Zabbix Server.
- [ ] Existe respaldo de base de datos y configuración.
- [ ] Existe procedimiento de restauración.
- [ ] Existe procedimiento de actualización.
- [ ] Existen credenciales no predeterminadas.
- [ ] Se han probado al menos un host Linux, un Oracle Database y un servidor Docker.
- [ ] Se documentaron puertos, IP, nombres, plantillas y responsables.

---

# 13. Principio operativo para las siguientes pruebas

A partir de este punto, cada nueva integración debe seguir esta regla:

```text
Primero validar infraestructura.
Después validar agente.
Después validar integración.
Después interpretar la métrica.
Finalmente parametrizar la alerta.
```

Esto evita intentar corregir Zabbix cuando el problema real está en red, permisos, sistema operativo, base de datos, Docker o la propia aplicación.
