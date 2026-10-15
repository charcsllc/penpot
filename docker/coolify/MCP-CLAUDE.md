# Usar Penpot como MCP en Claude

Guía para conectar Claude (Claude Code, Claude Desktop o claude.ai) a tu
Penpot en `https://penpot.charcs.net`, para que Claude pueda leer y editar tus
diseños.

Antes, despliega Penpot y crea tu usuario siguiendo `README.md`.

## Cómo funciona

Intervienen tres piezas:

- **Servidor MCP**: el contenedor `penpot-mcp`. Penpot lo publica en
  `https://penpot.charcs.net/mcp/stream`.
- **Plugin MCP**: corre dentro de tu pestaña de Penpot, en el navegador, y une
  el archivo abierto con el servidor (por `/mcp/ws`).
- **Claude**: se conecta al servidor con tu clave MCP y le manda órdenes.

Claude no lee Penpot por su cuenta: cada orden pasa por el servidor hasta el
plugin de tu pestaña, que la ejecuta en el archivo abierto. Por eso **Penpot
tiene que estar abierto en el navegador, con el archivo y el plugin
conectados**, mientras usas Claude.

Claude trabaja siempre sobre la **página que tienes abierta** en Penpot.

## 1. Activar el MCP y generar tu clave

1. Entra en `https://penpot.charcs.net`.
2. Ve a **Tu cuenta → Integraciones → Servidor MCP**.
3. Activa el interruptor **Estado**.
4. Pulsa **Generar clave MCP**. Penpot muestra la clave **una sola vez**:
   guárdala en un gestor de contraseñas. También verás su fecha de caducidad.
5. Copia la URL del servidor. Tiene esta forma:

   ```
   https://penpot.charcs.net/mcp/stream?userToken=TU_CLAVE
   ```

En el resto de la guía, cambia `TU_CLAVE` por tu clave.

## 2. Añadir Penpot a Claude

Elige el cliente que uses. Puedes configurar varios con la misma URL.

### Claude Code

```sh
claude mcp add --transport http --scope user penpot "https://penpot.charcs.net/mcp/stream?userToken=TU_CLAVE"
```

- Las comillas hacen falta: el `?` de la URL tiene un significado especial en
  la terminal.
- `--scope user` deja Penpot disponible en todos tus proyectos y guarda la
  clave en tu configuración personal, fuera de cualquier repositorio. No uses
  `--scope project`: escribe la clave en `.mcp.json`, dentro del proyecto, y
  podrías subirla a Git por error.

Para comprobarlo:

```sh
claude mcp list
```

Debe aparecer `penpot` como conectado. Dentro de una sesión de Claude Code,
`/mcp` muestra el mismo estado.

### Claude Desktop y claude.ai (conector personalizado)

1. En Claude, abre **Settings → Connectors** (también en
   `https://claude.ai/customize/connectors`).
2. Pulsa **Add custom connector**.
3. Nombre: `Penpot`. URL: la URL completa con tu clave.
4. Deja vacíos los campos de OAuth: la clave ya va en la URL.
5. En un chat, activa el conector Penpot desde el menú de herramientas.

El conector queda en tu cuenta, así que aparece en claude.ai y en Claude
Desktop. Según tu plan, puede que solo el propietario de la organización pueda
añadir conectores.

Con este método se conectan los servidores de Anthropic, no tu ordenador. Por
eso la URL tiene que estar abierta a internet, como ya lo está. Si Cloudflare
bloquea la conexión, mira la sección de problemas frecuentes.

### Alternativa para Claude Desktop: archivo de configuración

Si no puedes usar conectores personalizados, añade Penpot al archivo de
configuración de Claude Desktop. Necesitas Node.js instalado.

Ubicación del archivo:

- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`

Contenido:

```json
{
  "mcpServers": {
    "penpot": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://penpot.charcs.net/mcp/stream?userToken=TU_CLAVE"
      ]
    }
  }
}
```

Si el archivo ya tiene otros servidores, añade solo el bloque `"penpot"` dentro
de `"mcpServers"`. Cierra Claude Desktop del todo y vuelve a abrirlo.

## 3. Conectar el plugin en Penpot

Hazlo cada vez que vayas a trabajar con Claude:

1. Abre un archivo de diseño en `https://penpot.charcs.net`.
2. En el menú, ve a **Archivo → Servidor MCP → Conectar**.
3. La barra de herramientas mostrará **MCP conectado**.

El MCP solo puede estar activo en una pestaña. Si abres Penpot en otra, verás
el aviso "MCP está activo en otra pestaña. ¿Cambiar a esta?".

**Mantén la pestaña despierta.** Si el navegador la congela o la descarga para
ahorrar memoria, Claude no podrá hacer nada en Penpot:

- **Chrome**: en **Configuración → Rendimiento**, añade `penpot.charcs.net` a
  **Mantener siempre activos estos sitios**. Fijar la pestaña también ayuda.
- **Edge**: en **Configuración → Sistema y rendimiento**, añade el sitio a la
  lista de sitios que nunca se suspenden.
- **Firefox y Safari**: no tienen un ajuste por sitio. Deja la pestaña abierta
  y a la vista durante las sesiones largas.

## 4. Primeras pruebas

Empieza con peticiones que solo leen, para comprobar que todo funciona:

- "Lista las páginas de este archivo de Penpot."
- "Resume la estructura de la página actual."
- "¿Qué colores y tipografías usa esta página?"

Después prueba un cambio pequeño:

- "Crea un rectángulo azul de 200×100 en la página actual."

Para cambios grandes, pide a Claude que **describa lo que va a cambiar antes de
hacerlo**, y duplica el archivo si no quieres arriesgar el original.

Herramientas que ofrece tu servidor (modo remoto):

| Herramienta | Para qué sirve |
| --- | --- |
| `high_level_overview` | Instrucciones de uso de Penpot para Claude. |
| `penpot_api_info` | Documentación de la API de plugins de Penpot. |
| `execute_code` | Ejecuta código en el archivo abierto: leer, crear, mover, cambiar estilos… |
| `export_shape` | Exporta una forma como imagen y se la enseña a Claude. |

En modo remoto el servidor no tiene acceso a archivos de tu ordenador: Claude
no puede importar imágenes desde una ruta local ni guardar exportaciones en tu
disco.

## Seguridad

- **La URL lleva tu clave: trátala como una contraseña.** No la subas a Git, no
  la pegues en capturas ni en chats.
- **Si se filtra**, ve a **Integraciones → Servidor MCP → Regenerar clave MCP**.
  La clave anterior deja de funcionar al momento; actualiza la URL en Claude.
- **Cuando caduque**, Penpot lo indica en Integraciones. Regenera la clave y
  actualiza la URL en Claude.
- **Para cortar el acceso**, desconecta el plugin o desactiva el **Estado** en
  Integraciones. Claude deja de poder tocar tus diseños aunque siga configurado.

## Problemas frecuentes

| Síntoma | Causa | Solución |
| --- | --- | --- |
| Claude dice que no hay ninguna instancia del plugin de Penpot conectada. | El plugin no está conectado. | Abre un archivo y usa **Archivo → Servidor MCP → Conectar**. |
| Claude dice que la pestaña de Penpot parece suspendida. | El navegador congeló la pestaña. | Vuelve a la pestaña, espera a que reconecte y repite la petición. Ver "Mantén la pestaña despierta". |
| Las tareas largas fallan con el error 524. | El proxy de Cloudflare corta las peticiones de más de 100 segundos. | Divide la tarea en pasos más cortos, o pon el registro DNS en **DNS only**. |
| El conector de claude.ai no conecta, pero Claude Code sí. | Cloudflare bloquea a los servidores de Anthropic (Bot Fight Mode o una regla WAF). | En Cloudflare, mira **Security → Events**. Desactiva Bot Fight Mode o crea una regla que deje pasar la ruta `/mcp/`. |
| Claude no conecta con el servidor. | Clave mal copiada o caducada, o el servidor MCP está caído. | Revisa la URL y la clave, y haz la prueba de abajo. |

**Prueba rápida del servidor MCP** (desde cualquier terminal):

```sh
curl -s -o /dev/null -w "%{http_code}\n" https://penpot.charcs.net/mcp/stream
```

- `502`: el contenedor `penpot-mcp` no está corriendo. Revisa sus logs en
  Coolify.
- `301`: falta `enable-mcp` en `PENPOT_FLAGS`.
- `400`, `405` o `406`: el servidor responde. El error se debe a que `curl` no
  habla MCP, y es lo esperado.

Más información: <https://help.penpot.app/mcp/>
