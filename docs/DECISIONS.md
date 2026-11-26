# Decisiones y trade-offs

> Estas decisiones se tomaron **antes de implementar**, a partir de leer la consigna, el README y el código del `mock-api`, y de probar la API. Al final del desarrollo se agrega la sección "Qué cambió al implementar", con lo que se confirmó, se ajustó o se descartó. Al final de cada ambigüedad, "Dónde se implementa" indica el archivo previsto en el código; si cambia al implementar, se actualiza. La arquitectura completa está en [`ARCHITECTURE.md`](./ARCHITECTURE.md).

---

## Resumen

El producto es una **wallet** o **billetera personal**. El criterio que guía todas las decisiones es la pregunta de la consigna: *¿se entiende lo que está pasando con tu dinero en todo momento?*

Las tres ambigüedades que plantea la consigna orientan el resto del documento:

| Ambigüedad | Decisión en una línea |
|---|---|
| [1. Multimoneda](#1-multimoneda) | Se **bloquea** toda transferencia entre monedas distintas; hacia otro usuario, según la moneda que declara. Advertir permitiría corromper los datos |
| [2. Estado `pending` → `completed`](#2-estado-asincrónico-pending--completed) | **Polling** hasta conocer el estado real; si no se sabe, se dice. Nada de updates optimistas ni de "Reintentar" sobre algo que pudo haberse ejecutado |
| [3. Fondos insuficientes](#3-fondos-insuficientes-422) | **Validar antes y manejar el 422.** La validación es contra un **saldo disponible** propio, porque la API permite un doble gasto (verificado) |

De resolverlas salen los [principios transversales](#principios-transversales) que se aplican al resto de las decisiones.

---

## Las tres ambigüedades de la consigna

### 1. Multimoneda

**Qué dice la consigna:** *"Si transferís entre cuentas de monedas distintas, la API no convierte nada — el monto queda registrado tal cual en cada lado. ¿Lo permitís, lo advertís, lo bloqueás, mostrás una conversión aproximada?"*

**Qué encontré**, leyendo el código del `POST /transfers` ([`index.ts:541-542`](../mock-api/index.ts#L541-L542)):

- La API registra cada transferencia como dos movimientos, uno por cuenta: sus dos **patas** (ver [Glosario](#glosario)). Cada pata se registra con la moneda de su cuenta y el **mismo `amount`**. Si salen 5.000 ARS, entran 5.000 USD: se crea valor de la nada y se rompe el invariante básico de una transferencia (lo que sale de un lado es lo que entra en el otro).
- No hay forma de conocer la moneda de una cuenta ajena antes de transferir. `GET /accounts/:id` de otro usuario devuelve 404 y no existe un endpoint de búsqueda.
- La respuesta del `POST` sí trae la pata destino, con su moneda.

**Opciones evaluadas:**

| Opción | Evaluación |
|---|---|
| Permitir | ❌ Corrompe los datos de los dos usuarios |
| Advertir | ❌ Es permitir la corrupción con el consentimiento del usuario. El problema no es de UX, es de integridad |
| Convertir en el cliente | ❌ No es posible por **contrato de la API**: `POST /transfers` recibe un solo `amount`, no hay forma de expresar "debitá X ARS y acreditá Y USD". Una conversión solo informativa no evita la corrupción |
| Bloquear | ✅ La única opción que protege los datos con esta API |

**Decisión:**

| Caso | Comportamiento |
|---|---|
| Entre cuentas propias, misma moneda | Se permite |
| Entre cuentas propias, distinta moneda | **Se bloquea** antes de enviar, explicándole al usuario por qué |
| Hacia otro usuario (P2P) | El usuario **declara la moneda** de la cuenta destino. Si no coincide con la de origen, se bloquea |
| P2P, después del envío | Si la moneda real de la pata destino (de la respuesta del `POST`) no coincide, se muestra un **aviso de incidente**. No se puede revertir: la API no tiene endpoint para eso |

**Por qué no bloqueo todas las P2P:** la consigna exige transferir "hacia la cuenta de otro usuario". Bloquearlas por no poder verificar la moneda incumpliría un requisito mínimo.

**Riesgo residual aceptado:** si el usuario declara mal la moneda en una P2P, la transferencia se registra mal. Se detecta y se informa, pero no se previene. La solución de fondo es del backend (ver [Propuestas para el backend](#propuestas-para-el-backend)).

**Preparado para el cambio:** hay un port `AccountDirectory` que hoy siempre responde "moneda desconocida". Si el backend agrega una búsqueda de destinatario, solo cambia el adapter y el dominio pasa a usar la moneda verificada.

**Dónde se implementa:** `domain/transfer-policy.ts` (política de moneda) y `domain/money.ts` (`CurrencyMismatchError`: el propio tipo impide sumar monedas distintas).

---

### 2. Estado asincrónico `pending` → `completed`

**Qué dice la consigna:** *"Una transferencia nace en estado `pending` y se liquida sola a `completed` un segundo y medio después. ¿Cómo te enterás de ese cambio en la UI?"*

**Qué encontré:**

- La liquidación es un `setTimeout` en memoria ([`index.ts:546`](../mock-api/index.ts#L546)). Si la API se reinicia en ese 1,5 s, la transferencia queda `pending` para siempre.
- El estado `failed` existe en el schema, pero la API nunca lo emite.
- Los ids se generan con `Date.now()` y no hay idempotencia: si un `POST` no responde, no hay forma de saber si se creó.
- `pending` aparece en tres lugares: justo después de crear la transferencia, en el listado de movimientos (incluidas las que **recibo** de otros usuarios) y en el saldo.

**Opciones evaluadas:** mostrar solo "procesando" (deja estado viejo en pantalla), refrescar al volver a la pantalla, polling, push (la API no lo ofrece) y optimista (mostrar `completed` antes de tiempo).

**Decisión:**

- **Polling** de `GET /transfers/:id` cada ~500 ms, hasta ~15 s. El intervalo no depende del 1,5 s de la API: la política es genérica.
- **Refresco del listado** mientras haya movimientos `pending` visibles, y al volver a la pestaña.
- **Si se agota el tiempo:** *"Tu transferencia fue aceptada y se está procesando"*. No es un error, porque fue aceptada. **Nunca se ofrece "Reintentar"**: reintentar algo que se ejecutó duplicaría la transferencia.
- **Si el `POST` no responde** (error de red o timeout): *"No pudimos confirmar si la transferencia se realizó. Revisá tus movimientos antes de volver a intentar."* Es un tercer resultado, **desconocido**, distinto de aceptada y de rechazada.
- **Un error de red durante el polling** no significa que la transferencia falló: se sigue consultando hasta el límite.
- **`failed` está contemplado** aunque hoy no ocurra: *"La transferencia fue rechazada. No se debitó dinero de tu cuenta."*

**Por qué no optimista:** mostrar `completed` antes de que lo sea es mentirle al usuario sobre su dinero. Lo que sí cambia de inmediato es el saldo disponible, y no es optimista: la transferencia pendiente existe de verdad en la API (ver ambigüedad 3).

**Dónde se implementa:** `domain/polling-policy.ts` es una función pura, `nextPollStep(estado, intento, tiempo transcurrido) → esperar | terminado | agotado`, que se testea sin timers reales. El timer real está detrás de un port `Scheduler`, que en los tests se reemplaza por uno falso.

---

### 3. Fondos insuficientes (422)

**Qué dice la consigna:** *"Si una transferencia supera el **saldo disponible**, la API devuelve un error 422. ¿Validás antes de enviar o dejás que falle y mostrás el error?"*

**Qué encontré:** acá estuvo el hallazgo más importante.

- La API calcula el saldo sumando solo movimientos `completed` ([`index.ts:194`](../mock-api/index.ts#L194)), y el 422 compara contra ese saldo ([`index.ts:522-525`](../mock-api/index.ts#L522-L525)). **No descuenta las transferencias `pending`.**
- **Lo verifiqué con Postman** (Collection Runner, 2 iteraciones, sin delay): partiendo de $855.000, dos transferencias de $500.000 devolvieron **201 las dos**, y la cuenta quedó en **−$145.000**.
- No es una condición de carrera: Node procesa cada request de punta a punta y SQLite es sincrónico. Es la ventana de `pending`, sin reserva de fondos. Pasa igual con dos envíos secuenciales dentro de ese 1,5 s.
- Por eso la consigna habla de "saldo disponible", pero la API usa el **saldo contable**. Tomo esa diferencia como una ambigüedad más.
- Además, `POST /transactions` **no valida fondos**: un gasto mayor al saldo se acepta y deja la cuenta en negativo.

**Decisión: validar antes Y manejar el 422.** Ninguna de las dos alcanza sola. El cliente no conoce lo que pasa en otra pestaña u otro dispositivo, y el backend deja pasar el doble gasto.

1. **Saldo disponible** = saldo contable − transferencias **salientes** `pending`.
   - Las entrantes pendientes no suman, porque es plata que todavía no llegó.
   - Las `failed` no restan, porque no debitan.
2. **Validación en línea** contra el disponible, con el disponible visible junto al campo. Un monto **igual** al disponible se permite, porque es la misma regla que aplica la API.
3. **Si hay pendientes, el mensaje explica la diferencia:** *"Fondos insuficientes. Disponible: $355.000 (tenés $500.000 en transferencias en proceso)."*
4. **Si igual llega un 422:**
   - se refrescan saldos y pendientes;
   - se **conservan los datos del formulario**;
   - se muestra el disponible actualizado;
   - se permite reintentar, porque un 422 garantiza que la transferencia no se creó.
5. **Los gastos también se bloquean** si superan el disponible. En una billetera no tiene sentido gastar plata que no tenés, y la regla de fondos es una sola para transferencias y gastos.
6. **Si el saldo queda negativo por algo ocurrido fuera de la app,** se muestra en rojo con el texto "Saldo negativo" y se bloquea todo débito.
7. **Orden de las consultas:** primero las pendientes y después los saldos. Si una transferencia se liquida entre los dos requests, con este orden el disponible queda **subestimado**, que es el error conservador. Con el orden inverso quedaría sobreestimado. Ante la duda, el error tiene que caer del lado del usuario.

**Dónde se implementa:** `domain/balance.ts` (`availableBalance`, función pura) y `domain/transfer-policy.ts` (regla de fondos única). El 422 lo traduce el adapter HTTP a un error de dominio `insufficient-funds`, sin depender del texto que manda la API.

---

## Principios transversales

Son las conclusiones de resolver las tres ambigüedades. Se aplican a todas las decisiones que siguen:

1. **El saldo de la API no es lo que el usuario puede usar.** La API permite un doble gasto, y lo verifiqué: el chequeo de fondos ignora las transferencias `pending`. Si se ejecutan dos transferencias seguidas de $500.000 a partir de una cuenta con $855.000, pasan las dos y la cuenta queda con un saldo negativo de −$145.000. Por eso la app calcula un **saldo disponible** (saldo − transferencias salientes en proceso) y valida contra ese número, no contra el que devuelve la API. *(Ambigüedad 3)*
2. **Ante la duda, el error tiene que caer del lado del usuario.** Es preferible subestimar el disponible que sobreestimarlo, y decir "no sabemos" que suponer un resultado. *(Ambigüedades 2 y 3)*
3. **El estado de una transferencia nunca se inventa.** No hay updates optimistas sobre el estado: se hace polling hasta saber, y si no se sabe, se dice *"no sabemos"*. Nunca se ofrece "Reintentar" sobre algo que pudo haberse ejecutado. *(Ambigüedad 2)*
4. **La integridad de los datos está por encima de la comodidad.** Se bloquea lo que corrompe datos (monedas distintas) y se advierte lo que puede ser legítimo (un posible duplicado). *(Ambigüedad 1)*
5. **El dinero se maneja en enteros** (centavos), nunca en `number` decimal. Es la base para comparar saldos y montos sin errores de redondeo. *(Ambigüedades 1 y 3)*
6. **Las tres ambigüedades viven en el dominio**, en TypeScript puro y con tests, no repartidas por los componentes. Ver [`ARCHITECTURE.md`](./ARCHITECTURE.md).

---

## Otros trade-offs

### Dinero: value object propio en unidades menores

- `Money { minor: entero, currency }`. Nunca `number` decimal, porque `0.1 + 0.2 !== 0.3`.
- La API guarda `amount` como `REAL` y acepta montos con más de 2 decimales. La conversión se hace **solo en el borde**, en el adapter HTTP.
- **Descartado:** usar una librería (`dinero.js`, `big.js`). Es una dependencia para cinco operaciones (sumar, restar, comparar, negativo, cero). Tampoco `bigint`, que no serializa a JSON.
- **El input del usuario se parsea en formato es-AR estricto:** `"1.234,56"` → $1.234,56, y `"1.234"` es mil doscientos treinta y cuatro. **Nunca se redondea en silencio:** `"10,005"` es un error, *"Máximo 2 decimales"*. El campo es `type="text" inputmode="decimal"`, no `type="number"` (por la rueda del mouse, `1e5` y el separador según el navegador).
- **La salida se formatea con `Intl` es-AR:** `$ 1.234,56` y `US$ 440,00`. Las monedas se distinguen solas.

### Doble envío

La API no es idempotente y genera los ids con `Date.now()`. Un doble click crea dos transferencias, y con el doble gasto de arriba, las dos pasan.

- **Defensa principal:** *single-flight* en el **caso de uso**. Si hay un envío en curso, el segundo se ignora. Es TS puro, testeable e independiente del template.
- **Botón deshabilitado** mientras se envía: necesario, pero no suficiente.
- **Confirmación** antes de transferir, con origen, destino, monto, moneda y **disponible después de la operación**. Ingresos y gastos no la llevan: el riesgo es menor y alcanza con las dos defensas anteriores.
- **Después del éxito,** el formulario no queda cargado, así que no se puede reenviar por accidente.
- **Aviso de posible duplicado** si hay una transferencia igual (origen, destino y monto) en los últimos ~2 minutos. Acá **advertir sí es correcto**, a diferencia de la multimoneda: un duplicado puede ser legítimo y no corrompe datos.
- **Nunca se reintenta un `POST` automáticamente**, ni en el adapter ni en un interceptor.
- **No simulo un `Idempotency-Key`:** mandar un header que la API ignora daría una falsa sensación de protección.

### Transferencias entre cuentas propias en el listado

Una transferencia entre cuentas propias genera dos filas. En "todas las cuentas" se ven las dos.

- **Decisión:** dos filas vinculadas, con la etiqueta "Entre tus cuentas" y el nombre de la otra cuenta.
- **Descartado:** agruparlas en una sola fila. Rompe la paginación, porque la API pagina por filas y las dos patas pueden caer en páginas distintas: el orden de la API no tiene desempate.
- Las transferencias internas **no cuentan** como ingreso ni como gasto.

### Qué se puede editar

- Solo **categoría y descripción**, y solo de **ingresos y gastos**. El monto es inmutable y las transferencias no se editan.
- La regla vive en el dominio (`canEdit`), con tests. La UI no ofrece editar una transferencia, en lugar de dejar que falle con un 400.

### Errores de la API

La API devuelve errores con **formatos inconsistentes**:

- `400` con `{ errors: [{ field, message }] }` o con `{ error }`, según el caso;
- `404` y `422` con `{ error }`;
- `500` como HTML.

**Un único `mapApiError`**, en el adapter, los traduce a errores de dominio. El resto de la app nunca ve un formato de la API.

| Error | Qué ve el usuario |
|---|---|
| 400 de validación | El mensaje en el campo correspondiente (según `field`). Si no corresponde a un campo, un mensaje general |
| 401 | Se cierra la sesión y se redirige al login, guardando a dónde iba |
| 404 de cuenta destino | *"La cuenta destino no existe"*, en el campo |
| 422 | Fondos insuficientes (ver ambigüedad 3) |
| Red / 500 | *"Sin conexión"* o *"Ocurrió un error"*. **"Reintentar" solo en lecturas**, nunca en un envío |

Los mensajes son propios, en español. El texto de la API no se muestra al usuario.

### Estado: Signals, con la lógica fuera del framework

- Los **casos de uso son TypeScript puro**, y los ports devuelven `Promise`. El núcleo no depende de Angular ni de RxJS.
- **Signals** solo en *stores* finos de cada feature. Para cambiar de librería de estado, o de framework, hay que reescribir los stores y los componentes, no la lógica.
- **RxJS** solo en infraestructura: HTTP y el timer del polling.
- **NgRx descartado:** es excesivo para el tamaño de la app.

### Sesión

- El token va en `sessionStorage`, detrás de un port, así que cambiarlo a cookie o a memoria solo toca el adapter.
- Al cargar la app, se valida con `GET /auth/me`.
- **Limitación:** el token de la API es falso y no expira.

### Filtros y paginación

- Los **filtros viven en la URL**: se pueden compartir, sobreviven al refresh y "atrás" funciona.
- La búsqueda tiene debounce. Cambiar un filtro vuelve a la página 1.
- Hay dos estados vacíos distintos: *"Todavía no tenés movimientos"* y *"No hay resultados para estos filtros"*.

### Categorías

- **Lista fija en el dominio**, con etiquetas en español, en lugar de pedirla a `/categories`: la API la define fija en su código.
- La categoría `transfer` no se ofrece al cargar un ingreso o un gasto.

### Herramientas

- **TypeScript estricto**, con Angular 20 standalone.
- **Tests con Karma + Jasmine.** Evalué Vitest, pero en Angular 20.3 su builder está marcado como `[EXPERIMENTAL]`, y no quise apoyar un entregable en eso. Los tests del dominio son TS puro, así que migrar después sería casi solo configuración.
- **Angular Material**, porque es lo más económico: las decisiones de arriba necesitan dialog, snackbar, paginador y errores por campo accesibles, y Material los trae resueltos. Se usa **solo en la capa de UI**: el dominio no lo conoce.
- **Node 22 LTS** para el `mock-api`: `better-sqlite3` no compila en Node 24 sobre Windows. Lo resolví con la versión de Node, sin tocar la API.

---

## Glosario

La UI usa siempre estos términos, y solo estos:

| Término | Significado |
|---|---|
| **Disponible** | Lo que podés usar: saldo − transferencias salientes en proceso. Es el número principal |
| **Saldo** | Lo que devuelve la API: suma de los movimientos completados |
| **En proceso** | Transferencias salientes `pending` |
| **Por acreditar** | Transferencias entrantes `pending`: todavía no suman |

Términos que se usan en este documento (no en la UI):

| Término | Significado |
|---|---|
| **Pata** (*leg*) | Cada uno de los dos movimientos con que la API registra una transferencia: la **pata de salida** (débito en la cuenta origen: fila `{transferId}-out`, `direction: "out"`, campo `from` de la respuesta) y la **pata de entrada** (crédito en la cuenta destino: fila `{transferId}-in`, `direction: "in"`, campo `to`). Están vinculadas por el mismo `transferId`. No es la persona que envía o recibe: en una transferencia entre cuentas propias, las dos patas son del mismo usuario |
| **Saldo contable** | Sinónimo de "Saldo" cuando hace falta distinguirlo del disponible |
| **P2P** | Transferencia hacia una cuenta de otro usuario |

---

## No implementado y por qué

| Qué | Por qué |
|---|---|
| Conversión real de moneda (bonus) | El contrato de la API acepta un solo `amount` por transferencia: no hay forma de debitar X y acreditar Y. Ver ambigüedad 1 |
| Optimistic updates (bonus) | A propósito: mostrar un estado que todavía no es cierto es mentir sobre el dinero. Ver ambigüedad 2 |
| Crear y renombrar cuentas | No es requisito. Es la única forma de tener una cuenta en EUR, que el seed no trae. La política de moneda ya contempla EUR |
| Sincronización entre pestañas | Riesgo residual, mitigado con el refresco al volver a la pestaña y con el 422 |
| `Idempotency-Key` | La API no lo soporta. Simularlo daría una falsa sensación de protección |

---

## Preguntas abiertas

Son las preguntas que le haría a producto o backend si pudiera, con el supuesto que tomé mientras tanto:

| Pregunta | Supuesto |
|---|---|
| ¿El 422 debería considerar las transferencias pendientes? ¿"Saldo disponible" es el contable? | Disponible = saldo − salientes en proceso |
| ¿Existe una forma de conocer titular y moneda de una cuenta ajena (como un alias o un CBU)? | No: el usuario declara la moneda y se verifica después del envío |
| ¿Cuándo puede una transferencia pasar a `failed`? | Se contempla aunque hoy no ocurra |
| ¿Los gastos deberían validar fondos? | Sí, en el cliente |
| ¿Qué pasa con una transferencia `pending` si el servidor se reinicia? | Puede quedar pendiente para siempre: por eso el timeout muestra "en proceso" |

## Propuestas para el backend

El `mock-api` no se modifica, pero estas son las correcciones de fondo que propondría:

1. **Reservar fondos** al crear una transferencia, o descontar las pendientes en el chequeo del 422. Cierra el doble gasto.
2. **Rechazar transferencias entre monedas distintas**, o exponer una búsqueda de destinatario con titular y moneda.
3. **`Idempotency-Key`** en los `POST`.
4. **Un 422 estructurado:** `{ code, available, requested }`.
5. **Liquidación persistente** (cola o job) en lugar de un `setTimeout` en memoria.
6. **Chequeo de fondos también en los gastos.**
7. **Desempate en el orden** de los movimientos: `ORDER BY createdAt DESC, id`.
8. **Errores 500 en JSON**, no en HTML.

---

## Qué cambió al implementar

> Se completa al final del desarrollo.
