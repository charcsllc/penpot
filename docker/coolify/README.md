# Penpot en Coolify

Guía para desplegar Penpot en un VPS con Coolify, en `https://penpot.charcs.net`,
sin servidor de correo (SMTP) y con el servidor MCP activo.

Archivos de esta carpeta:

- `docker-compose.yaml`: el stack de Penpot adaptado a Coolify.
- `README.md`: esta guía (despliegue y gestión de usuarios).
- `MCP-CLAUDE.md`: cómo conectar Claude a Penpot por MCP.

## Requisitos

- Un VPS con Coolify instalado. Penpot usa bastante memoria (el backend es
  Java y el exporter abre un navegador sin interfaz): con menos de 4 GB de RAM
  irá justo.
- Un registro A en Cloudflare: `penpot.charcs.net` apuntando a la IP del VPS.

## 1. Cloudflare

Si el registro está en modo proxy (nube naranja):

1. En **SSL/TLS → Overview**, elige **Full (strict)**. No uses **Flexible**:
   Coolify redirige HTTP a HTTPS y se forma un bucle de redirecciones.
2. Coolify pide el certificado a Let's Encrypt en el primer despliegue. Si el
   navegador muestra el error 526 o el certificado no llega, pon el registro en
   **DNS only** (nube gris), despliega, espera a que Coolify obtenga el
   certificado y vuelve a activar el proxy.

Límites del proxy de Cloudflare en el plan gratis:

- Subidas de hasta 100 MB por petición. Penpot admite hasta 350 MB.
- Una petición que tarda más de 100 segundos se corta con el error 524. Puede
  pasar con tareas MCP largas (ver `MCP-CLAUDE.md`).

Si estos límites te molestan, deja el registro en **DNS only**: Coolify sigue
poniendo el HTTPS.

## 2. Crear el recurso en Coolify

Elige una de estas dos formas:

- **Pegar el compose** (la más sencilla): **New Resource → Docker Compose
  Empty** y pega el contenido de `docker-compose.yaml`.
- **Desde el repositorio Git**: **New Resource → (tu repositorio)**, con la
  rama `feature/main_coolify`, el build pack **Docker Compose** y como
  **Docker Compose Location** `/docker/coolify/docker-compose.yaml`. La rama
  tiene que estar subida a tu repositorio remoto.

## 3. Variables de entorno

Esta es una instancia de prueba, así que el compose trae dos claves fijas por
defecto. No hace falta rellenar nada para desplegar:

| Variable | Qué es | Valor por defecto |
| --- | --- | --- |
| `PENPOT_SECRET_KEY` | Clave maestra de Penpot (sesiones, tokens). | `llave_secreta_2026` |
| `POSTGRES_PASSWORD` | Contraseña de la base de datos. | `llave_secreta_2026` |

Cualquiera que lea este repositorio conoce esas claves. Con
`PENPOT_SECRET_KEY` se pueden falsificar sesiones y claves MCP, así que
cámbiala antes de guardar en Penpot algo que te importe. Para cambiarlas,
defínelas en la pestaña **Environment Variables** del recurso **antes del
primer despliegue**:

- `PENPOT_SECRET_KEY`: `python3 -c "import secrets; print(secrets.token_urlsafe(64))"`
- `POSTGRES_PASSWORD`: `openssl rand -hex 32`

Una vez desplegado, no las cambies:

- Si cambias `PENPOT_SECRET_KEY`, se cierran todas las sesiones y dejan de
  valer las claves MCP.
- Postgres solo lee `POSTGRES_PASSWORD` al crear la base de datos. Si la
  cambias más tarde, el backend no podrá conectarse.

Variables opcionales (ya tienen un valor por defecto en el compose):

| Variable | Valor por defecto |
| --- | --- |
| `PENPOT_PUBLIC_URI` | `https://penpot.charcs.net` (sin barra final) |
| `PENPOT_VERSION` | `2.18` |
| `PENPOT_FLAGS` | `disable-registration disable-email-verification enable-prepl-server enable-mcp enable-admin-console` |

`PENPOT_FLAGS` se comparte entre el frontend, el backend y el exporter. Si lo
cambias, cámbialo en la variable, no en un solo servicio.

## 4. Dominio

En la configuración del servicio **penpot-frontend**, en **Domains**, escribe:

```
https://penpot.charcs.net:8080
```

El `:8080` indica a Coolify el puerto interno del contenedor. La dirección
pública queda sin puerto: `https://penpot.charcs.net`.

Los demás servicios van sin dominio. Solo se comunican dentro de la red del
stack.

## 5. Desplegar y comprobar

1. Pulsa **Deploy**. El primer arranque tarda unos minutos: el backend crea las
   tablas de la base de datos.
2. Abre `https://penpot.charcs.net/readyz`. Debe responder `OK`.
3. Abre `https://penpot.charcs.net`. Verás la pantalla de inicio de sesión,
   sin opción de registro (está desactivado).

## 6. Crear tu usuario

Como el registro está desactivado, los usuarios se crean desde el contenedor
del backend con `manage.py`.

**Desde Coolify:** en el recurso, abre la pestaña **Terminal**, elige el
contenedor `penpot-backend` y ejecuta:

```sh
python3 manage.py create-profile
```

**Por SSH en el VPS:**

```sh
docker exec -ti $(docker ps -qf name=penpot-backend) python3 manage.py create-profile
```

El comando pregunta tres datos:

```
Email: tu@email.com
Fullname: Tu Nombre
Password:            <- no se ve al escribir
Created: tu@email.com / 3f9c...
```

También puedes pasar el email y el nombre como opciones y escribir solo la
contraseña:

```sh
python3 manage.py create-profile -e tu@email.com -n "Tu Nombre"
```

No pases la contraseña con `-p`: quedaría guardada en el historial de la
terminal.

Ya puedes entrar en `https://penpot.charcs.net` con ese email y contraseña.

## 7. Gestionar usuarios

Todos los comandos se ejecutan igual que en el paso 6 (Terminal de Coolify o
`docker exec`).

| Tarea | Comando |
| --- | --- |
| Cambiar la contraseña | `python3 manage.py update-profile -e tu@email.com` |
| Buscar un usuario | `python3 manage.py search-profile -e tu@email.com` |
| Borrar un usuario (se puede recuperar) | `python3 manage.py delete-profile -e tu@email.com` |
| Borrar un usuario para siempre | `python3 manage.py delete-profile -e tu@email.com -f` |

Sin SMTP, la opción "He olvidado mi contraseña" de la web no envía nada. Usa
`update-profile` para cambiarla.

## 8. Lo que no funciona sin SMTP

Penpot usa el correo para:

- recuperar la contraseña,
- invitar a otras personas a un equipo,
- avisos y notificaciones.

Para uso personal (por ejemplo, con Claude por MCP) no hace falta. Si más
adelante quieres invitar a otras personas, añade un proveedor SMTP: las
variables `PENPOT_SMTP_*` aparecen en `docker/images/docker-compose.yaml`, y
tendrás que añadir `enable-smtp` a `PENPOT_FLAGS`.

## 9. Mantenimiento

**Actualizar Penpot:** cambia `PENPOT_VERSION` (por ejemplo, a `2.19`) y vuelve
a desplegar. Lee antes las notas de la versión por si hay pasos manuales.

**Copias de seguridad:** los datos viven en dos volúmenes:

- `penpot_postgres_v15`: la base de datos.
- `penpot_assets`: imágenes, fuentes y archivos subidos.

Para volcar la base de datos a un archivo en el VPS:

```sh
docker exec $(docker ps -qf name=penpot-postgres) pg_dump -U penpot penpot > penpot-$(date +%F).sql
```

**Si nginx no arranca** en el contenedor `penpot-frontend` con un error sobre
`[::]:8080`, el VPS tiene IPv6 desactivado. Descomenta
`PENPOT_DISABLE_IPV6_LISTEN: "true"` en el compose y vuelve a desplegar.
