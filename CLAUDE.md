# Proto de suscripción BIMS

Prototipo estático (HTML + CSS + JS vanilla, sin build ni dependencias) del flujo de
suscripción de BIMS. Dos pantallas:

- `index.html` — Resumen de suscripción ("suscripcion-resumen"). Muestra el estado de
  la cuenta, el detalle de consumos y adicionales, y lleva a pagar. En En revisión
  suma un aviso de que ya hay un pago en validación.
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
| En revisión  | amarillo + punto que late (`bims-badge--dot`) | En el selector del proto y alcanzable por flujo |

### Transiciones implementadas

- **Activo → Por vencer**: 7 días antes del vencimiento (regla de negocio; el proto no
  calcula fechas, el estado se fuerza con el selector o el querystring).
- **Por vencer → Vencido**: vence sin pago.
- **Por vencer o Vencido + pago con tarjeta → Activo** (ciclo nuevo). Implementado de
  forma universal (cualquier estado + tarjeta → Activo), ya que pagar con tarjeta
  siempre deja la cuenta al día. El pago con tarjeta redirige a Bancard — ver
  "Pago con tarjeta → pasarela de Bancard" más abajo; esta transición solo ocurre si
  se simula la vuelta como "aprobado" (si se simula "rechazado", el tag no cambia).
- **Cualquier estado + comprobante de transferencia → En revisión** ("Ver estado del
  pago" e "Ir al inicio" llevan siempre a `?estado=revision`, con el monto y la fecha
  reales del pago — ver "En revisión" más abajo). Esto **cambió**: antes, si el origen
  era Activo o Por vencer, el tag se quedaba igual y solo la confirmación aclaraba
  "Pago en revisión"; ahora ese caso también navega a la vista de En revisión, para
  que el usuario vea el aviso y no pague de nuevo.
- **En revisión + validación de Operaciones → Activo** (no simulado: no hay un panel
  de Operaciones en este proto; se llegaría manualmente cambiando el querystring).

### Comprobante rechazado — NO implementado

Buscar `TODO` / el comentario `Comprobante rechazado: NO implementado` en el `<script>`
de ambas pantallas. Estado a definir con negocio (¿vuelve a Vencido? ¿un estado nuevo
"Rechazado"? ¿reintento directo?). No hay UI ni lógica para este caso.

### Cómo se pasa el estado entre pantallas

Query param `?estado=activo|por-vencer|vencido|revision` (default: `activo` si
falta o es inválido). `index.html` arma el link de "Ir a pagar" con el estado actual;
`pagar-suscripcion.html` lo lee al cargar (`ESTADO_ORIGEN`, fijo durante toda la
sesión de esa pantalla) y decide a dónde vuelven "Ir al inicio" / "Ver estado del
pago" según lo que pasó (`ESTADO_DESTINO`).

Para una transferencia, además se pasan `&monto=<Gs>&fecha=<DD-MM-AAAA>` (el `TOTAL`
pagado y la fecha real, `DESTINO_QUERY` en el script) — así `index.html?estado=revision`
muestra el monto/fecha que realmente se pagó, sea cual sea el estado de origen, en vez
de mostrar siempre el mismo ejemplo fijo. Si esos parámetros faltan (por ejemplo, si se
entra a `?estado=revision` directo desde el selector), se usan valores de ejemplo:
`Gs. 1.527.400` y `25-09-2026`.

`pagar-suscripcion.html` además acepta `?paso=redireccion` y
`?retorno=aprobado|rechazado` para entrar directo a cualquier paso del pago con
tarjeta/Bancard (se combinan con `?estado=`) — ver "Pago con tarjeta" más abajo.

### Selector del proto

En `index.html`, arriba del contenido: `Activo | Por vencer | Vencido | En revisión`
(`.proto-state-selector`, `<!-- NUEVO -->`). Son links a `?estado=...` (sin `monto`/
`fecha`, así que "En revisión" desde acá siempre muestra el ejemplo fijo), no un
control interactivo sin recarga — cada click navega. Vive solo en `index.html`;
`pagar-suscripcion.html` no lo repite, solo lee el estado que llega por URL.

## Regla de cobro (reemplaza la anterior "el plan queda en Gs. 0 hoy")

- **Activo o Por vencer**: se paga **plan base + adicionales hasta la fecha de pago**
  ("hasta hoy").
- **Vencido**: se paga **plan base + adicionales, pero estos quedan congelados en el
  valor que tenían el día del vencimiento** (dejan de crecer aunque pase el tiempo).
- El monto de "adicionales" en Vencido es el mismo que ya se mostraba como "Estimado
  al cierre" en Activo/Por vencer — no es casualidad: es el valor que el ciclo
  alcanza al llegar a la fecha de vencimiento.
- **En revisión NO es un estado "congelado"** en `index.html`: ya se pagó (el monto
  y la fecha que llegan por querystring, o el ejemplo `Gs. 1.527.400` si no hay
  ninguno) y se está esperando la validación — no hay un saldo que siga creciendo o
  se haya frenado. `congelado` en el script de `index.html` significa solo Vencido.

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
  congelados (Vencido).
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
se paga con la cuenta Vencida. En ese caso (en `pagar-suscripcion.html`, antes de
llegar a `index.html`):

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

## Estado "En revisión" en suscripcion-resumen.html (`?estado=revision`)

Alcance reducido a pedido: solo el aviso y el estado en sí. El resto de la pantalla
(panel "A pagar hoy", barra inferior con el botón "Pagar", promo, tabla de consumos)
**se comporta igual que en cualquier otro estado no congelado** — no hay ninguna
rama especial para "revision" ahí. Lo único que cambia:

- **Tag** en el header (`#stateTag`, amarillo con punto) y el 4to botón del selector
  del proto ("En revisión").
- **Aviso arriba del todo** (`#avisoRevision`, `<!-- NUEVO -->`): alert-warning de
  Bootstrap 3 (fondo `#fcf8e3`, texto `#8a6d3b` — no estaba en el kit, se agregó como
  `.bims-alert-warning`) con "Estamos validando tu pago de `Gs. X`", la fecha de
  envío, "Ver comprobante" (abre un modal de solo lectura, `#docModal`, reutiliza
  `.bims-modal` — no hay almacenamiento de archivos real en el proto) y la nota
  chica de "¿enviaste un comprobante equivocado? escribinos a
  facturacion@bimsapp.com".
- **Monto y fecha reales, no siempre el mismo ejemplo**: si se llega desde un pago
  real, `pagar-suscripcion.html` pasa `&monto=&fecha=` (ver "Cómo se pasa el estado
  entre pantallas" arriba) y el aviso los usa; si faltan (p. ej. entrando por el
  selector del proto), usa el ejemplo fijo `Gs. 1.527.400` / `25-09-2026`. Verificado
  con Puppeteer que una transferencia desde Activo y una desde Vencido (monto
  distinto, `Gs. 1.666.000`) navegan a `?estado=revision` y el aviso muestra el
  monto/fecha reales de cada una, coincidiendo con lo que mostró la confirmación.

**Se sacó** (existió brevemente, pedido explícito de simplificar): el panel
"Pago en revisión" reemplazando "A pagar hoy", la barra inferior sin botón, la nota
de adicionales en la tabla de consumos, y la sección "Pagos recientes" con el
historial de pagos — quedaron en el historial de git si hace falta retomarlos.

## Sección "Plan" en pagar-suscripcion.html

- **Activo o En revisión**: informativa, sin radio buttons (`#planInfo`). Muestra el
  plan que se está renovando — en este proto, siempre Plan Mensual (el dataset de
  ejemplo no contempla cuentas cuyo plan activo sea el semestral).
- **Por vencer o Vencido**: selector real (`#planPicker`) — 30 días o 180 días, con
  Plan Mensual preseleccionado por defecto (antes venía el Semestral preseleccionado
  incluso en Vencido; se cambió para que coincida con el ejemplo del brief). El plan
  elegido rige desde el ciclo siguiente (ver "Ciclos" arriba) — en Vencido, esa fecha
  queda a definir.
- La promo amarilla de 180 días (banner en `index.html`) se oculta en Activo y en En
  revisión (no tiene sentido ofrecer upgrade con un pago pendiente de validar); se
  muestra en Por vencer y Vencido.

## Pago con tarjeta → pasarela de Bancard

BIMS **no toma los datos de la tarjeta** (no hay campos de número/vencimiento/CVV/
titular): al elegir "Tarjeta de crédito/débito" se muestra el monto y un botón verde
que redirige a Bancard. Pantallas nuevas, todas marcadas `<!-- NUEVO -->`:

1. **`#tarjeta`** (dentro de `#payFlow`): monto + nota "Vas a completar el pago en el
   sitio seguro de Bancard" + botón "Pagar Gs. X con Bancard".
2. **`#bancardRedirect`** — pantalla de transición, fondo con gradiente navy
   (`var(--bims-navy)` → `var(--bims-navy-deep)`, tokens del kit, `.bancard-redirect-wrap`).
   Tarjeta centrada con spinner (no un check verde: todavía no se cobró), "Te estamos
   llevando a Bancard", el monto, una barra de progreso animada 3s
   (`.bims-progress`/`#bancardProgressBar`) y "Conexión segura (SSL)" en gris. Debajo,
   el aviso amarillo del kit (`.promo.promo--b`) con el mail de facturación. El topbar
   y el subbar de BIMS se mantienen.
3. **Sin redirección automática**: la pestaña del proto nunca navega sola a otro
   sitio. El botón "Ir a Bancard" (`#bancardManualLink`, un link real con
   `target="_blank" rel="noopener"`) y los links de simulación se ven **apenas
   aparece la pantalla** (no esperan a la barra de progreso, que es solo de
   ambientación). Al hacer click en "Ir a Bancard" se abre
   `https://vpos.infonet.com.py/payment/single_buy?process_id=L4nUbqFiVM8zBgB9dQqS`
   en una pestaña nueva. **El `process_id` es de ejemplo, fijo** — en prod lo genera
   el backend en cada intento de pago y vence a los pocos minutos (comentario en el
   script). (Antes esto se intentaba abrir solo con `window.open()` automático desde
   un `setTimeout`; se sacó porque un navegador real lo bloquea casi siempre al no
   venir de un click directo, y de paso dejaba sin ver los links de simulación hasta
   que ese intento terminaba.)
4. **Vuelta de Bancard (simulada)**: los links de proto "Simular pago aprobado" /
   "Simular pago rechazado" (`#bancardProtoLinks`) están siempre visibles en la
   pantalla de transición. En prod, Bancard redirige de nuevo a BIMS con el resultado
   real; acá no hay backend que lo reciba, así que se simula a mano.
   - **Aprobado** → `mostrarDone('tarjeta')`: la confirmación de siempre, tag →
     Activo, título "Pago aprobado".
   - **Rechazado** → `#bancardError`: "No pudimos procesar el pago" / "Bancard
     rechazó la operación. No se hizo ningún cobro.", botones "Intentar de nuevo"
     (vuelve a `#payFlow` con tarjeta ya seleccionada) y "Pagar por transferencia"
     (cambia el método y vuelve a `#payFlow`). El tag **no cambia** — nunca se toca
     `ESTADO_DESTINO` en este camino.
5. **Entrada directa para probar** (se combinan con `?estado=`): `?paso=redireccion`
   abre directo la pantalla de transición; `?retorno=aprobado` o `?retorno=rechazado`
   abren directo cada resultado. Ej.:
   `pagar-suscripcion.html?estado=vencido&paso=redireccion`.

El monto es una sola fuente (`TOTAL`, calculada en `recomputeResumen()`): el botón de
`#tarjeta`, la pantalla de transición y la confirmación siempre muestran el mismo
número — verificado en Por vencer y Vencido, con Plan Mensual y Semestral.

## Pendientes / a definir

- **Inicio del ciclo nuevo al pagar con la cuenta Vencida**: no calculado (ver
  "Ciclos" arriba).
- **Estado de "Comprobante rechazado"**: no implementado (ver arriba).
- **Validación de Operaciones** (En revisión → Activo): no hay una pantalla de
  operador en este proto; se simula solo navegando manualmente a `?estado=activo`.
- **"Ver comprobante" es una vista de ejemplo** (`#docModal`), no hay almacenamiento
  de archivos real en ningún punto del proto (tampoco al "subir" un comprobante en
  `pagar-suscripcion.html`).
- **Historial de pagos**: no implementado — se armó una versión ("Pagos recientes")
  y se sacó a pedido para dejar el alcance de "En revisión" acotado solo al aviso y
  al tag. Si se retoma, el punto a resolver es el mismo que ya se documentó: cómo
  reflejar el monto/fecha real de cada pago en vez de datos de ejemplo fijos.
- El selector de estado y el querystring son un recurso de prototipo — en prod el
  estado saldría del backend, no de una URL.

### Ya resuelto (sacado de acá)

- ~~Falta el estado "Por vencer"~~ → implementado (tag amarillo + transiciones).
- ~~Solo se acepta transferencia~~ → se acepta transferencia y tarjeta, con la regla
  de cobro y las transiciones de cada una.
- ~~"En revisión" solo se puede alcanzar por flujo, no se puede ver el pago hecho~~ →
  ahora está en el selector del proto y tiene su aviso en `index.html`, con el
  monto/fecha reales pasados desde la confirmación cuando vienen de un pago real.
