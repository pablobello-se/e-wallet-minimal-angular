# e-wallet-minimal-angular

Solución al ejercicio **Frontend Developer 2026** de Greelow ([repo original](https://github.com/Greelow-LLC/frontend-exercise-2026)).

## Documentación inicial

- [`Ejercicio-Frontend-Developer.md`](./Ejercicio-Frontend-Developer.md) — consigna del ejercicio.
- [`mock-api/`](./mock-api) — API fintech chica (Node + TypeScript + Express + SQLite) que se le entrega al candidato junto con la consigna.

## Documentación de la solución / entregables

La documentación presentada es extensa, dado que la idea es justamente identificar, verificar, documentar y justificar todas las ambigüedades o puntos pobremente definidos (intencionalmente) a lo largo de toda la consigna, para posterior implementación de las soluciones a dichos puntos y/o ambigüedades. Dado que se está tratando con un sistema (si bien ficticio) de gestión de información financiera, el tratamiento cuidadoso y atento de la información es un aspecto central.

- [Instalación y ejecución](#instalación-y-ejecución) — instrucciones para levantar la API y la UI (requisito obligatorio).
- [`DECISIONS.md`](./docs/DECISIONS.md) — documentación de decisiones (requisito obligatorio).
- [`AI_NOTES.md`](./docs/AI_NOTES.md) — documentación del uso de IA generativa (requisito obligatorio).
- [`ARCHITECTURE.md`](./docs/ARCHITECTURE.md) — documentación de decisiones de arquitectura (requisito opcional/bonus).

## Instalación y ejecución

**Requisitos:** Node 22 LTS (ver [`.nvmrc`](./.nvmrc); `better-sqlite3` no compila en Node 24 sobre Windows), npm y Google Chrome para los tests.

### 1. API (`mock-api`)

```bash
cd mock-api
npm install
npm run seed     # crea data.db con datos de ejemplo; correrlo de nuevo resetea la base
npm start        # http://localhost:3000
```

### 2. UI (`ui`)

En otra terminal, con la API corriendo:

```bash
cd ui
npm install
npx ng serve     # http://localhost:4200
```

### 3. Usuarios de prueba

| Email | Password | Cuentas |
|---|---|---|
| `demo@example.com` | `demo123` | ARS y USD |
| `test@example.com` | `test123` | ARS |
| `ana@example.com` | `ana123` | ARS |

`demo` tiene cuentas en dos monedas, lo que permite probar el bloqueo multimoneda. Las transferencias a otro usuario se prueban entre `demo` y `test` o `ana`.

### 4. Tests

```bash
cd ui
npm test -- --watch=false --browsers=ChromeHeadless
```
