# 05 · Operación, pruebas y reversión

## Cómo probar

1. Crear un ticket en el equipo 8 (formulario web, o por ORM):
   ```python
   env['helpdesk.ticket'].create({
       'name': '[CONTACTO][Prueba]Test acuse',
       'team_id': 8,
       'description': '<p>Necesito cotizar 3 UPS de 3 kVA.</p>',
       'partner_email': 'tu-correo@ejemplo.cl',   # externo; NO uses @zonaindustrial.cl si quieres ver el guard
   })
   ```
2. Verificar en el chatter del ticket que el único acuse es el enriquecido (mensaje tipo
   `email`), que abre con "Buenos días/tardes/noches", parafrasea el pedido y cita el número.
3. Revisar `zi.ai.worker` id 8: `last_run` / `last_state` = `ok` / `last_result`.

> El guard de email salta el envío si el destino está vacío o contiene `duemint`, y deja una
> nota interna en el ticket. Los correos internos `@zonaindustrial.cl` SÍ reciben (se usó
> francisco@zonaindustrial.cl para las pruebas).

## Prueba end-to-end realizada (2026-08-08)

- Supresión del core verificada: ticket #18017 en etapa 55, sin `auto_comment` de acuse.
- Envío enriquecido verificado: ticket #18018 recibió un único acuse con IA (ejemplo en
  `03-worker-prompt-guardrails.md`).
- Los tickets de prueba (#18014, #18016, #18017, #18018) fueron eliminados.
- Tickets reales migrados a la etapa 55 (siguen "Nuevo", sin cambio funcional): #18011,
  #18012, #17607, #17229.

## Manejo de fallos (nunca corta)

| Falla | Comportamiento |
|---|---|
| El worker/IA falla o devuelve vacío | `try/except` lo captura, deja `last_state=error`; el campo queda vacío y la plantilla envía el **genérico cálido por hora** (rama `else`). |
| El envío falla | `try/except` lo captura y hace `log(...)`; la creación del ticket ya ocurrió y no se ve afectada. |
| Email inválido (vacío o duemint) | No envía; deja nota interna pidiendo el email real. |
| La creación del ticket | Nunca se interviene ni se bloquea: la automatización corre después. |

## Reversión (rollback)

Para desactivar todo y volver al comportamiento nativo:

1. **Desactivar la automatización:**
   `base.automation` 127 → `active = False`.
2. **Devolver el equipo 8 a la etapa compartida** (para que el core vuelva a mandar el acuse):
   - `helpdesk.stage` 1 → `team_ids = [(4, 8)]` (re-agregar equipo 8).
   - Migrar los tickets del equipo 8 que estén en la etapa 55 de vuelta a la 1.
   - (Opcional) archivar/eliminar `helpdesk.stage` 55.
3. **Restaurar la plantilla 39** con el `body_html` original (abajo).
4. (Opcional) `zi.ai.worker` 8 → `active = False`. El campo `x_studio_zi_ai_ticket_ack`
   (22265) puede quedarse; es inocuo.

### `body_html` NUEVO de la plantilla 39 (el que está vivo)

```html
<div>
    <t t-set="hcl" t-value="((object.create_date.hour - 4) % 24) if object.create_date else 12"/>
    <t t-set="saludo" t-value="'Buenos días' if (hcl >= 5 and hcl < 12) else ('Buenas tardes' if hcl < 20 else 'Buenas noches')"/>
    <t t-set="nom" t-value="object.sudo().partner_id.name or ''"/>
    <t t-set="nom" t-value="'' if '@' in nom else nom"/>
    <t t-if="object.team_id.id == 8 and object.x_studio_zi_ai_ticket_ack">
        <t t-out="object.x_studio_zi_ai_ticket_ack">Acuse personalizado</t>
        <div style="text-align: center; padding: 16px 0px 16px 0px;">
            <a style="box-sizing:border-box;background-color: #17375A; padding: 8px 16px 8px 16px; text-decoration: none; color: #fff; border-radius: 5px; font-size:13px;" t-att-href="object.get_portal_url()" target="_blank">Ver el ticket</a><br>
        </div>
        Gracias,<br><br>
        Equipo Zona Industrial.
    </t>
    <t t-else="">
        <t t-out="saludo">Hola</t><t t-if="nom">, <t t-out="nom"/></t>.<br><br>
        Gracias por escribirnos. Recibimos su solicitud
        <t t-if="object.get_portal_url()">
            <a t-attf-href="/my/ticket/{{ object.id }}/{{ object.access_token }}" t-out="object.name or ''" style="text-decoration:none;box-sizing:border-box;color:#008f8c;"></a>
        </t>
        y está siendo revisada por nuestro equipo <t t-out="object.team_id.name or ''">equipo</t>. La referencia de su ticket es <strong style="color:#17375A;"><t t-out="object.ticket_ref or ''">15</t></strong>.<br>
        <div style="text-align: center; padding: 16px 0px 16px 0px;">
            <a style="box-sizing:border-box;background-color: #17375A; padding: 8px 16px 8px 16px; text-decoration: none; color: #fff; border-radius: 5px; font-size:13px;" t-att-href="object.get_portal_url()" target="_blank">Ver el ticket</a><br>
        </div>
        Si desea agregar más detalles, puede responder directamente a este correo y con gusto lo atenderemos.<br><br>
        <t t-if="object.team_id.show_knowledge_base">
            También puede visitar nuestro <a t-attf-href="{{ object.team_id.get_knowledge_base_url() }}" style="text-decoration:none;box-sizing:border-box;color:#008f8c;">centro de ayuda</a>.<br><br>
        </t>
        Un cordial saludo,<br><br>
        Equipo <t t-out="object.team_id.name or 'Zona Industrial'">Zona Industrial</t>.
    </t>
</div>
```

### `body_html` ORIGINAL de la plantilla 39 (para restaurar en rollback)

```html
<div>
    Estimado/a <t t-out="object.sudo().partner_id.name or 'Madam/Sir'">señor/a</t>,<br><br>
    Hemos recibido su solicitud
    <t t-if="object.get_portal_url()">
        <a t-attf-href="/my/ticket/{{ object.id }}/{{ object.access_token }}" t-out="object.name or ''" style="text-decoration:none;box-sizing:border-box;color:#008f8c;"></a>
    </t>
    y está siendo revisada por nuestro equipo <t t-out="object.team_id.name or ''">Table legs are unbalanced</t> .
    La referencia de su ticket es <t t-out="object.ticket_ref or ''">15</t>.<br>

    <div style="text-align: center; padding: 16px 0px 16px 0px;">
        <a style="box-sizing:border-box;background-color: #875A7B; padding: 8px 16px 8px 16px; text-decoration: none; color: #fff; border-radius: 5px; font-size:13px;" t-att-href="object.get_portal_url()" target="_blank">Ver el ticket</a><br>
    </div>

    Para agregar comentarios adicionales, responda a este correo electrónico<br><br>

    <t t-if="object.team_id.show_knowledge_base">
        Por favor, no dude en visitar nuestro <a t-attf-href="{{ object.team_id.get_knowledge_base_url() }}" style="text-decoration:none;box-sizing:border-box;color:#008f8c;">centro de ayuda</a>. Es posible que encuentre la respuesta a su pregunta.
        <br><br>
    </t>

    Gracias,<br><br>
    Equipo <t t-out="object.team_id.name or 'Helpdesk'">Helpdesk</t>.
</div>
```

## Ideas / pendientes (opcionales)

- **PEDIDO VINCULADO:** hoy la automatización pasa `PEDIDO VINCULADO: ninguno`. Para permitir
  informar fecha de entrega y adjuntar factura/cotización en PDF, resolver en la automatización
  el pedido del propio cliente ligado al ticket (con lista blanca de campos: estado,
  `commitment_date`, PDF) y pasarlo al `ctx`. El worker ya sabe usarlo.
- **Migrar a módulo:** hoy la lógica vive en registros (base.automation, plantilla, campo,
  stage) creados en producción. Idealmente versionar en `zi-odoo-addons` junto a los hermanos.
- **DST Chile:** el saludo del fallback aproxima la hora de Chile con UTC-4 (invierno). En
  verano (UTC-3) el borde del saludo puede correrse una hora. Irrelevante para el worker
  (usa la hora real vía `timezone('America/Santiago')`).
