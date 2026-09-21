# Lecciones aprendidas y prevención para una arquitectura Zabbix sobre Linux

> Propósito: conservar **principios transversales** aprendidos durante el laboratorio. Los comandos y controles ejecutables están centralizados en [Checklist preventivo Linux](checklist-preventivo-linux.md); las incidencias concretas están en [Base de conocimiento](base-conocimiento/README.md).

## 1. Dirección objetivo

La plataforma central debe simplificarse hacia Linux/Oracle Linux y reducir dependencias de Windows, WSL 2, NAT de escritorio y puertos alternos.

```text
Zabbix Server Linux
├── Base de datos
├── Zabbix Server
├── Frontend
└── Agent 2

Servidores monitoreados
├── Linux / Oracle Linux
├── Oracle Database
└── Linux + Docker
    ├── aplicaciones
    └── MongoDB
```

Windows continúa siendo un sistema monitoreable y un laboratorio válido; no es la arquitectura objetivo del servidor central.

## 2. Lecciones transversales

### 2.1. Validar la ruta real de red, no la IP que parece correcta

Un equipo puede tener varias interfaces, NAT o rutas diferentes. Antes de usar `ServerActive`, `Server=` o una interfaz Zabbix, probar conectividad desde el origen real.

**Principio:** una IP correcta administrativamente no es necesariamente una IP alcanzable desde el componente que hará la conexión.

### 2.2. Checks activos y pasivos son flujos independientes

```text
Activo:  host monitoreado -> Zabbix Server:10051
Pasivo:  Zabbix Server/Proxy -> Agent 2:10050
```

`ServerActive=` y `Server=` resuelven problemas distintos. Cada flujo debe validarse por separado.

### 2.3. El `Hostname` técnico debe coincidir exactamente

En checks activos, el nombre configurado por el Agent 2 debe coincidir con el nombre técnico del host en Zabbix. No confundirlo con el nombre visible del equipo ni con el hostname del sistema operativo si se usa una convención diferente.

### 2.4. Diagnosticar por capas

Orden preferido:

```text
Sistema operativo
  -> red/ruta
  -> firewall
  -> puerto
  -> servicio
  -> configuración Agent 2
  -> permisos
  -> dependencia externa
  -> plantilla/macros
  -> item
  -> trigger
```

No modificar tres capas a la vez. Hacer un cambio, validar, documentar y continuar.

### 2.5. Un puerto abierto no demuestra que la integración funciona

Conectividad TCP solo valida transporte. Todavía pueden fallar `Hostname`, permisos, credenciales, macros, librerías, plugins, JSONPath o la propia aplicación.

### 2.6. Validar configuración antes de reiniciar servicios

Agent 2 puede dejar de iniciar por una configuración o plugin ajeno al cambio actual. En el laboratorio un plugin NVIDIA defectuoso bloqueó el arranque completo del Agent 2.

**Principio:** validar sintaxis/configuración antes de reiniciar y revisar logs inmediatamente después.

La incidencia está documentada en [Agentes Zabbix](base-conocimiento/agentes-zabbix.md).

### 2.7. Las reglas de firewall deben sobrevivir al reload/reinicio

Una apertura temporal no es una solución terminada. Toda regla necesaria debe ser mínima, persistente y probada después de recargar el firewall.

### 2.8. No modificar plantillas oficiales directamente en producción

Las plantillas oficiales deben conservarse como referencia actualizable. Si Oracle, MongoDB u otra integración requiere adaptar items, permisos, licenciamiento o JSONPath, crear una copia controlada con nombre y versión propios.

### 2.9. Una plantilla compatible con Zabbix no garantiza compatibilidad perfecta con cualquier versión de la aplicación

MongoDB 8.2.x demostró que una plantilla de Zabbix 7.4 puede contener métricas diseñadas/probadas con versiones anteriores del producto.

**Principio:** revisar la versión exacta de la aplicación, las versiones probadas por la plantilla y todo elemento `No soportada` antes de producción.

Detalles: [MongoDB Docker: incidencias y compatibilidad](base-conocimiento/mongodb-docker-lecciones-linux.md).

### 2.10. Probar primero la dependencia externa

Antes de responsabilizar a Zabbix, probar directamente la tecnología monitoreada:

```text
Oracle -> SQL*Plus/listener
MongoDB -> mongosh/serverStatus
Docker -> docker ps/docker info
HTTP -> curl
Linux -> systemctl/ss
```

Si la dependencia falla fuera de Zabbix, corregirla antes de tocar la plantilla.

### 2.11. Los servicios `systemd` tienen su propio entorno

Que una librería o variable funcione en una terminal de usuario no implica que esté disponible para `zabbix-agent2` iniciado por `systemd`. Este punto fue determinante con Oracle Client.

### 2.12. Privilegio mínimo antes que “hacer que funcione”

No resolver errores otorgando roles amplios, `sudo` general, permisos `777`, acceso global al socket Docker o credenciales administrativas.

Primero identificar la operación exacta requerida; después otorgar el mínimo permiso y documentarlo.

### 2.13. Licenciamiento forma parte del diseño técnico

En Oracle, una métrica técnicamente accesible puede depender de opciones licenciadas. Antes de activar plantillas completas se deben revisar consultas, vistas y privilegios.

### 2.14. Un contenedor `running` no equivale a una aplicación sana

Separar siempre:

```text
Host Linux
Docker Engine
Contenedor
Proceso/aplicación/base dentro del contenedor
Servicio funcional para el usuario
```

Para aplicaciones críticas se necesita una comprobación funcional adicional: puerto, HTTP, endpoint, consulta, ping de base, etc.

### 2.15. El Agent 2 se instala normalmente en el host, no dentro de cada contenedor

Esto permite detectar la caída de un contenedor desde fuera de él y reduce agentes duplicados. Las excepciones deben justificarse explícitamente.

### 2.16. Varias instancias en un host requieren dimensionamiento conjunto

Con varios MongoDB, JVM, bases o servicios pesados en el mismo Linux, no se puede dimensionar cada proceso como si fuera el único consumidor del host. Deben considerarse límites de contenedor, cachés, CPU, I/O y RAM agregada.

### 2.17. Los umbrales de fábrica son punto de partida

Antes de modificar un trigger:

- entender la métrica;
- confirmar unidad e intervalo;
- revisar expresión;
- observar comportamiento histórico;
- validar que la semántica siga vigente en la versión actual del producto.

No modificar infraestructura únicamente para “poner en verde” una alerta genérica.

### 2.18. Una configuración no termina cuando aparecen gráficas

Debe probarse al menos una falla controlada y su recuperación cuando sea seguro hacerlo. El laboratorio MongoDB validó este principio deteniendo el contenedor y comprobando creación/cierre automático del problema.

### 2.19. Reinicio y persistencia son parte de la aceptación

Antes de considerar listo un servidor o integración comprobar, cuando corresponda:

- reinicio del servicio;
- reinicio del host;
- persistencia del firewall;
- persistencia de volúmenes;
- arranque automático;
- continuidad del monitoreo.

### 2.20. Cada solución debe poder reproducirse

Toda incidencia debe registrar:

```text
Síntoma
Ambiente
Causa
Diagnóstico
Cambio exacto
Validación
Reversión
Estado
```

La memoria del operador no sustituye la documentación.

## 3. Qué documento usar

| Necesidad | Documento |
|---|---|
| Procedimiento paso a paso | `docs/guias/` |
| Error real encontrado | `docs/base-conocimiento/` |
| Principio preventivo general | Este archivo |
| Verificación antes de desplegar | `docs/checklist-preventivo-linux.md` |
| Estado y secuencia del proyecto | `docs/implementacion-zabbix-docker-oracle.md` |

## 4. Regla de salida

Una integración se considera técnicamente lista solo cuando:

```text
conecta
+ recopila datos
+ no tiene elementos no soportados sin explicación
+ alerta ante una falla relevante
+ recupera correctamente
+ sobrevive al reinicio/persistencia aplicable
+ queda documentada
```
