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
separada** (después del `create`), el `run_worker` escribe sin corromper la creación.

## Paso 0 · Suprimir el acuse del core para el equipo 8

El acuse del core es la plantilla de la etapa de entrada. La etapa "Nuevo" original
(`helpdesk.stage` id **1**) tiene `template_id = 39` y está compartida por 11 equipos.

Se creó una **etapa de entrada propia** para el equipo 8, SIN plantilla:

- `helpdesk.stage` id **55**, nombre "Nuevo", `sequence = 0`, `template_id = False`,
  `team_ids = [8]`.
- Se **removió el equipo 8** de la etapa compartida id 1 (`team_ids = [(3, 8)]`).
- Se migraron los tickets del equipo 8 que estaban en la etapa 1 hacia la 55.

Resultado: un ticket nuevo del equipo 8 entra en la etapa 55 (sin plantilla) => el core
**no manda acuse**. Los otros 10 equipos siguen usando la etapa 1 con su acuse (ahora con el
saludo arreglado). Verificado: al crear un ticket de prueba en el equipo 8, no se generó
ningún `auto_comment` de acuse.

## Paso 1 · Ruta de creación

- **Ruta pública:** formulario web de Helpdesk del equipo (`helpdesk.team.use_website_helpdesk_form`),
  publicado en `zonaindustrial.cl`. El controlador crea un `helpdesk.ticket` con `team_id = 8`.
- **Etapa por defecto:** la de menor `sequence` entre las etapas del equipo => id **55** "Nuevo".
- **Asignación:** el equipo asigna automáticamente (hoy a Giuliana Velarde, `res.users`).

## Paso 2 · La automatización genera y envía

`base.automation` id **127** — "ZI · Acuse de ticket con IA"

| Atributo | Valor |
|---|---|
| `model_id` | 870 (`helpdesk.ticket`) |
| `trigger` | `on_create` |
| `filter_domain` | `[("team_id", "=", 8)]` |
| `state` | `code` |
| `active` | `True` |

Lógica: por cada ticket del equipo 8 recién creado:

1. Arma el `ctx` (asunto limpio sin el prefijo `[CONTACTO][Nombre]`, mensaje original desde
   `description` o el primer `mail.message`, contacto, empresa, historial de compras del
   `commercial_partner_id`, y el **saludo por hora de Chile**).
2. `res = env['zi.ai.worker'].run_worker(8, ctx)` → limpia fences → guarda en
   `x_studio_zi_ai_ticket_ack`.
3. Guard de email: si el destino está vacío o es casilla DTE `duemint`, NO envía y deja nota
   interna en el ticket.
4. `env['mail.template'].browse(39).send_mail(ticket.id)`.

Todo dentro de `try/except`: si el worker falla, el campo queda vacío y el envío igual ocurre
con el genérico cálido de la plantilla. La creación del ticket nunca se corta.

### Código completo de la automatización 127

```python
# ZI - ACUSE DE TICKET CON IA (equipo Contacto Sitio web, id 8)
# Interviene en el ENVIO (post-creacion, transaccion separada). La etapa de
# entrada del equipo 8 (id 55) NO tiene plantilla, asi que el core no manda
# acuse; aqui generamos el cuerpo con el worker id 8 y enviamos la plantilla
# 39, que renderiza x_studio_zi_ai_ticket_ack para el equipo 8. Si la IA falla,
# el campo queda vacio y la plantilla usa su saludo generico calido por hora.
# La creacion del ticket NUNCA se corta: todo va en try/except.
nl = chr(10)
tmpl = env['mail.template'].browse(39)
tickets = records if records else record
for ticket in tickets:
    if not ticket.team_id or ticket.team_id.id != 8:
        continue
    try:
        nm = (ticket.partner_id.name if ticket.partner_id else '') or ticket.partner_name or ''
        if nm and '@' in nm:
            nm = nm.split('@')[0]
        contacto = nm or 'cliente'
        empresa = (ticket.partner_id.commercial_partner_id.name if ticket.partner_id else '') or ''
        asunto = ticket.name or ''
        if ']' in asunto:
            asunto = asunto.rsplit(']', 1)[-1].strip() or (ticket.name or '')
        correo = (ticket.description or '').strip()[:3000]
        if not correo:
            m = env['mail.message'].sudo().search([('model', '=', 'helpdesk.ticket'), ('res_id', '=', ticket.id), ('message_type', 'in', ['email', 'comment'])], order='id asc', limit=1)
            correo = (m.body or '').strip()[:3000] if m else ''
        if not correo:
            correo = '(sin texto adicional, solo el asunto)'
        cp = ticket.partner_id.commercial_partner_id if ticket.partner_id else False
        hist_txt = 'Sin ficha de cliente vinculada o sin compras previas. Primer contacto, no inventes una relacion.'
        if cp:
            nfac = env['account.move'].sudo().search_count([('commercial_partner_id', '=', cp.id), ('move_type', '=', 'out_invoice'), ('state', '=', 'posted')])
            if nfac:
                hist_txt = 'Cliente con relacion previa: ' + str(nfac) + ' compras facturadas. Reconocelo con calidez en media frase, sin montos ni fechas.'
        hcl = datetime.datetime.now(timezone('America/Santiago')).hour
        saludo_h = 'Buenos dias' if hcl < 12 else ('Buenas tardes' if hcl < 20 else 'Buenas noches')
        ctx = 'CONTEXTO DEL ACUSE DE RECIBO, todo lo necesario, no necesitas consultar nada mas.' + nl
        ctx = ctx + 'Numero de ticket: ' + (ticket.ticket_ref or '') + nl
        ctx = ctx + 'Asunto de la solicitud: ' + asunto + nl
        ctx = ctx + 'Contacto (abre con el saludo horario y su nombre): ' + contacto + nl
        ctx = ctx + 'Empresa: ' + (empresa or 'no especificada') + nl
        ctx = ctx + 'Plazo de respuesta comprometido: dentro de las proximas 24 horas habiles' + nl
        ctx = ctx + 'Historial del cliente, contexto interno, sin montos ni fechas en el correo:' + nl + hist_txt + nl
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

## Paso 3 · El campo

`ir.model.fields` id **22265**

| Atributo | Valor |
|---|---|
| `name` | `x_studio_zi_ai_ticket_ack` |
| `model` | `helpdesk.ticket` (`model_id` 870) |
| `ttype` | `html` |
| `store` | `True` (campo normal, NO calculado) |
| `state` | `manual` |
| `field_description` | "ZI Acuse ticket IA" |

> Nota: se intentó hacerlo `compute` (calculado bajo demanda al renderizar) para no usar la
> automatización, pero llamar al worker dentro del compute rompe la creación del ticket.
> Descartado. El campo es un `html` normal que puebla la automatización 127.

## Paso 4 · La plantilla de correo

`mail.template` id **39** — "Servicio de asistencia: solicitar reconocimiento"
(XML id `helpdesk.new_ticket_request_email_template`). Se modificó su `body_html`:

- **Rama equipo 8 con campo poblado:** renderiza `object.x_studio_zi_ai_ticket_ack` + botón
  "Ver el ticket" + firma "Equipo Zona Industrial".
- **Rama fallback (todos los demás, o equipo 8 si el campo quedó vacío):** saludo por hora
  calculado desde `create_date` (Chile ~UTC-4) + nombre del cliente + número de ticket +
  invitación a responder. **Ya no dice "Estimado/a Madam/Sir".**

El `body_html` completo (nuevo y original para revertir) está en
`05-operacion-pruebas-reversion.md`.

## Paso 5 · La respuesta del cliente (sin cambios)

El cliente responde al correo → el hilo vuelve al chatter del ticket como `mail.message`
tipo `email` (comportamiento estándar de Odoo, threading por message-id). No se tocó.
