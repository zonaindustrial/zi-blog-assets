# 04 · Implementación de referencia: acuse de lead con IA

El acuse de ticket replica la fórmula del **acuse de lead** (CRM). Diferencia clave: en CRM
el acuse ya lo enviaba una acción automatizada (fácil de reemplazar), mientras que en
Helpdesk lo envía el core en el `create` (por eso hubo que crear una etapa de entrada sin
plantilla para el equipo 8).

## Cadena del acuse de lead

```
base.automation 38 "ZI · Acuse de lead con IA"
   (model crm.lead, on_write, filter_pre stage = "Por Contactar",
    filter user_id.name in [Luis Zegarra, Loredana, Ivo, Nicolás, Marco, Samantha])
        │
        └─ server action 7579 (mismo codigo)
               ├─ arma ctx (correo original, contacto, vendedor, sector, historial, prospecto)
               ├─ saludo horario Chile
               ├─ env['zi.ai.worker'].run_worker(4, ctx)
               ├─ guarda en x_studio_zi_ai_lead_ack (ir.model.fields 22241, html, crm.lead)
               ├─ GUARD DUEMINT (si el email es la casilla DTE, redirige o no envia)
               └─ mail.template 83 .send_mail(lead.id)   (renderiza el campo IA, fallback estatico)
```

- **Worker:** `zi.ai.worker` id **4**: "Enriquecimiento del acuse de lead".
- **Gate:** un solo acuse por lead (si `x_studio_zi_ai_lead_ack` ya tiene contenido, no regenera).
- **Guard Duemint:** miles de partners tienen como email la casilla de facturación
  electrónica `dte@duemint.com`. Si el email efectivo contiene `duemint`, busca un contacto de
  la empresa con email válido y redirige; si no hay, no envía y deja nota interna.

## Familia completa de enriquecimientos (`zi.ai.worker`)

| id | Worker | Enriquece |
|---|---|---|
| 1 | Enriquecimiento del email | correo genérico |
| 2 | Enriquecimiento del seguimiento | seguimiento comercial |
| 3 | Enriquecimiento del vencimiento | aviso de vencimiento |
| 4 | Enriquecimiento del acuse de lead | acuse de lead (CRM) |
| 5 | Enriquecimiento de recuperación de carrito | carrito abandonado web |
| 6 | Enriquecimiento de la confirmación de venta | confirmación de nota de venta |
| 7 | Enriquecimiento de notificación de entrega | notificación de entrega/logística |
| **8** | **Enriquecimiento del acuse de ticket** | **acuse de ticket helpdesk (NUEVO)** |

## Disparadores hermanos (referencia en el código de la acción de lead)

> "HERMANOS: 1579 (inicial SO), 7554 (seguimiento), 7555 (vencimiento), 7640 (carrito web).
> DOCUMENTACION: docs/ACUSE_LEAD_CON_IA.md (zi-odoo-addons)."

La documentación canónica de la familia vive en el repo **`zi-odoo-addons`**
(`docs/ACUSE_LEAD_CON_IA.md`). Este `ENRIQUECEDOR/` documenta específicamente el acuse de
ticket (id 8), que es el nuevo.

## Diferencias acuse de lead vs acuse de ticket

| | Lead (id 4) | Ticket (id 8) |
|---|---|---|
| Modelo Odoo | `crm.lead` | `helpdesk.ticket` |
| Voz | primera persona del VENDEDOR asignado (con género) | primera persona plural del EQUIPO |
| Disparador | `base.automation` 38 (on_write, cambio de etapa) | `base.automation` 127 (on_create, team_id=8) |
| Supresión del acuse viejo | reemplazó la acción 1035 | etapa de entrada propia sin plantilla (stage 55) |
| Campo | `x_studio_zi_ai_lead_ack` (22241) | `x_studio_zi_ai_ticket_ack` (22265) |
| Plantilla | `mail.template` 83 | `mail.template` 39 (`helpdesk.new_ticket_request_email_template`) |
