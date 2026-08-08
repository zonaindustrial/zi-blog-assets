# ENRIQUECEDOR · Acuse de ticket de Helpdesk con IA

Documentación completa del enriquecimiento con IA del **primer correo de acuse** que
recibe un cliente cuando envía una consulta por el formulario de contacto del sitio web
(equipo Helpdesk **Contacto Sitio web**, `helpdesk.team` id **8**).

Sigue la misma fórmula que el resto de enriquecimientos de Zona Industrial en Odoo: un
**Worker de IA** (`zi.ai.worker`) que redacta el cuerpo, una **automatización** que arma el
contexto y dispara el envío, y una **plantilla** de correo que renderiza el resultado.

- **Instancia Odoo:** producción ZI
- **Estado:** ACTIVO (probado end-to-end el 2026-08-08)
- **Modelo IA:** Claude Sonnet 4.6 (`claude-sonnet-4-6`), proveedor Anthropic

## Índice

| Archivo | Contenido |
|---|---|
| `README.md` | Este resumen + diagrama del flujo + tabla maestra de objetos |
| `01-framework-zi-ai.md` | El framework `zi.ai.*` de Odoo y el método `run_worker()` |
| `02-flujo-acuse-ticket.md` | El flujo del acuse de ticket paso a paso: rutas, métodos, código |
| `03-worker-prompt-guardrails.md` | El worker id 8: system prompt completo y guardrails |
| `04-referencia-acuse-lead.md` | La implementación de referencia (acuse de lead) y hermanos |
| `05-operacion-pruebas-reversion.md` | Cómo probar, cómo revertir, respaldo de la plantilla original |

## El problema que resuelve

El acuse automático nativo de Odoo:

1. Empezaba con `Estimado/a Madam/Sir,` (fallback en inglés sin traducir).
2. Era genérico: no saludaba por hora, no agradecía, no demostraba haber leído la consulta.
3. Firmaba con el nombre interno del equipo y arrastraba el branding de Odoo.

## El flujo (resumen)

```
Formulario web zonaindustrial.cl
        │  (crea helpdesk.ticket, equipo 8)
        ▼
helpdesk.ticket.create()  ──►  etapa por defecto = "Nuevo" (id 55, SIN plantilla)
        │                       => el core NO manda acuse (esa es la clave)
        │
        ├─ base.automation 127 "ZI · Acuse de ticket con IA" (on_create, team_id=8)
        │       │  (corre DESPUES de la creacion, en transaccion separada)
        │       ├─ arma el contexto del ticket (asunto, mensaje, contacto, saludo por hora)
        │       ├─ env['zi.ai.worker'].run_worker(8, ctx)  ──►  worker id 8 (Claude)
        │       ├─ guarda el HTML en x_studio_zi_ai_ticket_ack
        │       └─ mail.template(39).send_mail(ticket.id)
        │               └─ plantilla 39 renderiza el campo IA  ──►  correo al cliente
        ▼
Cliente recibe UN acuse calido, humano, con saludo por hora
```

Si el worker o el flujo falla, el `try/except` no corta nada: el campo queda vacío y la
plantilla 39 usa su **saludo genérico cálido por hora** (nunca "Estimado/a cliente").

## Tabla maestra de objetos (todos los IDs)

| Objeto | Modelo | ID | Rol |
|---|---|---|---|
| Equipo | `helpdesk.team` | **8** | "Contacto Sitio web" |
| Etapa de entrada | `helpdesk.stage` | **55** | "Nuevo" propia del equipo 8, SIN plantilla (suprime el acuse del core) |
| Etapa compartida | `helpdesk.stage` | 1 | "Nuevo" de los otros 10 equipos (equipo 8 fue removido) |
| Worker IA | `zi.ai.worker` | **8** | "Enriquecimiento del acuse de ticket" (redactor) |
| Modelo IA | `zi.ai.model` | 1 | Claude Sonnet 4.6 |
| Campo IA | `ir.model.fields` | **22265** | `x_studio_zi_ai_ticket_ack` en `helpdesk.ticket` (html) |
| Automatización | `base.automation` | **127** | "ZI · Acuse de ticket con IA" (on_create, genera y envía) |
| Plantilla correo | `mail.template` | **39** | `helpdesk.new_ticket_request_email_template` (modificada) |

## Regla de oro (petición del negocio)

- La **creación del ticket corre como siempre**: no se interviene el `create` (un compute
  con IA rompía la transacción; descartado).
- Se **interviene en el paso del envío del correo**: la automatización post-creación genera
  la respuesta y la envía.
- **Si la IA falla, no se corta**: se envía el genérico lo más mejorado posible (saludo por
  hora + nombre + número de ticket).
- El saludo es SIEMPRE "Buenos días / Buenas tardes / Buenas noches", cálido y humano.
