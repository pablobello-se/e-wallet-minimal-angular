# Uso de IA

> Este documento empieza **antes de implementar**, con lo que pasó en el análisis y la toma de decisiones. La sección [Implementación](#5-implementación) se completa durante el desarrollo.

**Herramienta:** Claude Code (Opus) dentro de VS Code, con acceso de lectura al repo.

**En una línea:** primero leí y analicé la consigna y la documentación de la API por mi cuenta. Con esa lectura le di contexto a la IA y le pedí analizar la documentación y el código. Todo lo que produjo pasó por una etapa de **revisión crítica** de mi parte, en la que decido, cuestiono y verifico contra la API real o con tests antes de aceptar.

---

## 1. Estrategia

### Cómo estructuré el trabajo

El orden fue siempre el mismo: **mi lectura → contexto → análisis de la IA → revisión crítica mía**.

| Fase | Qué hice yo | Qué le pedí a la IA |
|---|---|---|
| Lectura propia | Leer y analizar la consigna y el README de la API: alcance, prioridades y los puntos ambiguos | Nada: esta fase fue sin IA |
| Contexto | Pasarle mi lectura, las prioridades y las reglas de trabajo | — |
| Análisis | — | Analizar la documentación **y el código** de la API (`mock-api/index.ts`), y listar lo que encontrara |
| Revisión crítica | Revisar, descartar y refinar; verificar los hallazgos que cambiaban decisiones (ver [2](#2-multimoneda), [3](#3-estados-asincrónicos) y [4](#4-errores)) | — |
| Ambigüedades | Elegir, y rechazar o modificar la recomendación cuando no me convencía | Una por una: opciones posibles, trade-offs y una recomendación |
| Documentación | Revisar, corregir y reestructurar antes de commitear | Borradores de `DECISIONS.md` y `ARCHITECTURE.md` a partir de las decisiones cerradas |
| Implementación | Revisar el diff, probar y commitear cada paso | Código **por capa** (dominio → infraestructura → UI por feature), un paso a la vez |

### Qué contexto le di

- **Mi propia lectura de la consigna y de la API:** alcance, prioridades y los tres puntos ambiguos, como punto de partida del análisis.
- **Un documento de contexto vivo** (fuera del repo) con las reglas de trabajo, los hallazgos sobre la API, las decisiones tomadas y el plan. Cada sesión parte de ahí, no de la memoria de la conversación anterior.
- **`CLAUDE.md`** en el repo, con las reglas de arquitectura (qué capa puede importar qué), para que el código generado las respete. *(A crear en el setup.)*
- **El MCP de Angular** (`ng mcp`), para que la IA consulte la documentación y las prácticas actuales de Angular 20 en lugar de depender de lo que recuerda. *(A configurar en el setup.)*

### Reglas de trabajo con la IA

- **No hace ningún cambio en archivos sin que se lo pida explícitamente.** Leer y analizar, sí; proponer, en el chat.
- **La API es una caja negra:** no se modifica, aunque tenga bugs. Los problemas se documentan como propuestas para el backend.
- **Los commits y los push los hago yo**, después de revisar.
- **Un hallazgo no es un hecho hasta que se verifica.** Si la IA deduce algo del código, lo pruebo contra la API antes de tomar decisiones sobre eso.

---

## 2. Multimoneda

**El caso central de "¿la aceptaste tal cual o la cuestionaste?".**

| | |
|---|---|
| **Propuesta de la IA** | En transferencias a otro usuario, **advertir** si las monedas podían no coincidir y dejar que el usuario decida |
| **Lo que cuestioné** | Advertir es permitir la corrupción de datos con el consentimiento del usuario. El problema no es de UX, es de integridad: si salen 5.000 ARS, entran 5.000 USD. Con esta API, solo cabe bloquear o convertir |
| **Lo que precisó la IA** | Aceptó el argumento y agregó dos cosas: convertir no es imposible por arquitectura sino por el **contrato** de la API (un solo `amount`); y bloquear **todas** las transferencias a otro usuario incumpliría un requisito mínimo de la consigna |
| **Resultado: modificada** | Bloquear entre cuentas propias de distinta moneda; en transferencias a otro usuario, moneda declarada + bloqueo + detección después del envío; la solución de fondo, documentada para el backend |

Detalle en [DECISIONS: multimoneda](./DECISIONS.md#1-multimoneda).

---

## 3. Estados asincrónicos

| | |
|---|---|
| **Hallazgo de la IA** | Leyendo el código, dedujo que el saldo que valida el 422 solo cuenta movimientos `completed`. Si eso era cierto, dos transferencias seguidas dentro de la ventana de `pending` podían pasar las dos y dejar la cuenta en negativo. **El README no lo dice** |
| **Lo que hice** | No lo acepté como cierto. Lo verifiqué con el Collection Runner de Postman (2 iteraciones, sin delay): desde $855.000, dos transferencias de $500.000 devolvieron 201 y la cuenta quedó en −$145.000 |
| **Corrección de la lectura** | La IA lo había planteado como una condición de carrera. Se corrigió: no lo es (Node procesa cada request entero y SQLite es sincrónico). Es la ventana de `pending` sin reserva de fondos, y pasa igual con dos envíos secuenciales |
| **Resultado: verificado** | Cambió tres decisiones: el saldo **disponible** pasó a ser obligatorio, el bloqueo de doble envío pasó a ser crítico y apareció la diferencia entre saldo disponible y saldo contable como una ambigüedad más |

Este hallazgo une las ambigüedades 2 y 3. Detalle en [DECISIONS: fondos insuficientes](./DECISIONS.md#3-fondos-insuficientes-422).

---

## 4. Errores

- **README vs. código.** El README describe el 422 con un formato de error estructurado; el código devuelve otra cosa. Se detectó porque le pedí a la IA leer el código y no solo la documentación. Consecuencia: el adapter traduce los errores por status, sin depender del texto que manda la API.
- **`npm install` falló** al instalar la API (`better-sqlite3` en Node 24 / Windows). La IA diagnosticó a partir del log de npm y propuso usar Node 22 LTS **en lugar de modificar la API**, que es la regla.
- **Dudas que la IA marcó como dudas.** Sobre el builder de tests de Vitest en Angular 20.3 y sobre los comandos de IA del CLI, no respondió de memoria: se verificaron en `node_modules`. El builder de Vitest figura como `[EXPERIMENTAL]` y por eso se descartó (ver [DECISIONS: herramientas](./DECISIONS.md#herramientas)).

---

## 5. Implementación

> Se completa durante el desarrollo, un registro por paso:
>
> **Paso → qué se pidió → qué generó la IA → qué revisé o corregí → cómo se verificó.**

---

## 6. Lo que no le delegué a la IA

- **La primera lectura.** El análisis de la consigna y de la API lo hice yo antes de usar la IA; ella partió de ese contexto.
- **Las decisiones de producto.** La IA presentó opciones y recomendó; la elección y su justificación son mías.
- **La verificación.** Ningún hallazgo sobre la API se usó sin probarlo, y ningún paso de código se commitea sin revisar el diff y probarlo.
- **Los criterios.** Por ejemplo, para elegir la librería de componentes fijé un criterio de eficiencia temporal y "baja polución del resultado"; la IA propuso Angular Material justificándolo con las decisiones que necesitan esos componentes y acotándolo a la capa de UI.
- **La forma de los documentos.** La IA redactó los borradores con una estructura y contenido iniciales, que luego pasaron por una fase final de análisis y corrección de mi parte.
- **La arquitectura.** La IA descartó en un principio Clean Architecture como incompatible con hexagonal; se corrigió: son compatibles, y la arquitectura final usa las dos (ver [ARCHITECTURE](./ARCHITECTURE.md#estilo-elegido)).
