# Auditoría del flujo de compra automática Intcomex

**Fecha:** 2026-08-20 · **Gatillo:** "el sistema de Intcomex no compró"
**Alcance:** `zi-intcomex-scraper` (`src/intcomex_orders.py`), scheduler Cloud Run,
alertas n8n (`ZI Intcomex Alertas`), datos de producción en Odoo (solo lectura).

---

## 1. Veredicto

**Sí compró.** La venta ZI-140346 se confirmó a las 17:30 (hora Chile) y la orden
Intcomex **4960891** quedó colocada a las **17:46**, con la OC09398 confirmada,
`partner_ref = 4960891`, recepción MTZ/IN/01491 creada y fecha prometida
(2026-08-24) escrita. El correo `ORDEN_COLOCADA` salió a las 17:46:54.

Lo que falló es **lo que la venta muestra**: en el chatter de ZI-140346 la única
marca del comprador dice

```
ZI-INTCOMEX-ORDER so_line=204221 sku=PC001ASU71 qty=1.0 numero_orden= estado=colocando
```

`numero_orden` vacío y `estado=colocando`. Quien mire la venta concluye,
correctamente según lo que ve, que el sistema no compró. La confirmación existe
solo del lado de la OC.

---

## 2. Causa raíz (bug real, 100 % reproducible)

En `run()` (`src/intcomex_orders.py`), tras una colocación exitosa:

```python
for it in placeable:
    _mark_line_ordered(mcp_url, it, {"estado": "colocando", ...})   # marca de INTENCIÓN

resultado = placer.place_order(placeable, referencia)

for it in placeable:
    if _line_already_ordered(mcp_url, it["order_id"], it["so_line_id"]):
        continue                                                     # <-- siempre True
    _mark_line_ordered(mcp_url, it, resultado)                       # <-- inalcanzable
```

`_line_already_ordered()` busca el fragmento `ZI-INTCOMEX-ORDER so_line=<id>`, que
es exactamente lo que acaba de escribir la marca de intención. La marca de
confirmación —la que lleva `numero_orden` y `estado`— **nunca se escribe**.

La re-verificación se agregó para el caso "otra corrida colocó en paralelo", pero
desde que la marca de intención se escribe *antes* del checkout (cambio del
2026-08-02) siempre corta.

**Evidencia en producción:** las 3 marcas que existen en Odoo dicen
`estado=colocando`, ninguna trae número:

| Venta | Fecha | Marca | Orden Intcomex real |
|---|---|---|---|
| ZI-136871-2 | 2026-08-06 | `estado=colocando` | 4951444 (en la OC) |
| ZI-139660 | 2026-08-13 | `estado=colocando` | 4955467 (en la OC) |
| ZI-140346 | 2026-08-20 | `estado=colocando` | 4960891 (en la OC) |

**Impacto:** ninguno sobre la plata (la idempotencia se sostiene: la marca existe,
así que no hay recompra). El daño es de trazabilidad y de confianza: toda venta
comprada por el robot se ve como una compra a medias.

**Corrección propuesta (mínima):** escribir la marca de confirmación sin
condición. La barrera contra recompra ya la da la marca de intención, escrita
antes de tocar el carro; volver a consultarla justo después solo puede dar True.

```python
for it in placeable:
    _mark_line_ordered(mcp_url, it, resultado)
```

Alternativa si se quiere conservar el chequeo: pedir en el dominio
`["body", "not like", "estado=colocando"]`, para que busque la marca de
*confirmación* y no cualquier marca.

---

## 3. Por qué se percibe demora

El comprador no reacciona a la confirmación de la venta: corre por cron
(`zi-intcomex-ordenes-sched`, `45 * * * *`). Una venta confirmada a las 17:30 se
compra a las 17:45; una confirmada a las 17:46 espera hasta las 18:45. **La
latencia normal es de 0 a 60 minutos** y hoy fueron 16.

Además, de las 3 colocaciones automáticas de la historia, solo la de hoy cae en
el horario del cron (17:45 Chile); las del 6 y 13 de agosto (22:37 y 22:27 Chile)
fueron ejecuciones supervisadas. Hoy es, con la evidencia disponible, **la
primera compra desatendida del robot**.

Si 60 minutos es demasiado para el compromiso con el cliente, la cadencia se baja
a `*/15 * * * *` sin tocar el código: los topes (`MAX_ORDER_CLP`, `MAX_RUN_CLP`) y
la idempotencia no dependen de la frecuencia.

---

## 4. Hallazgos secundarios

### 4.1 Deriva entre `deploy.sh` y `setup_schedulers.sh` (documentación que miente)
`deploy.sh:139` dice que `zi-intcomex-ordenes-sched (45 * * * *)` **debe estar
ENABLED**. `setup_schedulers.sh` dice lo contrario: que el comprador quedó
**SIN cadencia y en PAUSED** desde la auditoría del 2026-08-02, y no lo crea. La
realidad operativa (la corrida de hoy a las 17:45) le da la razón a `deploy.sh`.
Quien lea `setup_schedulers.sh` para entender el sistema se lleva una idea falsa
del componente que mueve plata. Corresponde alinear el bloque comentado.

### 4.2 `_write_back_po()` no excluye OC canceladas
`_buscar_oc_venta()` y `_sale_manually_ordered()` filtran `state != cancel`;
`_write_back_po()` no. Busca `["origin", "like", order_name]` con `limit 1` y sin
orden explícito, así que puede escribir `partner_ref`, agregar el flete y
corregir precios **sobre una OC cancelada**, dejando la OC viva sin número. Hoy no
ocurre por suerte de ordenamiento (toma la de id más alto, que suele ser la
vigente). Caso que lo destaparía: ZI-138503, que ya tiene la OC09288 cancelada.
Corresponde agregar el mismo filtro de estado y un `order` explícito.

### 4.3 `origin like <nombre>` colisiona por prefijo (latente)
Las tres funciones que cruzan venta ↔ OC usan `like`, que es subcadena: el nombre
`ZI-136871` matchea la OC de `ZI-136871-2`. Hoy no muerde porque la convención de
la casa cancela la venta original al emitir la `-2` (verificado: ZI-136871,
ZI-137328, ZI-138157, ZI-129630, ZI-131689 y ZI-132826 están todas en `cancel`),
y una venta cancelada nunca entra al radar. Queda como riesgo latente: el día que
una venta base siga viva junto a su `-2`, el comprador la saltaría **en silencio**
(ver 4.4) o escribiría el número en la OC equivocada. Arreglo: filtrar en Python
partiendo `origin` por comas y exigiendo igualdad exacta.

### 4.4 El salto por "ya comprada a mano" no alerta
Cuando `_sale_manually_ordered()` da True, la venta se salta y solo queda en el
log y en el resumen de la corrida: no se emite evento a n8n. Un falso positivo
(4.3) sería invisible hasta que el verificador lo levante al día siguiente a las
09:00 con `venta_sin_orden`, y solo si supera las horas de gracia. La red está,
pero con hasta 24 h de retraso.

### 4.5 El módulo que mueve plata no tiene tests
`tests/` cubre red tags, pesos, fotos, categorías, Icecat, duplicados, universo
espejo y el verificador. No hay `test_intcomex_orders.py`. El bug de la sección 2
lo habría atrapado un test de 10 líneas sobre `run()` con un placer falso,
verificando que la marca final trae `numero_orden`.

### 4.6 Observación menor: `date_order` de las OC nace un día atrás
OC09398 (creada 20-08 21:30) quedó con `date_order` 19-08 21:30; lo mismo OC09329
y OC09258. El patrón aparece en las OC que crea la ruta de compra de Odoo, no en
las que crea el comprador, así que queda fuera del alcance de este flujo, pero
desalinea cualquier reporte de compras por fecha.

---

## 5. Lo que se verificó y está sano

- **Radar de compra:** con el criterio real del comprador (universo espejo +
  ventas confirmadas desde 2026-08-02 + almacenables), hoy hay 5 líneas en
  ventana y **las 5 tienen su orden Intcomex**. No hay ninguna venta sin comprar.
- **Guardas de plata:** doble armado (`PUSH_ORDERS=1` + `dry_run=False`), tope por
  venta ($500.000), tope por corrida, retención por stock espejo insuficiente,
  retención por falta de costo espejo, y verificación de contenido del carro por
  cantidad antes de confirmar.
- **Fecha prometida:** la promesa Next Day se calculó bien hoy — colocada 17:46
  (después del corte de 17:00) → día hábil efectivo viernes 21 → entrega lunes 24
  a las 17:00 Chile. `date_planned` quedó en `2026-08-24 21:00 UTC`. Correcto.
- **Parse de totales:** el bug del caso ZI-139660 (el "envío" capturaba el precio
  del producto) está corregido; hoy subtotal 463.751 + IVA 88.113 = total 551.864
  cuadra, y la OC09329 terminó con su flete real de $5.700 en línea aparte.
- **Alertas:** el canal n8n → correo funciona. Agosto: 1 `PIPELINE_ROTO`
  (ZI-138503, resuelto), 2 `STOCK_CON_ERRORES`, 4 `PIPELINE_DEGRADADO`
  (frescura_feed: 136 categorías vs 137 de baseline, pendiente) y 2
  `ORDEN_COLOCADA`.

---

## 6. Prioridad sugerida

| # | Hallazgo | Severidad | Esfuerzo |
|---|---|---|---|
| 1 | Marca de confirmación inalcanzable (§2) | Alta (trazabilidad) | 1 línea |
| 2 | `_write_back_po` sin filtro de cancelada (§4.2) | Media (plata) | 2 líneas |
| 3 | Deriva de documentación del scheduler (§4.1) | Media (operación) | comentario |
| 4 | Sin tests del comprador (§4.5) | Media | 1 archivo |
| 5 | `like` por prefijo (§4.3) + salto silencioso (§4.4) | Baja (latente) | pequeño |
| 6 | Cadencia :45 → `*/15` si se quiere menos espera (§3) | Decisión de negocio | cron |

Los cambios 1, 2, 4 y 5 son en `zonaindustrial/zi-intcomex-scraper`; esta sesión
tiene ese repo en modo lectura.
