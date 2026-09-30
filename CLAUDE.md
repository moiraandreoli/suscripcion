# Proto de suscripción BIMS

Prototipo estático (HTML + CSS + JS vanilla, sin build ni dependencias) del flujo de
suscripción de BIMS. Dos pantallas:

- `index.html` — Resumen de suscripción ("suscripcion-resumen"). Muestra el estado de
  la cuenta, el detalle de consumos y adicionales, y lleva a pagar.
- `pagar-suscripcion.html` — Selección de plan, método de pago (transferencia o
  tarjeta) y confirmación.

Sigue la skill `bims-ui`: todo el CSS del kit está pegado inline en cada `<style>`
(no hay archivos separados), se usan solo clases `bims-*`/variables `--bims-*`, y
todo lo que no existe en prod está marcado en el HTML con
`<!-- NUEVO: no validado contra prod -->`.

## Estados de la suscripción

Un tag junto al título (`#stateTag`) muestra el estado en las dos pantallas:

| Estado       | Color amarillo/verde/rojo | Notas |
|--------------|------------|-------|
| Activo       | verde (`bims-badge--success`)  | |
| Por vencer   | amarillo (`bims-badge--warning`) | |
| Vencido      | rojo (`bims-badge--danger`)   | |
| En revisión  | amarillo + punto que late (`bims-badge--dot`) | Solo se llega por flujo, no está en el selector |

### Transiciones implementadas

- **Activo → Por vencer**: 7 días antes del vencimiento (regla de negocio; el proto no
  calcula fechas, el estado se fuerza con el selector o el querystring).
- **Por vencer → Vencido**: vence sin pago.
- **Por vencer o Vencido + pago con tarjeta → Activo** (ciclo nuevo). Implementado de
  forma universal (cualquier estado + tarjeta → Activo), ya que pagar con tarjeta
  siempre deja la cuenta al día.
- **Vencido + comprobante de transferencia → En revisión** (la cuenta no se bloquea).
- **En revisión + validación de Operaciones → Activo** (no simulado: no hay un panel
  de Operaciones en este proto; se llegaría manualmente cambiando el querystring).
- **Activo o Por vencer + comprobante de transferencia** → el tag **no cambia**; la
  confirmación aclara "Pago en revisión".

### Comprobante rechazado — NO implementado

Buscar `TODO` / el comentario `Comprobante rechazado: NO implementado` en el `<script>`
de ambas pantallas. Estado a definir con negocio (¿vuelve a Vencido? ¿un estado nuevo
"Rechazado"? ¿reintento directo?). No hay UI ni lógica para este caso.

### Cómo se pasa el estado entre pantallas

Query param `?estado=activo|por-vencer|vencido|en-revision` (default: `activo` si
falta o es inválido). `index.html` arma el link de "Ir a pagar" con el estado actual;
`pagar-suscripcion.html` lo lee al cargar (`ESTADO_ORIGEN`, fijo durante toda la
sesión de esa pantalla) y decide a dónde vuelve "Ir al inicio" / "Ver estado de
cuenta" según lo que pasó en el pago (`ESTADO_DESTINO`).

### Selector del proto

En `index.html`, arriba del contenido: `Activo | Por vencer | Vencido`
(`.proto-state-selector`, `<!-- NUEVO -->`). Son links a `?estado=...`, no un control
interactivo sin recarga — cada click navega. Vive solo en `index.html`;
`pagar-suscripcion.html` no lo repite, solo lee el estado que llega por URL.

## Regla de cobro (reemplaza la anterior "el plan queda en Gs. 0 hoy")

- **Activo o Por vencer**: se paga **plan base + adicionales hasta la fecha de pago**
  ("hasta hoy").
- **Vencido**: se paga **plan base + adicionales, pero estos quedan congelados en el
  valor que tenían el día del vencimiento** (dejan de crecer aunque pase el tiempo).
- El monto de "adicionales" en Vencido/En revisión es el mismo que ya se mostraba
  como "Estimado al cierre" en Activo/Por vencer — no es casualidad: es el valor que
  el ciclo alcanza al llegar a la fecha de vencimiento.

### Datos de ejemplo del proto (fijos, no se recalculan por fecha real)

- Hoy: `15-09-2026` · Vence: `18-09-2026` · Plan Mensual: `Gs. 340.000`.
- Adicionales hasta hoy: `Gs. 1.187.400` · Adicionales hasta el vencimiento (congelado): `Gs. 1.326.000`.
- **Activo / Por vencer** → `340.000 + 1.187.400 = Gs. 1.527.400`.
- **Vencido** (plan mensual) → `340.000 + 1.326.000 = Gs. 1.666.000`.
- Plan Semestral (180 días, 20% off): `Gs. 1.632.000` (antes `Gs. 2.040.000`). Solo
  elegible en Vencido → total con semestral: `1.632.000 + 1.326.000 = Gs. 2.958.000`.

### Dónde coinciden los montos (una sola fuente de verdad por pantalla)

- `index.html`: KPI "A pagar (hoy)", tabla de consumos (fila del plan + fila total),
  barra de pago inferior. Todo se recalcula sumando las filas de la tabla
  (`aplicarEstado()` en el script), no hay números duplicados a mano.
- `pagar-suscripcion.html`: Resumen de pago (con sus períodos de fechas), importe de
  transferencia (texto + `data-copy` del botón de copiar), botón de tarjeta,
  confirmación (monto y "Próximo vencimiento"). Todo sale de `recomputeResumen()` /
  la variable `TOTAL`, reactivo al plan elegido (relevante en Por vencer y Vencido,
  donde se puede cambiar 30↔180 días).

## Ciclos: plan prepago, adicionales pospago

El pago de hoy cubre el **plan del ciclo SIGUIENTE**, no el actual — el ciclo nuevo
empieza el día después de que termina el actual (la fecha de vencimiento), no el día
del pago. Los **adicionales** son al revés: se cobran hasta la fecha de pago, y los
días que quedan entre el pago y el vencimiento se suman a la próxima factura (no se
pierden ni se cobran de más).

Dataset fijo del proto: ciclo actual `20-08-2026` al `18-09-2026` (vencimiento), hoy
`15-09-2026`. Con esos datos:

- Adicionales de este pago: **del `20-08-2026` al `15-09-2026`** (Activo/Por vencer)
  o **del `20-08-2026` al `18-09-2026`**, es decir el ciclo completo, cuando están
  congelados (Vencido/En revisión).
- Nota que se muestra bajo el resumen (solo si se puede calcular el ciclo nuevo, ver
  abajo): "Los adicionales del `16-09-2026` al `18-09-2026` se suman a tu próxima
  factura."
- Ciclo nuevo: empieza `19-09-2026` (= vencimiento + 1 día). Plan Mensual (30 días)
  termina `18-10-2026`; Plan Semestral (180 días) termina `17-03-2027`.
- Esa fecha de fin del ciclo nuevo es la que se muestra como "Próximo vencimiento" en
  la confirmación — reemplaza el cálculo anterior, que hubiera sido incorrecto
  ("hoy + 30 días" ≠ fin del ciclo siguiente).

Las fechas están **hardcodeadas** (`CICLO_ACTUAL_INICIO`, `FECHA_HOY`,
`FECHA_VENCIMIENTO`, `CICLO_NUEVO_INICIO`, `CICLO_NUEVO_FIN`), no calculadas con
aritmética de fechas — coherente con el resto del proto, que ya usaba fechas fijas
para todo. Si el dataset de ejemplo cambia, estas constantes hay que recalcularlas a mano.

### Pago con la cuenta Vencida: inicio del ciclo nuevo — A DEFINIR

Por pedido explícito del brief, **no se calculó** qué fecha tendría el ciclo nuevo si
se paga con la cuenta Vencida (o En revisión, que nace de un Vencido). En ese caso:

- El resumen de pago muestra el plan **sin** período ("Fecha de inicio del ciclo
  nuevo a definir" en vez de un rango de fechas).
- La nota de "días que pasan a la próxima factura" no se muestra (no aplica: ya no
  hay días parciales, todo el ciclo quedó congelado).
- La confirmación **no muestra** la fila "Próximo vencimiento" (se oculta con
  `doneProximoRow.hidden`).

Esto se controla con `PUEDE_CALCULAR_CICLO_NUEVO = !CONGELADO` en el script de
`pagar-suscripcion.html`. Cuando negocio defina la regla para cuentas Vencidas, hay
que: decidir la fecha de inicio (¿el día del pago? ¿el día siguiente? ¿se pierde el
tiempo vencido?), agregarla como constante, y sacar la condición `!CONGELADO` de los
tres puntos de arriba.

## Sección "Plan" en pagar-suscripcion.html

- **Activo o En revisión**: informativa, sin radio buttons (`#planInfo`). Muestra el
  plan que se está renovando — en este proto, siempre Plan Mensual (el dataset de
  ejemplo no contempla cuentas cuyo plan activo sea el semestral).
- **Por vencer o Vencido**: selector real (`#planPicker`) — 30 días o 180 días, con
  Plan Mensual preseleccionado por defecto (antes venía el Semestral preseleccionado
  incluso en Vencido; se cambió para que coincida con el ejemplo del brief). El plan
  elegido rige desde el ciclo siguiente (ver "Ciclos" arriba) — en Vencido, esa fecha
  queda a definir.
- La promo amarilla de 180 días (banner en `index.html`) se oculta solo en Activo;
  se muestra en Por vencer, Vencido y En revisión.

## Pago con tarjeta — nuevo en este proto

No existía antes (elegir "Tarjeta de crédito/débito" no mostraba nada). Se agregó
`#tarjeta` (`<!-- NUEVO -->`): un botón "Pagar Gs. X con tarjeta" con el monto ya
calculado, sin campos de número de tarjeta/vencimiento/CVV — simplificado a propósito
para el proto, ya que el brief solo pedía que el monto sea correcto. Al hacer click
simula el cobro y muestra la confirmación con destino `?estado=activo`.

## Pendientes / a definir

- **Inicio del ciclo nuevo al pagar con la cuenta Vencida**: no calculado (ver
  "Ciclos" arriba). Es el pendiente más importante de esta vuelta.
- **Estado de "Comprobante rechazado"**: no implementado (ver arriba).
- **Validación de Operaciones** (En revisión → Activo): no hay una pantalla de
  operador en este proto; se simula solo navegando manualmente a `?estado=activo`.
- El selector de estado y el querystring son un recurso de prototipo — en prod el
  estado saldría del backend, no de una URL.

### Ya resuelto (sacado de acá)

- ~~Falta el estado "Por vencer"~~ → implementado (tag amarillo + transiciones).
- ~~Solo se acepta transferencia~~ → se acepta transferencia y tarjeta, con la regla
  de cobro y las transiciones de cada una.
