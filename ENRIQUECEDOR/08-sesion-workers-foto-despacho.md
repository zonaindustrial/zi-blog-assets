# 08 · Sesión 2026-08-09 · Auditoría de workers, foto comercial, despacho y redundancia

Cambios aplicados y **verificados en producción** (Odoo 16, `www.zonaindustrial.cl`) el
2026-08-09. Todos por el framework `zi.ai.*` ya documentado en `01`–`07`.

## 1. Primera persona, género y cargo del vendedor (cierre del lote)

El lote anterior (ver `07`) dejó pendiente inyectar género+rol explícito en las acciones
hermanas de `7554`. Cerrado:

- **Worker 4 (acuse de lead), `system_prompt`:** ya no dice "con el género según su nombre".
  Ahora usa el **género EXPLÍCITO** del contexto (prohibido adivinar por el nombre), 1ª
  persona del vendedor, y refleja el cargo. Probado con Marco Bizama → masculino correcto.
- **Server actions `1579` (correo inicial), `7555` (vencimiento), `7579` (acuse lead):**
  inyectan al `ctx` el **género** (marca del cargo en `res.partner.function` + mapa del
  equipo) y el **rol**. Mismo bloque que `7554`.

Bloque de derivación de género (idéntico en las 4 acciones):

```python
seller_p = <user>.partner_id
rol_vend = (seller_p.function or '') if seller_p else ''
_rl = rol_vend.lower()
if 'consultora' in _rl or 'ejecutiva' in _rl or 'vendedora' in _rl or 'asesora' in _rl or 'jefa' in _rl or 'gerenta' in _rl:
    gen_vend = 'femenino'
elif 'consultor' in _rl or 'ejecutivo' in _rl or 'vendedor' in _rl or 'asesor' in _rl or 'jefe' in _rl or 'gerente' in _rl:
    gen_vend = 'masculino'
else:
    gen_vend = {'Ivo Rojas': 'masculino', 'Nicolás Díaz': 'masculino', 'Luis Landaeta': 'masculino',
                'Marco Bizama': 'masculino', 'Claudio Pérez': 'masculino', 'Martin Napanga': 'masculino',
                'Loredana Palumbo': 'femenino', 'Samantha Juarez': 'femenino', 'Giuliana Velarde': 'femenino'}.get(vendedor, '')
```

### Roster de ejecutivos (género + cargo consolidado)

| Ejecutivo | `res.users` | Cargo (`res.partner.function`) | Género |
|---|---|---|---|
| Ivo Rojas | 30 | Consultor de ventas Conectividad | M |
| Nicolás Díaz | 6452 | consultor de ventas Fuentes de poder | M |
| Luis Landaeta | 7289 | consultor de ventas Fuentes de poder | M |
| Loredana Palumbo | 96 | Consultora de ventas productos HTF | F |
| Samantha Juarez | 35 | Ejecutiva de venta portales B2B | F |
| Giuliana Velarde | 18 | Responsable de Ecommerce | F |
| **Marco Bizama** | 3705 | **Consultor de venta de protección y fusibles** (escrito hoy) | M |
| **Claudio Pérez** | 5041 | **Jefe de ventas Agroventas** (escrito hoy) | M |
| **Martin Napanga** | 13 | **Gerente comercial de Protección y fusibles** (escrito hoy) | M |

- **Cargos escritos hoy** en `res.partner.function`: Marco (43396), Claudio (45463), Martin (49).
- **Axon Carquín (user 31, `automatizacion@`) excluido** del roster: ya no trabaja en ZI; es
  cuenta técnica. ⚠️ **Pendiente operativo:** ~397 órdenes de los últimos 2 meses siguen
  asignadas a esa cuenta; conviene reasignar el vendedor.

## 2. Confirmación de dirección de despacho en el correo inicial (en el worker)

Para reducir errores de despacho, el correo inicial (worker 1 / action `1579`) ahora **pide
confirmar la dirección de despacho**. Se resolvió **en el worker** (no en la plantilla):

- **Action `1579`** computa `ship = order.partner_shipping_id`, detecta `es_retiro` (líneas
  `is_delivery` con "retiro"/"local") y arma `dir_desp`; inyecta `MODALIDAD DE ENTREGA` al `ctx`
  en 3 ramas: **retiro** (lo menciona, no pide dirección), **despacho con dirección** (pide
  confirmar respondiendo el correo), **sin dirección** (pide indicarla).
- **Worker 1 `system_prompt`:** nueva regla "Confirmación de despacho" que usa `MODALIDAD DE
  ENTREGA` para pedir la confirmación (o mencionar el retiro), en una sola frase, sin formulario.
- Probado: despacho → *"le pido que confirme… si la dirección registrada, Av. Los Industriales
  1234, Antofagasta, es correcta, o bien nos indique la que corresponda."*

## 3. Redundancia: aviso "Actualización de fechas de su pedido" al cliente (frenado)

Auditoría de 2 meses (2.945 correos sobre `sale.order`): el aviso **"Actualización de fechas de
su pedido"** se repetía hasta 6× sobre el mismo pedido. Origen: **cron 149 → server action
`1603`** (plantilla 144 en modo `reprogramacion`), un cron custom de ZI.

- **`ir.cron` 149 desactivado** (`active = False`). El cliente ya no recibe ese aviso.
- Intactos: disponibilidad/arribo (cron 148, "Sus productos ya están disponibles"), aviso a
  **vendedores** por cambio de fecha (cron 151) y la notificación **nativa del módulo de
  inventario**.
- **Reversión:** `ir.cron` 149 → `active = True`.

## 4. Foto comercial (`x_studio_prospecto_ai`) + cross-selling en los 8 workers

"La foto" del cliente es el campo `res.partner.x_studio_prospecto_ai` ("Orientación Prospecto":
perfil, rubro, necesidades, líneas afines). Objetivo: **hacer crecer la venta** ofreciendo
productos, marcas y áreas **complementarias** en cada comunicación, cordial, corto y directo,
**usando los insights sin citar scores**.

- **`system_prompt` de los 8 workers:** se agregó la sección `=== CRECIMIENTO COMERCIAL ===`
  (usar la ORIENTACION DEL CLIENTE para ofrecer 1–2 complementarios con criterio; una sugerencia
  basta; no forzar; respetar límites del correo, sin precios/stock inventados; **nunca** citar
  scores/etiquetas/datos internos).
- **Foto inyectada al `ctx` de todas las acciones:**
  - Ya la tenían: `1579` (W1), `7554` (W2), `7555` (W3), `7579` (W4).
  - Inyectada hoy: **`7640`** (carrito W5), **`7662`** (venta W6), **`7735`** (entrega W7),
    **`base.automation 127`** (ticket W8). Línea añadida:
    `ORIENTACION DEL CLIENTE (insumo interno para ofrecer productos, marcas o areas
    complementarias con criterio; NO la cites literal ni menciones scores): <foto o 'no disponible'>`
- Probado (worker 6, cliente rubro construcción): cierra con *"Si necesita complementar con
  protección para sus tableros (breakers, fusibles o riel DIN), con gusto le cotizamos de
  inmediato."* — usa el rubro de la foto, ofrece líneas afines, no cita el score.

## 5. Plantillas AI-aware (PENDIENTE de aplicar) — doble saludo / contenido duplicado

La plantilla es el shell que **siempre** se renderiza e **inserta** el cuadro IA; su saludo
estático es el **respaldo ante falla de IA**. Cuando hay cuadro IA, la plantilla debe **ocultar
la estructura fija que duplique** (doble saludo, doble pregunta); si el cuadro llega vacío, vuelve
a modo normal.

Estado por plantilla:
- **`mail.template` 55 (vencimiento):** OK, no requiere cambio (su "Estimado/a" ya está solo en
  el `t-else` del cuadro IA).
- **`mail.template` 141 (inicial):** el saludo "Estimado/a Nombre," + intro se muestran **siempre**
  y el cuadro IA vuelve a saludar → **doble saludo**. Fix: envolver saludo+intro en
  `t-if not has_ai` (respaldo) y mostrar solo el cuadro IA cuando existe. **No** agregar caja de
  despacho a la plantilla (el worker 1 ya la pide).
- **`mail.template` 111 (seguimiento):** el `<h1>` "Hola Nombre, ¿en qué etapa se encuentra
  nuestra propuesta?" se muestra **siempre** y el worker 2 saluda + pregunta lo mismo → **doble
  saludo + doble pregunta**. Fix: `<h1>` condicional a `object.x_studio_zi_ai_followup` → con IA,
  título neutro ("Seguimiento de su cotización ZI-…"); sin IA, el saludo+pregunta actual.

**Hallazgo técnico (importante para aplicar):** `mail.template.body_html` es `translate=True`.
Escribirlo desde una **server action** (`run()`) **no persiste** (ni `write` ni
`update_field_translations`). **Sí** persiste vía **write directo del ORM** (equivalente a
`odoo_write` del MCP). Por eso este cambio quedó pendiente: requiere un write directo del cuerpo
completo. Aplicar 141 y 111 con ese método, validando el render en vivo con y sin cuadro IA.

## 6. Pendientes

1. **Aplicar** las plantillas AI-aware 141 y 111 (write directo del `body_html`).
2. **Reasignar** el vendedor de las ~397 órdenes de la cuenta técnica `automatizacion@` (Axon).
3. **Enriquecer con IA** el aviso "Sus productos ya están disponibles" (arribo, cron 148 fase 2,
   plantilla 144 modo `arribo`) — hoy es nativo, sin IA; ~100 correos/2 meses, alta intención.
4. Versionar toda la lógica (acciones, plantillas, campos, crons) en `zi-odoo-addons`.
