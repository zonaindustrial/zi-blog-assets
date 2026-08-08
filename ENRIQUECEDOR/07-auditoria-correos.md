# 07 · Auditoría de correos salientes (últimas 4 semanas)

Lectura por MCP de `mail.message` (model `sale.order`, `message_type=email`) desde el
2026-07-11. Volumen: **1.435 correos**, la mayoría "Seguimiento de Cotización" (worker 2).
Vendedores activos: Ivo Rojas, Luis Landaeta, Loredana Palumbo, Nicolás Díaz, Samantha Juarez,
Marco Bizama, Martín Napanga, Giuliana Velarde.

## Hallazgos

### 1. Rol del vendedor: NO se usaba (corregido)
Ningún enriquecimiento incorporaba el cargo del vendedor. El contexto solo pasaba el nombre
(para primera persona y género). El **rol real existe** en `res.partner.function` del usuario
vendedor, por ejemplo:
- Luis Landaeta: "consultor de ventas Fuentes de poder"
- Loredana Palumbo: "Consultora de ventas productos HTF"
- Samantha Juarez: "Ejecutiva de venta portales B2B"
- Ivo Rojas: "Consultor de ventas Conectividad"
- Nicolás Díaz: "consultor de ventas Fuentes de poder"

### 2. Género del vendedor: se adivinaba por el nombre (corregido)
Los prompts pedían "ajusta el género a su nombre", lo que falla con nombres ambiguos y produce
tono femenino para vendedor masculino. **Fix:** pasar el género EXPLÍCITO en el contexto,
derivado de la marca del cargo (Consultora/Ejecutiva vs Consultor/Ejecutivo) más un mapa del
equipo, y el prompt ahora usa ese género y tiene prohibido adivinar.

### 3. "Estimado" en el saludo: pese a estar anulado (corregido en workers)
La IA enriquecida ya usaba saludo por hora ("Buenas tardes, César."). Pero:
- Los **prompts** de los workers 1, 2 y 3 aún decían "Estimado si es masculino, Estimada si es
  femenino". Corregido: ahora saludo por hora obligatorio, prohibido "Estimado".
- El "Estimado/a Nombre" de las capturas es el **fallback estático de las plantillas** (cuando
  el campo IA quedó vacío) y de plantillas tipo "Nueva etapa: Presupuesto enviado". PENDIENTE
  de corregir en las plantillas (ver abajo).

## Cambios aplicados (2026-08-08)

- **Worker 1 (email), 2 (seguimiento), 3 (vencimiento):** prompt actualizado.
  1. Saludo por hora obligatorio, prohibido "Estimado".
  2. Usa el género del vendedor explícito del contexto, prohibido adivinar por el nombre.
  3. Incorpora el rol o cargo del vendedor con naturalidad.
- **Server action 7554 (seguimiento):** inyecta al contexto el género (marca del cargo + mapa)
  y el rol (`res.partner.function`) del vendedor.

### Validación (worker 2, vendedor Luis Landaeta, masculino, rol fuentes de poder)

> Buenos días, Jorge. Le escribo en relación con la cotización ZI-139164... Como su
> **consultor de ventas de fuentes de poder**, estoy disponible para resolver cualquier duda
> técnica, ajustar cantidades o coordinar una llamada...

Saludo por hora, género masculino correcto, rol reflejado, sin "Estimado", sin raya larga.

## Pendiente (siguiente lote)

1. **Inyectar rol + género en las demás acciones** (mismo bloque que 7554): correo inicial
   **1579**, vencimiento **7555**, y si se desea acuse lead **7579**, venta y entrega. Bloque:
   ```python
   seller_p = order.user_id.partner_id if order.user_id else False
   rol_vend = (seller_p.function or '') if seller_p else ''
   _rl = rol_vend.lower()
   if 'consultora' in _rl or 'ejecutiva' in _rl or 'vendedora' in _rl or 'asesora' in _rl or 'jefa' in _rl:
       gen_vend = 'femenino'
   elif 'consultor' in _rl or 'ejecutivo' in _rl or 'vendedor' in _rl or 'asesor' in _rl or 'jefe' in _rl:
       gen_vend = 'masculino'
   else:
       gen_vend = {'Ivo Rojas': 'masculino', 'Luis Landaeta': 'masculino', 'Loredana Palumbo': 'femenino', 'Samantha Juarez': 'femenino', 'Ivo Rojas': 'masculino'}.get(vendedor, '')
   # y agregar al ctx: 'Genero del vendedor...' y 'Rol o cargo del vendedor...'
   ```
   Nota interina: worker 3 (vencimiento) ya exige género explícito; mientras 7555 no lo pase,
   el aviso de vencimiento sale en género neutro para el vendedor (seguro, sin misgendering).
2. **Fallbacks estáticos de las plantillas** que aún dicen "Estimado/a": plantilla 111
   (seguimiento), 141 (inicial), la de vencimiento y las "Nueva etapa: Presupuesto enviado".
   Cambiar a saludo por hora, como se hizo en la plantilla 39 (ticket). Hay que ubicar la
   plantilla de "Nueva etapa" (cambio de etapa de sale.order) que sigue usando "Estimado/a".
3. **Confirmación de dirección de entrega (nuevo):** el correo de envío de cotización debe
   LEER la dirección de despacho del cliente y pedir SIEMPRE que confirme si es correcta, en
   TODOS los casos (chatbase o cotización por backend). Requiere: leer `partner_shipping_id`
   de la orden y agregar un bloque de confirmación en la plantilla 141 (y en el flujo de
   chatbase). Definir texto y ubicación antes de implementar.

## Sugerencias de mejora en plantillas aún no tocadas

- Unificar TODAS las plantillas de correo a cliente al saludo por hora (una sola convención).
- Quitar el branding "con tecnología de Odoo" de los pies donde aún aparezca.
- Revisar las plantillas "Nueva etapa" para que el cuerpo enriquecido y el estático no
  produzcan doble saludo (uno estático "Hola Nombre," y otro de la IA).
