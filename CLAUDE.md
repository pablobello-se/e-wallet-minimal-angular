# CLAUDE.md

Contexto para asistentes de IA que trabajan en este repo. Las decisiones de producto están en [`docs/DECISIONS.md`](./docs/DECISIONS.md) y la arquitectura en [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md): leelos antes de proponer cambios.

## Reglas de trabajo

- **No hacer ningún cambio en archivos sin un pedido explícito.** Leer y analizar, sí; las propuestas van en el chat.
- **`mock-api/` no se modifica nunca.** Es una caja negra. Si tiene un bug, se documenta como propuesta para el backend en `DECISIONS.md`.
- **Commits y push los hace el desarrollador.** No ejecutar `git commit`, `git push` ni comandos que reescriban historial.
- **Un hallazgo no es un hecho hasta que se verifica** (contra la API real o con un test). Si algo no se sabe, decirlo.
- Un paso del plan = una rama = un PR.

## Stack

- Angular 20.3 standalone, TypeScript estricto, Signals, Angular Material 20.
- Tests: Karma + Jasmine (no Vitest: en 20.3 es experimental).
- Node 22 LTS (`.nvmrc`). `better-sqlite3` no compila en Node 24 sobre Windows.
- Usar el CLI del proyecto (`npx ng`), no un CLI global de otra versión.

## Comandos

```bash
# API (http://localhost:3000)
cd mock-api && npm install && npm run seed && npm start

# UI (http://localhost:4200)
cd ui && npm install && npx ng serve

# Tests de la UI
cd ui && npm test -- --watch=false --browsers=ChromeHeadless
```

## Arquitectura (resumen)

Hexagonal (Ports & Adapters) con las capas de Clean. **Las dependencias apuntan hacia adentro.** Todo vive en `ui/src/app/`:

| Capa | Puede importar | No puede importar |
|---|---|---|
| `domain/` | Nada | Angular, RxJS, HTTP, DTOs de la API |
| `application/` (ports, use-cases) | `domain/` | Angular, RxJS, HTTP |
| `infrastructure/` (adapters) | `domain/`, `application/`, Angular, RxJS | `features/` |
| `features/`, `shared/ui/` | `application/`, `domain/` (tipos y formato), Angular, Material | `infrastructure/` (solo vía inyección) |

- `app.config.ts` es el composition root: ahí se enlaza cada port con su adapter (`InjectionToken`).
- Los ports devuelven `Promise`. RxJS solo en infraestructura (`firstValueFrom` en el adapter).
- Angular Material solo en `features/` y `shared/ui/`.
- Los componentes no tienen lógica de negocio: delegan en stores (Signals), que delegan en casos de uso.

## Reglas de dominio que no se negocian

- **Dinero:** `Money { minor: entero, currency }`. Nunca `number` decimal para montos. La conversión desde/hacia el `amount` de la API se hace solo en el adapter.
- **Saldo disponible** = saldo de la API − transferencias salientes `pending`. Se valida contra el disponible, no contra el saldo de la API. Al consultar: primero pendientes, después cuentas.
- **Multimoneda:** se bloquea toda transferencia entre monedas distintas.
- **Doble envío:** single-flight en el caso de uso. **Nunca reintentar un `POST` automáticamente.**
- **Estado de una transferencia:** polling con `nextPollStep` y el port `Scheduler`. Sin updates optimistas. Si el `POST` no responde, el resultado es *desconocido*, no un error ni un éxito.
- **Errores de la API:** un único `mapApiError` los traduce a errores de dominio. Fuera de `infrastructure/` nadie ve status HTTP ni bodies de la API.
- **Edición:** solo categoría y descripción de ingresos y gastos (`canEdit`). Descripción ≤ 500 caracteres; monto ≥ 0,01.

## Testing

- Dominio con TDD: tests unitarios puros, sin `TestBed`.
- Casos de uso con ports fake y reloj falso: sin HTTP ni timers reales.
- Infraestructura: mappers y `mapApiError` contra la tabla de errores de `DECISIONS.md`.
