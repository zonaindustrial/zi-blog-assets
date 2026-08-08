# Acuse de ticket enriquecido con IA (Helpdesk "Contacto Sitio web")

Enriquecimiento del **primer correo de respuesta** a los tickets del equipo Helpdesk
`Contacto Sitio web`, siguiendo la misma fórmula que los demás enriquecimientos de
Zona Industrial en Odoo (familia `zi.ai.worker`).

> Nota: la implementación vive en Odoo (no en n8n). La documentación canónica de esta
> familia está en el repo `zi-odoo-addons` (`docs/ACUSE_LEAD_CON_IA.md`). Este archivo es
> solo un resumen del acuse de ticket.

## Motivación (auditoría del acuse actual)

El acuse automático nativo de Odoo tiene problemas:

1. **Bug de idioma:** empieza con `Estimado/a Madam/Sir,` (fallback en inglés sin traducir).
2. **Genérico:** no saluda según la hora, no agradece, no demuestra que se entendió el motivo.
3. **Firma interna:** cierra como "Equipo Contacto Sitio web" y arrastra el branding de Odoo.

## La fórmula (idéntica al acuse de lead)

Cadena del acuse de lead que se replica:

```
base.automation 38 (asignación en etapa Por Contactar)
   -> server action 7579 "ZI · Acuse de lead con IA" (arma contexto + saludo horario Chile)
      -> env['zi.ai.worker'].run_worker(4, ctx)   # redactor puro, Claude Sonnet 4.6, read
         -> guarda HTML en x_studio_zi_ai_lead_ack
            -> envia mail.template 83 (renderiza el campo IA, fallback estatico)
```

Workers hermanos ya existentes (`zi.ai.worker`): acuse de lead (4), confirmación de venta (6),
notificación de entrega (7), seguimiento, vencimiento, recuperación de carrito, email.

## Lo creado para tickets

- **Worker `zi.ai.worker` id 8 — "Enriquecimiento del acuse de ticket"** (creado).
  - Modelo: Claude Sonnet 4.6 (`claude-sonnet-4-6`), `provider: anthropic`.
  - `access_mode: read`, sin servidores MCP, sin herramientas, `max_iterations: 1`.
  - Redactor puro: recibe todo el contexto pre-armado y devuelve solo el cuerpo HTML.

### Reglas del worker (system prompt)

- Voz de equipo (recibimos, revisamos, le responderemos), trato de usted, español de Chile,
  tildes perfectas, **sin raya (em dash)**, destacados con `<strong style="color:#17375A;">`.
- Abre con el **saludo horario** que le entrega el contexto + nombre del cliente.
- Parafrasea el motivo, cita el **número de ticket**, da el plazo de respuesta e invita a
  responder el correo con más detalles.
- **Confidencialidad estricta:** jamás precios, márgenes, costos, proveedores, stock interno,
  ni datos o pedidos de otros clientes. No inventa precios/plazos/disponibilidad.
- **Excepción PEDIDO VINCULADO:** solo si el contexto trae un bloque verificado con el pedido
  del propio cliente, puede indicar estado, **fecha de entrega comprometida** y mencionar el
  **PDF adjunto** (factura/cotización). En ningún otro caso.

## Pendiente de cablear en Odoo (mismos pasos que el lead, aún NO hechos)

Estos pasos hacen que el acuse llegue al cliente. No se ejecutan solos hasta activarlos.

1. **Campo** en `helpdesk.ticket` para guardar el cuerpo IA (ej. `x_studio_zi_ai_ticket_ack`).
2. **Server action** (modelo `helpdesk.ticket`) que replique la 7579: arma el contexto del
   ticket (asunto, mensaje original del chatter, contacto, empresa, historial, saludo horario;
   y, si el ticket está ligado a un pedido del propio cliente, el bloque PEDIDO VINCULADO con
   lista blanca de campos: estado, fecha de entrega, PDF), llama `run_worker(8, ctx)`, guarda
   el campo y envía la plantilla. Reutilizar el guard anti-Duemint.
3. **Plantilla** de acuse de ticket que renderice el campo IA (fallback estático) en lugar del
   `Estimado/a Madam/Sir`.
4. **Trigger** `base.automation` al crear ticket del equipo `Contacto Sitio web`.

## Modo híbrido (decisión de negocio)

- Auto-enviable: el acuse enriquecido (saludo + motivo entendido + próximos pasos), sin datos
  sensibles.
- Datos de pedido (fecha de entrega, PDF) solo vía PEDIDO VINCULADO verificado por la server
  action (minimización de datos en el contexto: el modelo nunca ve campos sensibles).

## Prueba en vivo (ticket #18011)

`run_worker(8, ctx)` con el contexto del ticket de Jaime Rubilar devolvió un acuse correcto:
saludo por hora, paráfrasis del pedido (drivers LED con modelos y cantidades), número de
ticket citado, plazo de 24 h hábiles e invitación a responder. Sin precios ni plazos de
entrega. Guardrails respetados.
