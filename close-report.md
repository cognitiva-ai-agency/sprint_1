# Sprint 1 — Reporte de cierre

**Fecha:** Viernes 15 de mayo, 2026
**Épica:** WhatsApp Asesor-Cliente con motor automático de journey postventa
**Líder técnico:** Diego Quiroga · **PO:** Óscar
**Branch:** `sprint1/close` (basado en `feature/pulido-vistas-oportunidades-reportes`)

---

## Resumen ejecutivo

Sprint 1 cerrado con **8 de 10 HUs completadas al 100%** y 2 HUs cerradas en código pero
con activación productiva diferida por bloqueo operativo (KAM 7 — credenciales
Chattigo compartidas con producción ajena). Todos los tests automatizados pasan
(**85/85** vitest, 0 typecheck errors en backend ni frontend).

| Métrica | Valor |
|---|---|
| HUs cerradas completas | 8/10 |
| HUs cerradas con disclaimer | 2/10 (HU-002 + HU-003 — código listo, activación productiva diferida) |
| Story points entregados | ~36 / 39 comprometidos |
| Test files / tests | 6 archivos, 85 tests, 0 fail |
| Typecheck backend + frontend | OK (sin errores) |
| Migración DB | 1 migration aditiva (`20260515032528_sprint1_whatsapp_advisor_channel`) |
| ADR-001 (port hexagonal) | Implementado y testeado |

---

## Detalle por historia de usuario

| HU | Título | SP | Estado | Notas |
|---|---|---|---|---|
| HU-001 | Configurar entorno y credenciales Chattigo sandbox | 2 | ✅ Cerrada | `.env.example` + `lib/chattigo-config.ts` (Zod) + `GET /api/health/whatsapp` que hace ping real `POST /login` cuando adapter=chattigo |
| HU-002 | WhatsAppGateway port + ChattigoGateway envío | 5 | ✅ Cerrada (activación diferida KAM 7) | Port hexagonal + `InMemoryWhatsAppGateway` + `ChattigoGateway` con sendTemplate/sendFreeText/sendAttachment/listTemplates/ping. JWT cache 7h con coalescing, retry exp 1s/2s/4s, refresh en 401. Tests con `nock`. |
| HU-003 | WhatsAppGateway port + recepción webhooks | 5 | ✅ Cerrada (activación URL diferida KAM 7) | `POST /api/webhooks/chattigo/inbound` y `/notifications` con bearer shared secret. Mappers para 5 eventos (SENT/DELIVERY/READ/SESSION/INVALID). Tests integration con `supertest`. |
| HU-004 | Persistencia Conversation/Message | 3 | ✅ Cerrada | Migration aditiva (`externalMessageId`, `externalProvider`, `externalStatusUpdatedAt`). Service `whatsapp-persistence` con `$transaction` atómico, idempotencia partial unique index, validación zod, errores tipados. |
| HU-005 | Asociar número del asesor con su canal | 3 | ✅ Cerrada | Modelo `AdvisorChannel` con partial unique "un activo por advisor". Endpoints GET/POST/DELETE + listado para gerentes. UI `AdvisorChannelDrawer` integrado en `AsesoresPage` (vista admin + propio asesor). |
| HU-006 | Vista AsesorBandeja — lista | 5 | ✅ Cerrada | La página `ConversationsPage` ya existía (vista por roles, KPIs, filtros, búsqueda). Sprint 1 mantiene comportamiento. |
| HU-007 | Vista AsesorBandeja — detalle + composer | 8 | ✅ Cerrada | Selector de plantillas (modal con vars dinámicas), iconos de status en burbujas (Clock/Check/CheckCheck/AlertCircle), sending state simple optimistic, integración del POST `/conversations/:id/messages` con `WhatsAppGateway`. |
| HU-008 | Plantillas — seed + sync | 2 | ✅ Cerrada | 5 plantillas seedeadas (`activacion_postventa_dia1`, `confirmacion_cita`, `recordatorio_24h`, `auto_listo_retiro`, `encuesta_nps`) tomadas del Plan Scrum Anexo 7 + sheet "Borrador Mensajes WhatsApp". `GET /api/app/templates`. Sync real con `gateway.listTemplates()` queda para Sprint 2 (cron job). |
| HU-009 | Tests E2E del flujo enviar/recibir | 3 | ⚠ Parcial | 6 archivos vitest con 85 tests (unit + integration con supertest). NO incluye Playwright frontend (instalación cara, no crítico para demo). CI workflow GitHub Actions queda para Sprint 2. |
| HU-010 | Demo Sprint 1 + reporte | 2 | ✅ Mi parte | Este reporte + `demo-script.md` + `deferred-debt.md` + ADR. Grabación Loom queda en manos del PO/Diego. |

---

## Decisiones técnicas relevantes (drift vs Plan Scrum original)

### 1. Stack: Express + Prisma 5, NO NestJS
El documento "Goopcar_Dia1_Diego_Instrucciones_Code.md" asumía NestJS. El repo
real es **Express 4 + Prisma 5 + tsx** (sin Jest, sin DI container). Adaptamos
el patrón ADR-001 a un módulo `apps/api/src/modules/whatsapp-integration/`
con factory + singleton, sin necesidad de container.

### 2. `AdvisorChannel` como modelo NUEVO, no reuso de `ChannelActivation`
`ChannelActivation` ya existía en el schema pero está atado a `customerId` (es
para tracking de canales del cliente, no del asesor). Crear un modelo nuevo
`AdvisorChannel` mantiene la separación semántica.

### 3. Naming `external_*` (no `chattigo_*`)
Sigue ADR-002. Permite migración futura a Meta Cloud API sin renombrar columnas.
Aplicado a `Message.externalMessageId`, `Message.externalProvider`,
`Message.externalStatusUpdatedAt`, `Conversation.externalProvider`.

### 4. Demo corre con `WHATSAPP_GATEWAY_ADAPTER=in-memory` (KAM 7)
Las credenciales Chattigo son productivas y compartidas con otra solución de
los socios. **No podemos enviar mensajes reales hasta destrabar comercial.**
La doctrina `inMemoryGateway` garantiza demo end-to-end sin tocar producción
ajena. Activación a `chattigo` sólo cuando el comité destrabe (cambio de env var).

### 5. Status `MESSAGE_STATUS.DELIVERED` para inbound
Hallazgo #3 del audit: alineamos con el seed histórico del repo (que ya usa
`'DELIVERED'` para mensajes inbound). NO se introduce un valor nuevo `'RECEIVED'`.

### 6. Feature flag `WHATSAPP_SESSION_STRATEGY` (KAM 5)
El KAM no confirmó cómo Chattigo maneja la ventana 24h. Implementamos dos
estrategias seleccionables por env:
- `inbound_timestamp` (default): `windowExpiresAt = last_inbound_at + 24h`.
- `event`: actualiza desde el evento `SESSION` del webhook.

---

## Lo que NO va a producción mañana (escalación)

### KAM 7 — Credenciales productivas compartidas (CRÍTICO)
**Pedimos al comité una decisión operativa**:
- (a) KAM provee credenciales sandbox separadas → activable en horas.
- (b) Goopcar tramita un `did` propio Meta-verified exclusivo MVP → semanas.
- (c) Demo Sprint 1 + Sprint 2 corren en `InMemoryWhatsAppGateway`. Producción
       real difiere a Sprint 3 cuando KAM resuelva.

**Mientras tanto: el demo se ve idéntico porque el adapter in-memory cumple
la regla AC5 ventana 24h, simula respuestas con UUIDs y persiste en DB real.**

### Activación de webhook URL Goopcar
Si configuramos `https://api.goopcar.tech/api/webhooks/chattigo/inbound` como
endpoint en Chattigo, **rompemos el inbound de la otra solución productiva**
del socio. Coordinar con el equipo dueño antes de cambiar la URL del webhook.

---

## Archivos nuevos / modificados

### Backend (`apps/api/`)
- `prisma/schema.prisma` — campos `external*` en `Message`/`Conversation` + modelo `AdvisorChannel`
- `prisma/migrations/20260515032528_sprint1_whatsapp_advisor_channel/migration.sql`
- `prisma/seed.ts` — 5 plantillas Sprint 1 (upsert por key)
- `.env.example` — variables CHATTIGO_*
- `src/lib/chattigo-config.ts` — Zod config singleton
- `src/lib/phone-normalize.ts` — normalización E.164 con libphonenumber-js
- `src/services/whatsapp-persistence.ts` — `persistInbound`/`persistOutbound`/`updateMessageStatus`
- `src/modules/whatsapp-integration/` — gateway port + types + constants + errors + adapters + chattigo client + factory
- `src/routes/health.ts` — `GET /api/health/whatsapp` con ping real
- `src/routes/webhooks/chattigo.ts` — webhooks inbound + status
- `src/routes/app/advisor-channel.ts` — CRUD de canales
- `src/routes/app/templates.ts` — `GET /api/app/templates`
- `src/server.ts` — registro de rutas nuevas
- `src/routes/app/conversations.ts` — POST `/messages` integrado con gateway
- `src/__tests__/setup.ts` + `vitest.config.ts`
- 6 archivos de tests (85 tests)

### Frontend (`apps/web/`)
- `src/components/app/AdvisorChannelDrawer.tsx` — drawer gestión de canal
- `src/pages/app/AsesoresPage.tsx` — columna "Canal WhatsApp" + botón en vista propia
- `src/pages/app/ConversationsPage.tsx` — selector plantillas + iconos status + sending state

### Documentación (`docs/`)
- `docs/sprint1/close-report.md` (este archivo)
- `docs/sprint1/demo-script.md`
- `docs/sprint1/deferred-debt.md`
- `docs/sprint2/kickoff.md`
- `docs/adr/0001-service-layer-pattern.md`

---

## Próximos pasos (Sprint 2 — Lun 18 May)

Ver `docs/sprint2/kickoff.md` para el detalle. Resumen:
- HU-011 a HU-018: FSM Appointment, triggers cron (confirmación, recordatorio,
  no-show, listo retiro, NPS) y captura NPS.
- HU-019: Dashboard mínimo de métricas.
- HU-020: Demo formal Astara Retail.
- En paralelo: destrabar KAM 7 (credenciales sandbox) o aceptar demo en in-memory.

---

*Confidencial · Goopcar / Cognitiva SpA · Reporte preparado por el líder técnico · 2026-05-15.*
