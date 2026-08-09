# 06 · Todos los workers de IA y sus rutas

Los 8 `zi.ai.worker` que enriquecen correos, con su disparador (ruta), campo y plantilla.
Todos usan Claude Sonnet 4.6 (`zi.ai.model` id 1), `access_mode: read`, sin MCP ni
herramientas, `max_iterations: 1`. Cada disparador arma el contexto (lee datos del cliente,
vendedor, productos, etc.) y llama `run_worker(id, ctx)` o `run_task(ctx)`.

| Worker | id | Disparador (ruta) | Modelo Odoo | Campo destino | Plantilla | Saludo |
|---|---|---|---|---|---|---|
| Enriquecimiento del email (cotización inicial) | 1 | Botón "Enviar por correo" (vista 4915) → server action **1579** (`run_task`) | sale.order | `order.note` (firma en `x_studio_zi_ai_sig`) | **141** | por hora |
| Enriquecimiento del seguimiento | 2 | base.automation **47** (on_time +48h, state=sent) → server action **7554** | sale.order | `x_studio_zi_ai_followup` | **111** | por hora |
| Enriquecimiento del vencimiento | 3 | server action **7555** (cron de vencimiento) | sale.order | (campo IA vencimiento) | (plantilla venc.) | por hora |
| Enriquecimiento del acuse de lead | 4 | base.automation **38** → server action **7579** | crm.lead | `x_studio_zi_ai_lead_ack` (22241) | **83** | por hora |
| Enriquecimiento de recuperación de carrito | 5 | base.automation → server action **7640** | sale.order | (campo IA carrito) | (plantilla carrito) | TUTEO web |
| Enriquecimiento de la confirmación de venta | 6 | server action (inicial SO confirmada) | sale.order | (campo IA venta) | (plantilla venta) | por hora |
| Enriquecimiento de notificación de entrega | 7 | server action (entrega/logística) | sale.order / stock | (campo IA entrega) | (plantilla entrega) | por hora |
| **Enriquecimiento del acuse de ticket** | **8** | base.automation **127** (on_create team 8) | helpdesk.ticket | `x_studio_zi_ai_ticket_ack` (22265) | **39** | por hora |

## Patrón común de cada disparador

1. Arma `ctx` (string) con: contacto, empresa, vendedor, sector, diagnóstico/prospecto,
   productos, ganchos (descuentos, despacho), stock verificado, línea TECNOLOGÍA (Intcomex),
   historial de compras, y el **saludo horario de Chile**.
2. `run_worker(id, ctx)` (o `run_task` en el inicial), limpia fences ```` ``` ````.
3. Guarda el HTML en el campo destino.
4. Guard Duemint: si el email es la casilla DTE `dte@duemint.com` (o interno), redirige a un
   contacto válido o no envía y deja nota.
5. Envía la plantilla (o abre el compose, en el inicial).

## Reglas de negocio transversales (en los prompts)

- **Línea TECNOLOGÍA (Intcomex, categ 4999):** productos sin stock físico propio dependen del
  stock del proveedor y NO se reservan con la cotización. Advertencia obligatoria.
- **Stock:** solo se afirma disponibilidad si el contexto la confirma. Nunca inventar
  importación/tránsito/demora.
- **Historial:** reconocer relación previa con calidez, sin montos/cantidades/fechas.
- **Cross-selling (worker 1):** solo candidatos verificados en stock, de distinta familia;
  nunca ofrecer otra variante del mismo producto.
- **Guard boleta:** "CLIENTE BOLETA" nunca se menciona; se trata como persona natural.

## Método `run_worker` vs `run_task`

- `run_worker(id, ctx)`: usado por seguimiento, vencimiento, acuse lead, acuse ticket,
  carrito. Devuelve el HTML y actualiza `last_run`/`last_state`.
- `run_task(ctx)` sobre `browse(id)`: usado por el correo inicial (1579), equivalente.
- Motor: repo `zonaindustrial/zi-odoo-addons`, módulo `zi_ai_runner`.
