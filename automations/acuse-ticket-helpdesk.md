# Acuse de ticket enriquecido con IA (Helpdesk "Contacto Sitio web")

Enriquecimiento del **primer correo de respuesta** a los tickets del equipo Helpdesk
`Contacto Sitio web` de Odoo, generado por un worker de IA en n8n siguiendo la misma
fórmula que el resto de los `ZI Worker - *`.

- **Workflow n8n:** `ZI Worker - Acuse Ticket Helpdesk` (`ZDj8Wu1BGBBwFCvq`)
- **Estado:** inactivo / invocable (no envía correos por sí solo)
- **Modelo:** `gpt-5.4-mini`

## Motivación (auditoría del acuse actual)

El acuse automático que hoy manda Odoo tiene problemas:

1. **Bug de idioma:** empieza con `Estimado/a Madam/Sir,` (fallback en inglés de Odoo sin
   traducir). Sale en todos los acuses.
2. **Genérico:** no saluda según la hora, no agradece de forma cálida, no demuestra que se
   entendió el motivo del cliente.
3. **Firma interna:** cierra como "Equipo Contacto Sitio web" (nombre del equipo de
   helpdesk, no comercial) y arrastra el branding de Odoo.

## Qué hace el worker

Replica la fórmula ZI Worker:

```
Cuando lo llaman (executeWorkflowTrigger, passthrough)  ─┐
Probar (manualTrigger)                                  ─┴─▶ Ticket (mcpClient: odoo_get_ticket)
   ─▶ Preparar (Code) ─▶ Redactor Acuse (Agent) ◀── GPT (gpt-5.4-mini)
```

1. **Ticket** — lee el ticket en Odoo por MCP (`odoo_get_ticket`, scope acotado: nombre,
   descripción, partner). Nada sensible.
2. **Preparar** (Code) — calcula el **saludo según la hora de Chile** (Buenos días / Buenas
   tardes / Buenas noches), limpia el asunto y la descripción, y arma el `system` + `prompt`
   con los guardrails.
3. **Redactor Acuse** (Agent) — redacta con IA y devuelve HTML con **dos bloques**:
   - `<!--ACUSE_CLIENTE-->` correo listo para el cliente (auto-enviable, sin datos sensibles).
   - `<!--BORRADOR_INTERNO-->` nota para el vendedor (motivo detectado, categoría, qué falta,
     acción sugerida). No se envía al cliente.

## Guardrails de confidencialidad

El `system` prohíbe de forma estricta:

- costos, márgenes, precios de compra;
- proveedores / fabricantes;
- niveles de stock o inventario interno;
- datos o pedidos de **otros** clientes;
- inventar precios, plazos o disponibilidad.

Datos de un pedido/cotización del propio cliente (estado, **fecha de entrega comprometida**,
PDF de factura o cotización) **solo** si el llamador entrega el objeto `PEDIDO_VINCULADO` ya
verificado (ticket ligado a un pedido y correo del solicitante coincidente). Si no viene, el
worker no menciona ningún pedido ni fecha. La minimización de datos se hace en el llamador:
el modelo nunca recibe campos sensibles.

## Contrato de entrada (invocación)

```json
{
  "ticket_id": 17930,
  "ticket_ref": "18012",
  "pedido": null
}
```

`pedido` es opcional y, cuando existe, solo debe traer campos seguros, por ejemplo:

```json
{ "referencia": "S01234", "estado": "en preparacion", "fecha_entrega": "2026-08-20", "pdf": true }
```

## Modo híbrido (decisión de negocio)

- **Auto-enviable:** el bloque `ACUSE_CLIENTE` (saludo + agradecimiento + motivo entendido +
  próximos pasos). No lleva datos sensibles.
- **Revisión del vendedor:** el bloque `BORRADOR_INTERNO` se publica como nota interna en el
  ticket para que el ejecutivo lo apruebe/complete.

## Pasos pendientes para producción

1. En el nodo **GPT**, cambiar la credencial `n8n free OpenAI API credits` por la credencial
   OpenAI principal de ZI.
2. Construir el **orquestador** (automatización de Odoo o flujo n8n) que: dispare al crear el
   ticket, resuelva el `PEDIDO_VINCULADO` con lista blanca de campos, invoque este worker,
   parta el output por los marcadores, **envíe** `ACUSE_CLIENTE` y **publique**
   `BORRADOR_INTERNO` como nota interna.
3. Corregir en paralelo la plantilla nativa de Odoo (el `Estimado/a Madam/Sir`) como respaldo.
