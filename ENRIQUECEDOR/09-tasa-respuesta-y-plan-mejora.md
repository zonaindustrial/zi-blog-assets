# 09 · Auditoría de tasa de respuesta y plan de mejora (2026-08-09)

Segunda pasada de la auditoría de los últimos 2 meses, con un agregado: **cruzar lo
saliente con lo entrante** para medir qué correos generan mejor **tasa de respuesta**
del cliente. El objetivo es saber qué potenciar, qué es redundante y dónde está la
oportunidad no explotada.

## Metodología

- **Ventana:** desde 2026-06-09 (2 meses), modelo `sale.order`.
- **Saliente (nuestro):** `mail.message` con `message_type='email'` y remitente
  `@zonaindustrial.cl`.
- **Entrante (respuesta del cliente):** `mail.message` con `message_type='email'`,
  remitente externo (subtipo "Conversaciones"; asuntos `Re:`, `RE:`, `RV:` o envío de OC).
- **Clasificación del saliente** por asunto: cotización inicial, seguimiento, pedido
  confirmado, entrega/despacho, aviso de disponibilidad/arribo, actualización de fechas,
  proforma y otros.
- **Atribución de la respuesta:** cada correo entrante se asigna al **último correo
  nuestro que lo precede** en el mismo hilo, dentro de **30 días**. Así una respuesta
  acredita al correo que probablemente la gatilló y no se cuenta dos veces.
- **Alcance / salvedad:** mide **respuesta por correo** (que el cliente conteste), no
  conversión: una orden de compra o una compra pueden llegar por teléfono, WhatsApp o
  un correo nuevo. Sirve para comparar la **efectividad relativa** entre tipos de correo.

## Resultados

Base: **2.364** correos salientes, **593** respuestas de clientes, **378** cotizaciones
con al menos una respuesta.

| Correo | Enviados | Con respuesta | Tasa |
|---|---:|---:|---:|
| **Cotización inicial** | 1.602 | 115 | **7,2 %** |
| Pedido confirmado | 22 | 2 | 9,1 % (n bajo) |
| Entrega / despacho | 118 | 6 | 5,1 % |
| Aviso de disponibilidad / arribo | 97 | 4 | 4,1 % |
| Otros | 183 | 6 | 3,3 % |
| Seguimiento (+3 días) | 174 | 4 | 2,3 % |
| Actualización de fechas | 168 | 1 | 0,6 % |

> Ampliar la ventana de respuesta de 14 a 30 días casi no movió las cifras (el inicial
> pasó de 7,0 % a 7,2 %): las tasas son reales, no un artefacto de la ventana.

## Hallazgos

1. **El correo inicial de cotización es el motor.** Concentra prácticamente todas las
   respuestas del período y tiene la mejor tasa con volumen representativo. Es donde el
   cliente efectivamente contesta.
2. **El seguimiento (+3 días) rinde poco (2,3 %).** Confirma la sospecha de redundancia:
   mucho envío, muy poca respuesta. Volumen que hoy no se traduce en conversación.
3. **"Actualización de fechas" ~0,6 %.** Prácticamente muerto. Valida haber frenado la
   copia al cliente (se mantiene solo el aviso nativo del módulo de inventario).
4. **Oportunidad no explotada:** el **aviso de arribo/disponibilidad** (4,1 %) y el de
   **entrega/despacho** (5,1 %) tienen tracción decente y **hoy no se enriquecen con IA**.
   Son los mejores candidatos a potenciar.

## Plan de mejora (priorizado)

- **P1 · Enriquecer el aviso de arribo/disponibilidad** con IA + cross-selling, con el
  mismo enfoque del inicial (cordial, corto, usando la orientación del cliente). Es la
  mayor oportunidad no explotada: buena tracción y cero enriquecimiento actual.
- **P2 · Rediseñar el seguimiento** para subirlo del 2,3 %: hacerlo **selectivo** (solo
  cotizaciones con monto o probabilidad relevantes) y con **valor real** (novedad de
  stock, gancho, vencimiento próximo), en vez de un envío masivo a los 3 días. Menos
  frecuencia, más pertinencia.
- **P3 · Enriquecer entrega/despacho** como comunicación post-venta (confirmación clara
  + cross-selling de accesorios/complementos), aprovechando su 5,1 %.
- **P4 · Re-medir en 3–4 semanas** para validar el efecto de los cambios sobre la tasa
  de respuesta (idealmente comparando antes/después por tipo de correo).
- **P5 · Pendientes de base** ya documentados: reasignar las órdenes de la cuenta de
  automatización y versionar la lógica en `zi-odoo-addons`.

## ¿Las mejoras de hoy son determinantes?

**En foco, sí; en número, aún por confirmar.**

- Lo aplicado hoy refuerza **exactamente el correo de mayor retorno** (el inicial):
  orientación del cliente (la "foto") + cross-selling, género y rol correctos del
  vendedor, confirmación de la dirección de despacho, y se eliminó el doble saludo tanto
  en el inicial como en el seguimiento. Además se frenó el correo de fechas (0,6 %, puro
  ruido).
- **Interpretación:** las mejoras atacan la palanca correcta —donde el cliente responde—
  y quitan ruido donde no responde. En ese sentido son **estratégicamente determinantes**:
  concentran el esfuerzo en el punto de mayor probabilidad de conversación.
- **Salvedad honesta:** el impacto sobre la tasa **todavía no es medible**. Los cambios
  son de hoy y no hay datos posteriores; toda cifra de "mejora" ahora sería especulación.
  La confirmación empírica requiere la re-medición de P4. Hasta entonces: **determinante
  en dónde apuntan, no probado en cuánto suben.**
