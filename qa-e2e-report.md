# Sprint 1 — Reporte de QA E2E

**Fecha:** 2026-05-15
**Ejecutado por:** Diego Quiroga (líder técnico) con asistencia Claude Code
**Stack QA:** Postgres 16 local (docker), backend `pnpm dev` en `:4000`,
frontend `pnpm dev` en `:3000` apuntando a backend local.

---

## Resumen ejecutivo

**Backend: VERDE.** 16 escenarios curl probados contra DB real (no mocks).
4 bugs encontrados y arreglados durante la pasada. Tests automatizados
**90/90** pasando (5 nuevos sumados durante QA).

**Frontend: pendiente pasada visual del usuario.** Servidor levanta sin
errores, todas las APIs que consume responden correctamente, sin
regresión visible en datos de bandeja existente.

---

## Bugs encontrados y arreglados durante QA E2E

### Bug Q-1 (CRÍTICO) — `InMemoryGateway.parseInboundWebhook` requería `enqueueInbound()`
**Síntoma:** El adapter in-memory (default por KAM 7) lanzaba error al recibir
un webhook real con body Chattigo, pidiendo que se haya pre-enqueued un mensaje
(comportamiento sólo válido en tests). En producción con adapter in-memory, el
webhook fallaba 500/200 silencioso, ningún mensaje persistido.

**Fix:** Comportamiento dual — si la queue está vacía, parsea el body real
usando los mismos mappers de Chattigo (`mapChattigoInboundToInboundMessage`).
Requiere `setTenantResolver()` inyectado por el factory desde Prisma.

**Verificación:** Test 1 con bearer válido + payload Chattigo → 200, mensaje
persistido en DB con todos los campos correctos (Customer creado, Conversation
abierta, Message con externalProvider='in-memory').

### Bug Q-2 (CRÍTICO) — `InMemoryGateway` no validaba bearer en webhooks
**Síntoma:** Cualquier request al webhook (incluso sin Authorization header)
respondía 200 y persistía el mensaje. **Vulnerabilidad de inyección.**

**Fix:** Agregada validación opcional de bearer (`setWebhookTokens()`)
inyectada por el factory desde `CHATTIGO_WEBHOOK_*_TOKEN` envs. Comparación
constant-time idéntica a `ChattigoGateway`.

**Verificación:** Test 2 (bearer inválido) → 401. Test 3 (sin Authorization)
→ 401. Bearer válido → 200.

### Bug Q-3 (ALTO) — Resolución de AdvisorChannel priorizaba el caller, no el asesor de la conversación
**Síntoma:** El POST `/conversations/:id/messages` buscaba el canal del
`userId` (caller) ANTES de `assignedAdvisorId`. Si un TENANT_ADMIN o MANAGER
enviaba un mensaje en una conversación de otro asesor, fallaba 412 aunque
el asesor real sí tuviera canal.

**Fix:** Orden corregido — busca canal en este orden: `assignedAdvisorId` →
`assignedToId` → `userId` (caller). Usa el primer match. Esto refleja la
intención real: el cliente espera ver el mensaje desde el número de SU asesor.

**Verificación:** TENANT_ADMIN envía mensaje en conv asignada a Juan Pérez
→ 201, persistido con externalId correcto.

### Bug Q-4 (MEDIO) — Componentes legacy ignoraban el error 412 del nuevo endpoint
**Síntoma:** `ConversationDetailPage.tsx` y `dashboard/ContactCenterView.tsx`
hacían `await api.post(...)` sin try/catch visible. Cuando el endpoint nuevo
devuelve 412 (sin canal asignado) o 422 (gateway rechazó), el mensaje
desaparecía del input pero el usuario no veía error — UX silenciosamente
regresiva vs el comportamiento previo (que escribía directo a Message).

**Fix:** Agregado try/catch con alert tipado por código de error. 412 muestra
"el asesor no tiene canal WhatsApp configurado". 422 muestra "gateway rechazó
el envío".

**Verificación:** typecheck OK. Pasada visual queda como acción del usuario.

---

## Backend QA — 16 escenarios probados con curl

### Healthchecks
- ✅ `GET /api/health` → 200 `{status:"ok"}`
- ✅ `GET /api/health/whatsapp` con adapter=in-memory → 200 `{ok:true, adapter:"in-memory"}`

### Auth + tenant context
- ✅ `POST /api/auth/login` admin@goopcar.com → JWT 207 chars
- ✅ `POST /api/auth/login` demo@autosummit.cl → JWT 251 chars + roleInTenant=TENANT_ADMIN

### Templates (HU-008)
- ✅ `GET /api/app/templates` → 17 templates (12 legacy + 5 nuevas Sprint 1)
- ✅ Las 5 nuevas tienen variables correctas (`nombre_cliente`, `sucursal`,
  `fecha_hora_cita`, `patente`)

### AdvisorChannel CRUD (HU-005)
- ✅ `GET /api/app/advisors/:id/channel` (sin asignar) → null
- ✅ `POST /api/app/advisors/:id/channel` → 201 con cuid + phoneE164 normalizado
- ✅ `GET /api/app/advisors/:id/channel` (asignado) → datos correctos
- ✅ `POST` con phone inválido → 400 con error tipado
- ✅ `POST` con phone duplicado en tenant → 409 con mensaje claro
- ✅ `GET /api/app/advisors-channels` (lista) → 1 canal Juan Pérez con advisor name

### Webhook /inbound (HU-003)
- ✅ T1 — Bearer válido + DID asignado → 200, persistInbound completo
- ✅ T2 — Bearer inválido → 401, NO persiste
- ✅ T3 — Sin Authorization header → 401, NO persiste
- ✅ T4 — Con `id` externo → 200, persistido con externalMessageId
- ✅ T5 — Repetir T4 (idempotencia) → 200, NO duplica (DB confirma 1 sola fila)
- ✅ T6 — DID huérfano (no mapea tenant) → 200 `{ok:true, note:"received but not actionable"}`, NO persiste

### POST /messages outbound (HU-007 backend)
- ✅ T7 — Free text con conv asignada a Juan Pérez (que tiene canal) → 201
  con `externalId` UUID, `provider:"in-memory"`, status SENT
- ✅ T8 — Con templateId (confirmacion_cita) → 201, persistido con templateId
- ✅ T9 — Conv huérfana sin asesor con canal → 412 con mensaje claro

### Webhook /notifications (HU-003)
- ✅ T10 — Evento DELIVERY → processed=1, status sube a DELIVERED en DB
- ✅ T11 — Evento READ → processed=1, status sube a READ + `readAt` poblado
- ✅ T12 — Batch [SENT, DELIVERY] → processed=2 sobre el mismo mensaje
- ✅ T13 — Bearer inválido → 401, NO procesa
- ✅ T14 — DELIVERY tardío sobre msg ya READ → processed=1 pero NO regresa
  estado (verificado en DB: status sigue READ)
- ✅ T15 — Evento SESSION → processed=1, registra evento (windowExpiresAt
  sólo se actualiza si `WHATSAPP_SESSION_STRATEGY=event`, default es
  `inbound_timestamp`)
- ✅ T16 — Evento INVALID con errors[] → processed=1, status pasa a FAILED

### Estado final DB (verificado con psql)
```
INBOUND   | DELIVERED | "Hola"
INBOUND   | DELIVERED | "con ID" (idempotente — única fila pese a 2 POSTs)
OUTBOUND  | READ      | "Confirmamos tu cita..."   readAt+deliveredAt poblados
OUTBOUND  | FAILED    | "Hola QA, te confirmamos..." (después del evento INVALID)

Conversation: 1 fila, externalProvider='in-memory', windowExpiresAt = +24h
Customer: "QA" creado con source='WHATSAPP_INBOUND', phone normalizado
```

### Bandeja sin regresión
- ✅ `GET /api/app/conversations?period=30d` → 592 conversaciones (toda la
  data del seed sigue cargando, NO hay error 500 ni payload roto)

---

## Frontend QA — pendiente pasada visual

**Servidor:** http://localhost:3000 (vite v6.4.1, build OK, sin errores
de transpile)

**Pre-cargado:**
- Login admin: `admin@goopcar.com` / `admin123`
- Login tenant: `demo@autosummit.cl` / `demo123` (TENANT_ADMIN)
- Login asesor: `jperez@autosummit.cl` / `demo123` (probable, default seed)
- AdvisorChannel ya asignado a Juan Pérez (`+56987654321`)
- 1 conversación QA Cliente con 2 inbound DELIVERED + 1 outbound READ
  + 1 outbound FAILED

**Checklist visual sugerido (5 min):**

1. **Bandeja sin regresión** — abrir http://localhost:3000 → login
   `demo@autosummit.cl` / `demo123` → debe ir a `/app/dashboard`. Ir a
   `/app/conversations` → ver bandeja con ~600 conversaciones cargando.
   Sin crash en consola browser.

2. **AdvisorChannel UI** — ir a `/app/asesores`. Como TENANT_ADMIN,
   debe verse columna "Canal WhatsApp" con `+56987654321` para Juan Pérez.
   Click ⚙️ en otro asesor sin canal → drawer abre → asignar
   `+56998765432` → click "Asignar canal" → debe aparecer badge verde.

3. **Conversación QA + selector plantillas** — buscar "QA" en la bandeja
   → click la conversación → InlineChatPanel abre. Ver:
   - 2 mensajes INBOUND (gris, alineados izquierda) sin icono status
   - 1 OUTBOUND READ (verde con doble check verde a la derecha del timestamp)
   - 1 OUTBOUND FAILED (fondo rojo, icono AlertCircle)

4. **Enviar plantilla** — click botón 📄 Plantillas → modal abre con
   las 5 plantillas Sprint 1. Click `confirmacion_cita` → formulario
   con 4 inputs aparece. Llenar las 4 variables → ver preview en tiempo
   real → click "Enviar plantilla" → modal cierra → mensaje aparece
   optimistic con icono Clock → en ~200ms cambia a Check (SENT).

5. **Enviar texto libre** — escribir mensaje en el input → Enter o click
   Send → mismo flow optimistic.

6. **Status dinámico via curl** — desde otra terminal:
   ```bash
   EXT=$(curl -sS http://localhost:4000/api/app/conversations/<conv-id> \
     -H "Authorization: Bearer $TENANT_TOK" \
     | jq -r '.messages[-1].id')
   # Necesitamos externalMessageId, no id. Tomar del DB:
   psql -c "SELECT \"externalMessageId\" FROM ..." # del último outbound
   curl -X POST http://localhost:4000/api/webhooks/chattigo/notifications \
     -H "Authorization: Bearer qa-status-token-32-chars-bbbbbbbbb" \
     -d '{"id":"<EXT>","type":"READ"}'
   ```
   En el browser, recargar: el icono de ese mensaje debe cambiar a
   doble check verde (READ).

---

## Veredicto

**Backend Sprint 1: GARANTÍA E2E VERDE.** 90 tests automatizados +
16 escenarios curl con datos reales en DB. 4 bugs encontrados y
arreglados. Persistencia, idempotencia, ventana 24h, status updates,
auth, multi-tenant, todos verificados.

**Frontend Sprint 1: GARANTÍA TÉCNICA VERDE, pasada visual pendiente.**
Typecheck OK en `apps/web`, vite levanta sin errores, todas las APIs
que consume responden correctamente. Riesgo de regresión en componentes
legacy mitigado con manejo de error visible.

**Recomendación:** ejecutar el checklist visual de 5 min antes de
grabar el Loom. Si todo OK, **Sprint 1 cerrado con sello E2E**.

---

*Reporte generado durante la sesión de QA E2E del 2026-05-15.*
