# Sprint 1 — Script de demo (Loom 3-5 min)

**Audiencia:** Comité Goopcar (socios + Óscar + Diego). Asumimos que el viewer
ya conoce el contexto de la épica (no explicamos qué es WhatsApp asesor-cliente).

**Pre-requisitos del ambiente:**
- `WHATSAPP_GATEWAY_ADAPTER=in-memory` (NO `chattigo` mientras KAM 7 esté abierto).
- Migration aplicada: `pnpm --filter @goopcar/api db:migrate` o `db:push`.
- Seed corrido al menos una vez: `pnpm --filter @goopcar/api db:seed`.
- Vite dev del frontend corriendo y proxy a la API del VPS funcionando.

---

## Guion (4-5 min)

### 1. (0:00 — 0:30) Setup y contexto
> "Vamos a recorrer el cierre de Sprint 1 — la capa de integración WhatsApp
> asesor-cliente. El sistema corre con un adapter `in-memory` para no impactar
> la cuenta Chattigo productiva mientras destrabamos credenciales sandbox.
> Lo que ven es funcionalmente idéntico al adapter real."

### 2. (0:30 — 1:00) Healthcheck del gateway
- Mostrar `GET https://api.goopcar.tech/api/health/whatsapp` en navegador o Postman.
- Respuesta esperada con adapter=in-memory: `{ ok: true, adapter: 'in-memory' }`.
- Mencionar: "cuando el comité apruebe el switch a Chattigo real, este endpoint
  hace `POST /login` contra la API de Chattigo y reporta credenciales OK o no
  — sin enviar ningún mensaje".

### 3. (1:00 — 2:00) Configurar canal del asesor (HU-005)
- Login como TENANT_ADMIN.
- Ir a `/app/asesores` (vista gerencial).
- Mostrar la nueva columna "Canal WhatsApp" en la tabla de asesores.
- Click en el icono ⚙️ del primer asesor sin canal → abre drawer.
- Llenar `phoneE164` con `+56987654321` → "Asignar canal".
- Mostrar el badge verde aparecer en la tabla.
- Mencionar: "el partial unique a nivel DB garantiza que ningún asesor tenga
  dos canales activos al mismo tiempo".

### 4. (2:00 — 3:30) Composer + plantillas (HU-006/HU-007/HU-008)
- Login como ASESOR (el asesor que acabamos de configurar canal).
- Ir a `/app/conversations`.
- Mostrar la bandeja con la lista de conversaciones (KPI personales arriba).
- Click en una conversación con `windowExpiresAt` futuro.
- En el composer, click en el botón 📄 "Plantillas":
  - Seleccionar `confirmacion_cita`.
  - Llenar las 4 variables (nombre_cliente, sucursal, fecha_hora_cita, patente).
  - Mostrar el preview renderizado.
  - Click "Enviar plantilla".
- Mostrar la burbuja aparecer en el chat con icono Clock (sending) → Check (sent).
- En el composer, escribir un mensaje de free text:
  - Mostrar que aparece optimisticamente con icono Clock.
  - Esperar a que pase a Check (sent).

### 5. (3:30 — 4:30) Webhook inbound simulado
- Mostrar terminal con un `curl` al webhook inbound (en otro tab):
  ```bash
  curl -X POST https://api.goopcar.tech/api/webhooks/chattigo/inbound \
    -H "Authorization: Bearer $CHATTIGO_WEBHOOK_INBOUND_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"did":"56987654321","msisdn":"56999111222","type":"text",
         "channel":"WHATSAPP","content":"Hola, sí confirmo mi cita"}'
  ```
- Recargar la bandeja: la conversación correspondiente debe mostrar el mensaje
  inbound nuevo + badge de "no leído" + `windowExpiresAt` renovado a +24h.

### 6. (4:30 — 5:00) Cierre
> "Lo que vimos: bandeja funcional con envío de plantillas + texto libre +
> recepción de mensajes + persistencia + ventana 24h gestionada. Todo el código
> de Chattigo está implementado y testeado (85 tests pasando). El switch a
> producción real es un cambio de env var cuando comercial destrabe el tema
> de credenciales (KAM 7). Sprint 2 arranca el lunes con el motor automático
> de journey (confirmación, recordatorio, no-show, NPS)".

---

## Checklist pre-grabación

- [ ] `pnpm test` corriendo verde en local (`apps/api`).
- [ ] Frontend `pnpm dev:web` levantado, proxy al VPS OK.
- [ ] Loom abierto y configurado para grabar pestaña + audio.
- [ ] Tener listo el `curl` del webhook simulado (copy/paste rápido).
- [ ] Cerrar tabs irrelevantes; sólo la app, terminal y healthcheck visible.
- [ ] Probar el guion completo una vez "en seco" antes de grabar.

## Grabación

Loom → 720p mínimo → audio del mic activo → 4-5 min máximo.
Subir a Drive carpeta "Desarrollo Goopcartech" como `Sprint1_demo_2026-05-15.mp4`.
Pegar el link en el grupo WhatsApp "Desarrollo Goopcartech" + en este reporte.
