# 02 · Flujo del acuse de ticket con IA (paso a paso)

## Por qué se interviene en el ENVÍO y no en la creación

El acuse nativo de Odoo lo dispara el **core del helpdesk en el mismo `create()` del ticket**
(la plantilla de la etapa de entrada se renderiza y envía como parte de la creación). Dos
consecuencias:

1. Un campo calculado que llame a la IA durante la creación **rompe la transacción** (probado:
   el `create` falla). Por eso NO se interviene la creación.
2. Una automatización `on_create` corre *después* de que el core ya envió, así que "llega
   tarde" al correo. Poblar un campo y confiar en el envío del core NO sirve.

**Solución:** quitarle al equipo 8 el envío automático del core y que una automatización
post-creación genere y envíe el acuse. Como la automatización corre en una **transacción
separada** (después del `create`), el `run_worker` escribe sin corromper la creación. Así:

- La creación del ticket corre como siempre.
- Se interviene en el paso del ENVÍO del correo.
- Si la IA o el flujo falla, no corta: se envía el genérico cálido.
- Un solo correo por ticket (el core está suprimido para el equipo 8).

## Paso 0 · Suprimir el acuse del core para el equipo 8

El acuse del core es la plantilla de la etapa de entrada. La etapa "Nuevo" original
(`helpdesk.stage` id **1**) tiene `template_id = 39` y está compartida por 11 equipos.

Se creó una **etapa de entrada propia** para el equipo 8, SIN plantilla:

- `helpdesk.stage` id **55**, nombre "Nuevo", `sequence = 0`, `template_id = False`,
  `team_ids = [8]`.
- Se **removió el equipo 8** de la etapa compartida id 1 (`team_ids = [(3, 8)]`).
- Se migraron los tickets del equipo 8 que estaban en la etapa 1 hacia la 55.

Resultado: un ticket nuevo del equipo 8 entra en la etapa 55 (sin plantilla), el core **no
manda acuse**. Los otros 10 equipos siguen usando la etapa 1 con su acuse (ahora con el saludo
arreglado). Verificado: al crear un ticket de prueba en el equipo 8, no se generó ningún
`auto_comment` de acuse.

## Paso 1 · Ruta de creación

- **Ruta pública:** formulario web de Helpdesk del equipo (`helpdesk.team.use_website_helpdesk_form`),
  publicado en `zonaindustrial.cl`. El controlador crea un `helpdesk.ticket` con `team_id = 8`.
- **Etapa por defecto:** la de menor `sequence` entre las etapas del equipo, id **55** "Nuevo".
- **Asignación:** el equipo asigna automáticamente (hoy a Giuliana Velarde, `res.users`).

## Paso 2 · La automatización genera y envía

`base.automation` id **127**: "ZI · Acuse de ticket con IA"

| Atributo | Valor |
|---|---|
| `model_id` | 870 (`helpdesk.ticket`) |
| `trigger` | `on_create` |
| `filter_domain` | `[("team_id", "=", 8)]` |
| `state` | `code` |
| `active` | `True` |

Lógica por cada ticket del equipo 8 recién creado:

1. **Guard de idempotencia:** si `x_studio_zi_ai_ticket_ack` ya tiene contenido, no reprocesa
   ni reenvía (evita dos correos).
2. **Resuelve el cliente:** usa `ticket.partner_id`; si no hay, busca `res.partner` por
   `partner_email`. Determina **NUEVO vs EXISTENTE** (existente = tiene RUT o compras
   facturadas). Reúne los datos que ya tenemos (RUT, razón social, dirección, teléfono).
3. **Arma el `ctx`:** asunto limpio, nombre (del partner o del prefijo `[CONTACTO][Nombre]`),
   mensaje original, tipo de cliente + datos/faltantes, saludo por hora de Chile, y la
   instrucción de **no comprometer plazos**.
4. `res = env['zi.ai.worker'].run_worker(8, ctx)`, limpia fences, guarda en el campo.
5. **Guard de email:** si el destino está vacío o es casilla DTE `duemint`, no envía y deja
   nota interna.
6. `env['mail.template'].browse(39).send_mail(ticket.id)`.

Todo dentro de `try/except`: si el worker falla, el campo queda vacío y el envío igual ocurre
con el genérico cálido de la plantilla. La creación del ticket nunca se corta.

### Código completo de la automatización 127 (vivo)

```python
# ZI - ACUSE DE TICKET CON IA (equipo Contacto Sitio web, id 8)
# Interviene en el ENVIO (post-creacion, transaccion separada). La etapa de
# entrada del equipo 8 (id 55) NO tiene plantilla, asi que el core no manda
# acuse; aqui generamos el cuerpo con el worker id 8 y enviamos la plantilla 39.
# Detecta cliente NUEVO vs EXISTENTE leyendo res.partner (como los otros
# templates) y lo pasa al contexto. Si la IA falla, el campo queda vacio y la
# plantilla usa su saludo generico calido por hora. La creacion del ticket
# NUNCA se corta: todo va en try/except. Guard de idempotencia para no enviar
# dos veces.
nl = chr(10)
tmpl = env['mail.template'].browse(39)
tickets = records if records else record
for ticket in tickets:
    if not ticket.team_id or ticket.team_id.id != 8:
        continue
    if ticket.x_studio_zi_ai_ticket_ack:
        continue
    try:
        partner = ticket.partner_id
        if not partner and ticket.partner_email:
            partner = env['res.partner'].sudo().search([('email', '=ilike', ticket.partner_email)], limit=1)
        cp = partner.commercial_partner_id if partner else False
        raw = ticket.name or ''
        asunto = raw
        nom_subj = ''
        if ']' in raw:
            asunto = raw.rsplit(']', 1)[-1].strip() or raw
            segs = raw.split(']')
            if len(segs) >= 2 and '[' in segs[1]:
                c = segs[1].split('[')[-1].strip()
                if c and '@' not in c:
                    nom_subj = c
        nm = (partner.name if partner else '') or nom_subj or (ticket.partner_name or '')
        if nm and '@' in nm:
            nm = nm.split('@')[0]
        contacto = nm or 'cliente'
        nfac = 0
        if cp:
            nfac = env['account.move'].sudo().search_count([('commercial_partner_id', '=', cp.id), ('move_type', '=', 'out_invoice'), ('state', '=', 'posted')])
        existente = bool(cp and (nfac > 0 or cp.vat))
        datos = []
        if cp:
            if cp.vat:
                datos.append('RUT ' + cp.vat)
            if cp.name and not (cp.name or '').startswith('Empresa '):
                datos.append('razon social ' + cp.name)
            dirp = ', '.join([x for x in [cp.street, cp.city] if x])
            if dirp:
                datos.append('direccion ' + dirp)
            if cp.phone or cp.mobile:
                datos.append('telefono de contacto')
        correo = (ticket.description or '').strip()[:3000]
        if not correo:
            m = env['mail.message'].sudo().search([('model', '=', 'helpdesk.ticket'), ('res_id', '=', ticket.id), ('message_type', 'in', ['email', 'comment'])], order='id asc', limit=1)
            correo = (m.body or '').strip()[:3000] if m else ''
        if not correo:
            correo = '(sin texto adicional, solo el asunto)'
        hcl = datetime.datetime.now(timezone('America/Santiago')).hour
        saludo_h = 'Buenos días' if hcl < 12 else ('Buenas tardes' if hcl < 20 else 'Buenas noches')
        ctx = 'CONTEXTO DEL ACUSE DE RECIBO, todo lo necesario, no necesitas consultar nada mas.' + nl
        ctx = ctx + 'Numero de ticket: ' + (ticket.ticket_ref or '') + nl
        ctx = ctx + 'Asunto de la solicitud: ' + asunto + nl
        ctx = ctx + 'Contacto (abre con el saludo horario y su nombre): ' + contacto + nl
        ctx = ctx + 'Tipo de cliente: ' + ('EXISTENTE' if existente else 'NUEVO') + nl
        if existente:
            ctx = ctx + 'Datos que YA tenemos, NO los pidas: ' + (', '.join(datos) if datos else 'ficha basica registrada') + '. Compras facturadas: ' + str(nfac) + '. Reconoce la relacion con calidez, sin montos ni fechas.' + nl
        else:
            ctx = ctx + 'Es un cliente NUEVO o sin ficha completa. Para poder cotizar, pide con amabilidad en el correo los datos que falten: razon social, RUT, giro, direccion de despacho y un telefono de contacto; y si aplica, detalles del producto (marcas, cantidades, especificaciones). Explica que asi agilizamos su cotizacion.' + nl
        ctx = ctx + 'IMPORTANTE: NO comprometas ningun plazo de respuesta especifico (nada de 24 horas ni fechas). Di que le responderemos a la brevedad.' + nl
        ctx = ctx + 'PEDIDO VINCULADO: ninguno' + nl
        ctx = ctx + 'Mensaje original del cliente (puede venir con HTML sucio, interpretalo y parafrasea la necesidad):' + nl + correo + nl
        ctx = ctx + 'SALUDO HORARIO OBLIGATORIO segun la hora actual de Chile: abre el correo con ' + saludo_h + ' seguido del nombre y una coma (ejemplo: ' + saludo_h + ', ' + contacto + '.).'
        res = env['zi.ai.worker'].run_worker(8, ctx)
        body = (res or '').strip()
        if body[:3] == '```':
            p = body.find(nl)
            if p != -1:
                body = body[p + 1:]
            body = body.strip()
            if body[-3:] == '```':
                body = body[:-3].strip()
        if body:
            ticket.write({'x_studio_zi_ai_ticket_ack': body})
    except Exception as e:
        log('ZI Acuse ticket IA: worker fallo para ticket %s: %s (se envia generico)' % (ticket.ticket_ref, str(e)), level='warning')
    try:
        dest = ''
        if ticket.partner_id and ticket.partner_id.email:
            dest = ticket.partner_id.email
        elif ticket.partner_email:
            dest = ticket.partner_email
        d = (dest or '').lower().strip()
        if (not d) or ('duemint' in d):
            ticket.message_post(body='<b>Acuse IA NO enviado:</b> el cliente no tiene un email valido (vacio o casilla DTE duemint). Conseguir el email real del contacto y reenviar el acuse.')
            continue
        tmpl.send_mail(ticket.id)
    except Exception as e:
        log('ZI Acuse ticket IA: fallo el envio para ticket %s: %s' % (ticket.ticket_ref, str(e)), level='warning')
```

## Paso 3 · Cliente nuevo vs existente (datos del cliente)

La IA no está conectada al MCP. Los datos del cliente los lee la **automatización** desde
`res.partner` (igual que los otros templates) y los pasa al contexto:

- **EXISTENTE** (tiene RUT o compras facturadas): se listan los datos que ya tenemos para que
  el worker NO los pida, y reconozca la relación con calidez.
- **NUEVO** (email no registrado o ficha incompleta): el worker pide en el correo los datos que
  faltan para cotizar (razón social, RUT, giro, dirección de despacho, teléfono, y detalles del
  producto). Así se gana tiempo para preparar la cotización.

## Paso 4 · El campo

`ir.model.fields` id **22265**

| Atributo | Valor |
|---|---|
| `name` | `x_studio_zi_ai_ticket_ack` |
| `model` | `helpdesk.ticket` (`model_id` 870) |
| `ttype` | `html` |
| `store` | `True` (campo normal, NO calculado) |
| `state` | `manual` |

> Nota: se intentó hacerlo `compute` (calculado al renderizar) para no usar la automatización,
> pero llamar al worker dentro del compute rompe la creación del ticket. Descartado.

## Paso 5 · La plantilla de correo

`mail.template` id **39** (XML id `helpdesk.new_ticket_request_email_template`). `body_html`:

- **Rama equipo 8 con campo poblado:** renderiza `object.x_studio_zi_ai_ticket_ack` + botón
  "Ver el ticket" + firma "Equipo Zona Industrial".
- **Rama fallback (otros equipos, o equipo 8 con campo vacío):** saludo por hora calculado
  desde `create_date` + nombre + número de ticket. Ya no dice "Estimado/a Madam/Sir".

El `body_html` completo (nuevo y original para revertir) está en
`05-operacion-pruebas-reversion.md`.

## Paso 6 · La respuesta del cliente (sin cambios)

El cliente responde al correo, el hilo vuelve al chatter del ticket como `mail.message` tipo
`email` (threading estándar de Odoo). No se tocó.
