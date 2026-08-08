# 01 · El framework `zi.ai.*` de Odoo

Zona Industrial tiene un framework de IA propio en Odoo. Los enriquecimientos de correos
se apoyan en el modelo `zi.ai.worker`.

## Modelos del framework

| Modelo | Nombre | Rol |
|---|---|---|
| `zi.ai.worker` | Worker de IA (agente headless por cron) | Un agente/redactor configurable. **Es lo que usamos.** |
| `zi.ai.runner` | Motor server-side IA: run_code, edición y confirmación | Motor de ejecución |
| `zi.ai.engine` | Motor agéntico de Integración IA | Motor agéntico |
| `zi.ai.assistant` | Asistente de IA | Asistente interactivo |
| `zi.ai.mcp.server` | Servidor MCP | Registro de servidores MCP disponibles para los workers |
| `zi.ai.model` | Modelo de IA | Catálogo de modelos (id 1 = Claude Sonnet 4.6) |
| `zi.ai.provider` | Proveedor de IA | Anthropic / OpenAI / compatibles |
| `zi.ai.chat` / `zi.ai.chat.message` | Conversación de IA | Historial de chats |

## `zi.ai.worker` · campos

| Campo | Tipo | Notas |
|---|---|---|
| `name` | char | Nombre del worker |
| `sequence` | integer | Orden |
| `active` | boolean | |
| `model_id` | m2o `zi.ai.model` | Modelo (id 1 = Claude Sonnet 4.6) |
| `provider_type` | selection | `anthropic` / `openai` / `openai_compatible` |
| `access_mode` | selection | `read` (solo lectura) / `read_write` |
| `mcp_server_ids` | m2m `zi.ai.mcp.server` | Servidores MCP habilitados (vacío = sin herramientas) |
| `tool_whitelist` | text | Lista blanca de herramientas |
| `system_prompt` | text | Reglas permanentes del worker |
| `task_prompt` | text | Descripción de la tarea/contexto que recibe |
| `max_iterations` | integer | Nº de iteraciones (1 = una sola pasada, sin bucle de tools) |
| `last_run` / `last_state` / `last_result` / `last_log` | | Traza de la última corrida |

## Método clave: `run_worker`

```python
resultado_html = env['zi.ai.worker'].run_worker(worker_id, context_string)
```

- **`worker_id`** (int): el id del `zi.ai.worker` a ejecutar (para el acuse de ticket, **8**).
- **`context_string`** (str): TODO el contexto ya preparado que el worker necesita. El worker
  del acuse no tiene herramientas (`mcp_server_ids` vacío, `access_mode: read`): solo redacta
  con lo que recibe en este string.
- **Retorna**: el texto que produjo el modelo (para nuestros workers, el HTML del cuerpo).
- **Efecto lateral**: actualiza `last_run`, `last_state`, `last_result` del worker.

### Patrón de la fórmula ZI

1. Un **disparador** (automatización/acción) arma el `context_string` con todo lo necesario.
2. Llama a `run_worker(id, ctx)`.
3. Limpia el resultado (quita fences ```` ``` ```` si vinieran) y lo usa (guardar en un campo,
   enviar un correo, etc.).

Los workers redactores (acuse de lead, venta, entrega, ticket) NO consultan Odoo: reciben el
contexto pre-armado. Esto es intencional (minimización de datos: el modelo nunca ve campos
sensibles que no se le pasen).
