# Proto de suscripción BIMS

Prototipo estático (HTML + CSS + JS vanilla, sin build ni dependencias) del flujo de
suscripción de BIMS. Dos pantallas:

- `index.html` — Resumen de suscripción ("suscripcion-resumen"). Muestra el estado de
  la cuenta, el detalle de consumos y adicionales, y lleva a pagar. En En revisión
  muestra el bloque del pago que se está validando y no deja iniciar otro pago.
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
| En revisión  | amarillo + punto que late (`bims-badge--dot`) | En el selector del proto y alcanzable por flujo. Lleva además la clase `sub-tag--revision` (pedida en el brief; es solo un hook, sin estilos propios — los colores salen de `bims-badge--warning`) |

### Transiciones implementadas

- **Activo → Por vencer**: 7 días antes del vencimiento (regla de negocio; el proto no
  calcula fechas, el estado se fuerza con el selector o el querystring).
- **Por vencer → Vencido**: vence sin pago.
- **Por vencer o Vencido + pago con tarjeta → Activo** (ciclo nuevo). Implementado de
  forma universal (cualquier estado + tarjeta → Activo), ya que pagar con tarjeta
  siempre deja la cuenta al día. El pago con tarjeta redirige a Bancard — ver
  "Pago con tarjeta → pasarela de Bancard" más abajo; esta transición solo ocurre si
  se simula la vuelta como "aprobado" (con "rechazado" o "cancelado" el tag no cambia).
- **Cualquier estado + comprobante de transferencia ENVIADO → En revisión**. El tag de
  `pagar-suscripcion.html` cambia recién al enviar (`setTag(ESTADO_DESTINO)` en
  `mostrarDone()`): elegir o cargar el archivo sin enviarlo no lo cambia. "Ver estado
  del pago" e "Ir al inicio" llevan siempre a `?estado=revision` (ver "En revisión"
  más abajo).
- **En revisión + validación de Operaciones → Activo** (no simulado: no hay un panel
  de Operaciones en este proto; se llegaría manualmente cambiando el querystring).

### Reglas todavía sin definir — NO implementadas

Por pedido explícito del brief, no hay UI ni lógica para: **rechazo del comprobante
por Administración**, **diferencias de monto** entre lo transferido y lo adeudado, y
**vencimiento de la prórroga** (qué pasa el 29-09-2026 a las 10:24 hs si el pago no
se validó). Hay un comentario en el `<script>` de ambas pantallas. No agregar nada de
esto hasta que negocio defina las reglas.

### Cómo se pasa el estado entre pantallas

Query param `?estado=activo|por-vencer|vencido|revision` (default: `activo` si
falta o es inválido). `index.html` arma el link de "Ir a pagar" con el estado actual;
`pagar-suscripcion.html` lo lee al cargar (`ESTADO_ORIGEN`, fijo durante toda la
sesión de esa pantalla) y decide a dónde vuelven "Ir al inicio" / "Ver estado del
pago" según lo que pasó (`ESTADO_DESTINO`).

Para una transferencia, además se pasan (`DESTINO_QUERY` en el script):

- `&monto=<Gs>` — el `TOTAL` pagado, para que el bloque de En revisión coincida con
  la confirmación.
- `&origen=activo|por-vencer|vencido` — el estado desde el que se pagó, para que el
  detalle de consumos de `index.html` coincida con ese monto (congelado si se pagó
  Vencido, "hasta hoy" si no).
- `&archivo=<nombre>` — el nombre del archivo enviado, que muestra "Ver comprobante".

Si faltan (por ejemplo, entrando a `?estado=revision` desde el selector), se usa el
ejemplo fijo: `Gs. 1.666.000` (el total de una cuenta Vencida con Plan Mensual),
origen `vencido` y `comprobante-transferencia-sudameris.pdf`. **Ya no se pasa
`&fecha=`**: la fecha de carga es un dato fijo del proto (ver "Datos de ejemplo").

`pagar-suscripcion.html?estado=revision` **redirige a `index.html?estado=revision`**
(`location.replace`): con un pago en revisión no se puede iniciar otro pago del mismo
período.

`pagar-suscripcion.html` además acepta `?paso=redireccion`,
`?bancard=ok|rechazado|cancelado` y `?plan=semestral` para entrar directo a cualquier
paso del pago con tarjeta/Bancard (se combinan con `?estado=`) — ver "Pago con
tarjeta" más abajo.

### Selector del proto

En `index.html`, arriba del contenido: `Activo | Por vencer | Vencido | En revisión`
(`.proto-state-selector`, `<!-- NUEVO -->`). Son links a `?estado=...` (sin `monto`/
`origen`/`archivo`, así que "En revisión" desde acá siempre muestra el ejemplo fijo), no un
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
- **En revisión** no tiene saldo a pagar: ya se pagó y se espera la validación. En
  `index.html`, `congelado` es Vencido **o** En revisión con `origen=vencido` (el
  default) — así el detalle de consumos suma lo mismo que el monto pagado.

### Datos de ejemplo del proto (fijos, no se recalculan por fecha real)

- Hoy: `15-09-2026` · Vence: `18-09-2026` · Plan Mensual: `Gs. 340.000`.
- Adicionales hasta hoy: `Gs. 1.187.400` · Adicionales hasta el vencimiento (congelado): `Gs. 1.326.000`.
- **Activo / Por vencer** → `340.000 + 1.187.400 = Gs. 1.527.400`.
- **Vencido** (plan mensual) → `340.000 + 1.326.000 = Gs. 1.666.000`.
- Plan Semestral (180 días, 20% off): `Gs. 1.632.000` (antes `Gs. 2.040.000`).
  Elegible en Por vencer y Vencido → en Vencido: `1.632.000 + 1.326.000 = Gs. 2.958.000`;
  en Por vencer: `1.632.000 + 1.187.400 = Gs. 2.819.400`.
- **Pago por transferencia en revisión**: fecha de carga `25-09-2026 10:24 hs`, medio
  "Transferencia", monto de ejemplo `Gs. 1.666.000`, prórroga hasta el
  `29-09-2026 a las 10:24 hs` (48 horas hábiles: viernes 25 → martes 29). Son las
  constantes `FECHA_CARGA` y `HORA_CARGA`, **repetidas en el script de las dos
  pantallas** (si cambian, hay que cambiarlas en ambas), y `PRORROGA_HASTA`, que solo
  usa `index.html`. La prórroga no se
  calcula con aritmética de fechas. La confirmación de transferencia muestra esa
  misma fecha fija (ya no la fecha real del navegador); la de tarjeta sigue mostrando
  la fecha real.

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

- **Tag** en el header (`#stateTag`, amarillo con punto, clase extra
  `sub-tag--revision`) y el 4to botón del selector del proto ("En revisión").
- **Bloque de pago en revisión** arriba del todo (`#avisoRevision`, `<!-- NUEVO -->`,
  sobre `.bims-alert-warning`): título "Pago en revisión", tres datos
  (`.revision-datos`: Monto pagado, Fecha de carga `25-09-2026 10:24 hs`, Medio de
  pago "Transferencia"), el texto "Tu cuenta sigue activa hasta el 29-09-2026 a las
  10:24 hs mientras validamos el pago.", el link "Ver comprobante" y el texto
  secundario "¿Te equivocaste de comprobante? Escribí a facturacion@bimsapp.com"
  (mailto).
- **"Ver comprobante"** abre `#docModal` (reutiliza `.bims-modal`): solo lectura, con
  el nombre del archivo y la fecha de carga, sin opción de reemplazar ni eliminar
  (el único botón es "Cerrar"). No hay almacenamiento real de archivos.
- **No se puede iniciar otro pago del mismo período**: se oculta la barra inferior
  con "Pagar" (`.paybar`) y la promo de 180 días (que lleva al simulador y de ahí a
  pagar). No queda ningún link visible a `pagar-suscripcion.html`, y esa pantalla
  redirige de vuelta si se entra con `?estado=revision`.
- **Primer panel**: deja de decir "A pagar hoy" y pasa a "Pago en revisión" con el
  monto pagado y "Transferencia cargada el 25-09-2026 10:24 hs." (es solo un cambio
  de label/valor del mismo `.kpi`, no un panel nuevo).
- **Monto**: el que llega por `?monto=`; si falta, el total de la tabla, que con el
  origen por defecto (`vencido`) da `Gs. 1.666.000` — el mismo total que muestra
  `pagar-suscripcion.html?estado=vencido`.
- **Queda igual que en los otros estados**: la tabla de consumos (la columna se sigue
  llamando "A pagar hoy") y el segundo panel. Si el pago incluyó el Plan Semestral,
  el monto pagado no coincide con el total de la tabla, que siempre lista el plan de
  30 días del ciclo actual.

Sigue sin implementarse la sección "Pagos recientes" (historial); quedó en el
historial de git.

## Sección "Plan" en pagar-suscripcion.html

- **Activo**: informativa, sin radio buttons (`#planInfo`). (En revisión ya no llega
  a esta pantalla: redirige al resumen.) Muestra el
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
4. **Vuelta de Bancard (simulada)**: controles de proto con borde punteado
   (`.proto-switch` dentro de `#bancardProtoLinks`) — "Bancard aprobado / rechazado /
   cancelado" — siempre visibles en la pantalla de transición. En prod, Bancard
   redirige de nuevo a BIMS con el resultado real; acá no hay backend que lo reciba.
   Todo pasa por `volverDeBancard(resultado)`:
   - **Aprobado** (`ok`) → `mostrarDone('tarjeta')`: la confirmación de siempre, tag →
     Activo, título "Pago aprobado".
   - **Rechazado** → vuelve a `#payFlow` (paso de método de pago) con alert-danger en
     `#bancardAviso`: "Bancard rechazó el pago. No se hizo ningún cobro. Probá con
     otra tarjeta o pagá por transferencia."
   - **Cancelado** → vuelve a `#payFlow` con alert-info: "Cancelaste el pago en
     Bancard. No se hizo ningún cobro."
   - En rechazo y cancelación **se conservan el plan, el resumen de pago y el método**
     (tarjeta sigue seleccionada, con su botón de pago listo para reintentar) y el tag
     **no cambia** — nunca se toca `ESTADO_DESTINO` en ese camino. El aviso se oculta
     al cambiar de método o al volver a ir a Bancard.
   - **Se sacó** la pantalla intermedia `#bancardError` ("No pudimos procesar el
     pago", con "Intentar de nuevo" / "Pagar por transferencia"): el rechazo ahora
     deja al cliente directamente en el paso de método de pago.
5. **Entrada directa para probar** (se combinan con `?estado=`): `?paso=redireccion`
   abre la pantalla de transición; `?bancard=ok|rechazado|cancelado` abre cada
   resultado con tarjeta ya seleccionada; `?plan=semestral` preselecciona el plan
   donde se puede elegir. Ej.:
   `pagar-suscripcion.html?estado=vencido&plan=semestral&bancard=rechazado`.
   `?retorno=aprobado|rechazado` (el nombre anterior) sigue funcionando como alias.

El monto es una sola fuente (`TOTAL`, calculada en `recomputeResumen()`): el botón de
`#tarjeta`, la pantalla de transición y la confirmación siempre muestran el mismo
número — verificado en Por vencer y Vencido, con Plan Mensual y Semestral.

## Comprobante de transferencia: validación y archivo sin enviar

En `pagar-suscripcion.html`, sección "Instrucciones de transferencia":

- **Formatos**: PDF, JPG y PNG en cualquier ancho de pantalla (antes JPG solo en
  mobile). Dropzone: "Subí tu comprobante en PDF, JPG o PNG".
- **Errores**, en `#uploadError` (alert-danger, `role="alert"`), asociados al control
  con `aria-describedby` + `aria-invalid` en el `<input type="file">`:
  - Formato: "Subí el comprobante en PDF, JPG o PNG."
  - Tamaño: "El archivo pesa X MB. Subí uno de hasta 5MB." (sin cambios)
  - Archivo dañado: "No pudimos abrir este archivo. Intentá de nuevo."
  Se validan en ese orden. Ninguno toca el plan, el método ni el total.
- **Archivo dañado** (`archivoDanado()`): un archivo real se considera dañado si está
  vacío o si sus primeros bytes no corresponden a su extensión (`%PDF`, firma PNG,
  firma JPG). Es un criterio de prototipo, no una regla de negocio. Los archivos
  simulados no tienen contenido: usan la marca `corrupt`.
- **Links de proto**: "simular carga de un PDF", "simular archivo de más de 5MB" y
  "simular archivo dañado" (`#demoCorrupt`).
- **Archivo elegido sin enviar**: cuando termina la carga aparece debajo del archivo
  "Todavía no enviaste el comprobante." (`#sinEnviar`, `.bims-upload__pending`,
  `role="status"`) y se habilita el botón verde "Enviar comprobante" como acción
  principal. El tag no cambia hasta que se envía. El aviso se oculta al quitar el
  archivo o al enviar.

## Factura: solo una frase en la confirmación (sin estado aparte)

(Se armaron y se sacaron a pedido: "Sin pagos pendientes" en `index.html` — nunca
llegó a un commit, no está en el historial de git — y "Cambió el total" y "Factura
pendiente" en `pagar-suscripcion.html`, que sí quedaron en el commit `3972926` por
si hace falta retomarlos.)

No hay un estado de "factura pendiente" (existió con un alert y un control de proto, y
se sacó a pedido). La factura se menciona solo en la bajada de la confirmación
(`#doneSub`):

- **Tarjeta, pago aprobado**: "Tu suscripción ya está activa. Te vamos a enviar la
  factura por mail." (reemplaza a "El comprobante de pago llega por correo.")
- **Transferencia, En revisión**: no menciona la factura. La bajada es "**Pago en
  revisión**: lo validaremos en un plazo de hasta 48 horas hábiles. Mientras tanto,
  podés seguir usando BIMS con normalidad." (a pedido, reemplazó al texto con la
  fecha de la prórroga y la frase de la factura; la fecha `29-09-2026 a las 10:24 hs`
  se muestra solo en el bloque de En revisión de `index.html`).

## Pendientes / a definir

- **Inicio del ciclo nuevo al pagar con la cuenta Vencida**: no calculado (ver
  "Ciclos" arriba).
- **Rechazo de Administración, diferencias de monto y vencimiento de la prórroga**:
  no implementados, reglas sin definir (ver arriba).
- **Validación de Operaciones** (En revisión → Activo): no hay una pantalla de
  operador en este proto; se simula solo navegando manualmente a `?estado=activo`.
- **"Ver comprobante" es una vista de ejemplo** (`#docModal`), no hay almacenamiento
  de archivos real en ningún punto del proto (tampoco al "subir" un comprobante en
  `pagar-suscripcion.html`).
- **Historial de pagos**: no implementado — se armó una versión ("Pagos recientes")
  y se sacó a pedido para dejar el alcance de "En revisión" acotado solo al aviso y
  al tag. Si se retoma, el punto a resolver es el mismo que ya se documentó: cómo
  reflejar el monto/fecha real de cada pago en vez de datos de ejemplo fijos.
- **Botón "Pagar Gs. X con Bancard"**: se ve sin espacios ("PAGARGS. 2.819.400CON
  BANCARD") — bug visual previo, sin corregir.
- El selector de estado y el querystring son un recurso de prototipo — en prod el
  estado saldría del backend, no de una URL.

### Ya resuelto (sacado de acá)

- ~~Falta el estado "Por vencer"~~ → implementado (tag amarillo + transiciones).
- ~~Solo se acepta transferencia~~ → se acepta transferencia y tarjeta, con la regla
  de cobro y las transiciones de cada una.
- ~~"En revisión" solo se puede alcanzar por flujo, no se puede ver el pago hecho~~ →
  ahora está en el selector del proto y tiene su bloque en `index.html`, con el
  monto real pasado desde la confirmación cuando viene de un pago real.
