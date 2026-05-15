# Sprint 1 — Reporte de entrega

**Fecha de entrega:** Viernes 15 de mayo, 2026
**Épica entregada:** WhatsApp Asesor-Cliente con motor automático de
journey postventa
**Tenant pilot:** Astara Retail (`demo@autosummit.cl`)
**Branch entregada:** `sprint1/close`
**Líder técnico:** Diego Quiroga · **PO:** Óscar

> Documento de entrega formal al comité. Para detalle técnico granular ver
> `close-report.md` (cierre de ingeniería) y `qa-e2e-report.md` (pasada QA
> E2E con DB real). Para riesgos diferidos ver `deferred-debt.md`. Para la
> grabación demo ver `demo-script.md`.

---

## 1. Veredicto

**Sprint 1 entregado en verde, con un único bloqueo comercial pendiente
(KAM 7 — credenciales Chattigo compartidas).**

| Dimensión | Estado | Evidencia |
|---|---|---|
| HUs comprometidas | 8/10 cerradas completas, 2/10 cerradas con activación productiva diferida | `close-report.md` §detalle por HU |
| Story points | ~36 / 39 entregados (92%) | Plan Scrum Sprint 1 |
| Tests automatizados | **90/90 vitest** (5 sumados durante QA) | CI local, `pnpm test` |
| Typecheck backend + frontend | OK · 0 errores | `pnpm typecheck` |
| QA E2E con DB real | 16/16 escenarios curl verdes | `qa-e2e-report.md` |
| Bugs encontrados durante QA | 4 (2 críticos, 1 alto, 1 medio) — todos arreglados | `qa-e2e-report.md` §bugs |
| Pasada visual usuario | Completada hoy 2026-05-15, fixes UX aplicados in-line | Sección 4 de este reporte |
| ADR-001 (port hexagonal WhatsAppGateway) | Implementado y testeado | `docs/adr/0001-service-layer-pattern.md` |
| Migración DB | 1 migration aditiva, sin downtime | `20260515032528_sprint1_whatsapp_advisor_channel` |

**Lo que mañana queda en producción:** todo el código de Sprint 1 corriendo
en VPS con `WHATSAPP_GATEWAY_ADAPTER=in-memory` por la doctrina del adapter.
La demo se ve y se comporta igual que con Chattigo real (persiste en DB,
respeta ventana 24h, simula status updates con UUIDs). El cambio a
`chattigo` es una sola variable de entorno cuando el comité destrabe KAM 7.

---

## 2. Lo que se entrega — épica completa

### Backend
- **Port hexagonal `WhatsAppGateway`** con dos adapters intercambiables
  (`InMemoryGateway` para demo + tests, `ChattigoGateway` para producción).
  Cliente Chattigo con JWT cache 7h, coalescing de refresh, retry exponencial
  1s/2s/4s y refresh automático en 401.
- **Webhooks `/inbound` y `/notifications`** con bearer shared secret,
  idempotencia por `(externalProvider, externalMessageId)`, mappers para
  los 5 eventos Chattigo (SENT, DELIVERY, READ, SESSION, INVALID).
- **Persistencia transaccional** (`whatsapp-persistence.ts`) — `Conversation`
  + `Message` atómicos, validación zod, errores tipados, ventana 24h
  poblada en cada inbound.
- **`AdvisorChannel`** modelo nuevo + CRUD completo (GET / POST / DELETE +
  listado para gerentes), partial unique "un canal activo por advisor".
- **`Message` y `Conversation`** extendidas con campos `external*`
  (provider-agnóstico, alineado a ADR-002 — ya listo para migración a
  Meta Cloud API si fuera necesario).
- **5 plantillas WhatsApp** seedeadas y verificadas (`activacion_postventa_dia1`,
  `confirmacion_cita`, `recordatorio_24h`, `auto_listo_retiro`, `encuesta_nps`).
- **Healthcheck `/api/health/whatsapp`** con ping real al `POST /login`
  Chattigo cuando `adapter=chattigo`.

### Frontend
- **`AdvisorChannelDrawer`** integrado en `AsesoresPage` — gerente ve la
  columna "Canal WhatsApp" de toda la fuerza de ventas; asesor edita su
  propio canal.
- **`ConversationsPage` composer** — selector modal de plantillas con
  variables dinámicas, iconos status en burbujas (Clock / Check / CheckCheck /
  AlertCircle), sending state optimistic, manejo visible de errores 412/422.
- **Doctrina WhatsApp-only respetada** — sin selectores de canal en
  ninguna superficie del tenant.

### Documentación
- `docs/sprint1/close-report.md` — cierre de ingeniería
- `docs/sprint1/qa-e2e-report.md` — pasada QA con DB real
- `docs/sprint1/deferred-debt.md` — deuda explícita con justificación
- `docs/sprint1/demo-script.md` — guion de la demo (Loom/comité)
- `docs/sprint1/delivery-report.md` — este documento
- `docs/sprint2/kickoff.md` — plan operativo del próximo sprint
- `docs/adr/0001-service-layer-pattern.md` — port hexagonal WhatsAppGateway

---

## 3. Riesgo principal — KAM 7 (credenciales compartidas)

**Las credenciales Chattigo del KAM están atadas a otra solución productiva
de los socios.** Si activamos el adapter `chattigo` mañana, intercalaríamos
mensajes con esa otra solución y romperíamos su flujo.

**Mitigación implementada:** la doctrina `InMemoryGateway` permite cerrar
Sprint 1 y correr la demo del comité sin contaminar producción ajena. Toda
la lógica de persistencia, ventana 24h, status updates y multi-tenant
está probada con datos reales.

**Pedido al comité — decisión operativa:**
- (a) KAM provee credenciales sandbox separadas → activable en horas.
- (b) Goopcar tramita un `did` propio Meta-verified exclusivo MVP → semanas.
- (c) Sprint 2 corre también en `InMemoryWhatsAppGateway`. Producción real
       difiere a Sprint 3.

Recomendación del líder técnico: **(a)**. Sin esto, Sprint 2 entrega los
triggers automáticos pero también en in-memory, lo que aún no genera
revenue real para el tenant.

---

## 4. Pasada visual del usuario (completada hoy)

Durante la pasada visual del 2026-05-15 surgieron 2 mejoras UX que se
aplicaron directamente en branch antes del cierre y 2 ítems que se
movieron al backlog Sprint 2 con justificación.

**Aplicadas in-line (commit pendiente):**
- `apps/web/src/pages/app/ConversationsPage.tsx` — refinamientos del
  composer y selector de plantillas detectados durante la pasada.
- `apps/api/src/routes/app/templates.ts` — ajustes en el endpoint que
  alimenta el modal.

**Movidas a backlog Sprint 2 (`deferred-debt.md` D-12 + D-13):**
- **D-12** · Resolver autofill ANTES de abrir el modal — el modal abre con
  header de contexto ("Cliente · Vehículo · Sucursal") y los inputs llegan
  pre-llenados sin necesidad de seleccionar plantilla primero.
- **D-13** · `template-variable-resolver.ts` centralizado en backend —
  resuelve TODAS las variables conocidas (`nombre_cliente`, `marca`, `modelo`,
  `patente`, `sucursal`, `fecha_hora_cita`, `amount`, `tenantName`, `tecnico`)
  desde la conversación + appointment + service order + monetized action.

**Ambas son dependencia bloqueante de Sprint 2** porque los triggers
automáticos (HU-012 confirmación, HU-013 recordatorio, HU-016 NPS) usan
exactamente el mismo resolver. Sin él, los mensajes automáticos saldrían
con `{{sucursal}}` literal. Marcadas como ítem día 3 del kickoff Sprint 2.

---

## 5. Deuda diferida — registro completo

Trece (13) ítems registrados en `deferred-debt.md` con origen, síntoma,
justificación de diferimiento y plan. Resumen por severidad:

| Severidad | Cantidad | Más relevantes |
|---|---|---|
| CRÍTICO | 1 | D-1 cross-tenant en partial unique de `Message` (se trae al promover a productivo o sumar segundo tenant) |
| ALTO | 4 | D-2 Optimistic UI con rollback (Sprint 2 + ADR React Query); D-12, D-13 (Sprint 2 día 3 — bloqueantes); D-3 Playwright + CI workflow |
| MEDIO | 4 | logging correlation IDs, refactor `ConversationDetailPage` legacy, etc. |
| BAJO | 4 | anotaciones |

**Ningún ítem CRÍTICO bloquea la demo ni el uso del MVP por Astara Retail
en su tenant.** El único CRÍTICO (D-1) sólo aplica al sumar segundo tenant.

---

## 6. Próximo sprint — Sprint 2 (Lun 18 May)

Detalle completo en `docs/sprint2/kickoff.md`. Resumen:

**Épica:** Motor automático de journey post-venta — del agendamiento al NPS.

- **HU-011 a HU-014** — FSM `Appointment` + cron jobs de confirmación
  (`confirmacion_cita`), recordatorio 24h y detección no-show.
- **HU-015** — Botón "Listo para retiro" en bandeja, dispara `auto_listo_retiro`.
- **HU-016** — Trigger NPS post-entrega + captura del puntaje del cliente.
- **HU-017** — Motor de plantillas con variables nombradas → posicionales
  `{{1}} {{2}}` Meta-spec. **Bloqueante:** incluye D-12 + D-13 del backlog
  diferido.
- **HU-018** — Métricas de canal (mensajes enviados, entregados, leídos,
  rate de respuesta).
- **HU-019** — Dashboard mínimo de motor de journey en panel gerente.
- **HU-020** — Demo formal Astara Retail.

**En paralelo (no bloqueante de Sprint 2 pero sí de revenue real):**
destrabar KAM 7. Mientras tanto Sprint 2 sigue en `InMemoryWhatsAppGateway`.

---

## 7. Aprobaciones requeridas del comité

Para considerar Sprint 1 formalmente entregado y desbloquear Sprint 2,
necesitamos firma sobre:

1. **Aceptación funcional** del cierre — basada en el demo Loom + recorrido
   visual del entorno. Owner: PO.
2. **Decisión KAM 7** — opción (a), (b) o (c) de la sección §3. Owner: comité
   comercial.
3. **Confirmación del scope Sprint 2** — kickoff Lun 18 May 09:00. Owner: PO + líder técnico.

---

*Confidencial · Goopcar / Cognitiva SpA · Reporte de entrega preparado por
el líder técnico · 2026-05-15.*
