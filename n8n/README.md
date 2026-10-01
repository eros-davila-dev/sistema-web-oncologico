# Flujos de n8n — Recordatorios de citas (sección 15, flujo 2)

Implementa el flujo que ataca directamente el problema central del sistema (sección 1 del prompt): **reducir el ausentismo mediante recordatorios automáticos**. Cada día, a las 8:00 a. m., el workflow busca las citas programadas dentro de las próximas 24–48 horas y envía un recordatorio por WhatsApp al paciente, dejando registro del resultado del envío en el backend.

## Archivos

| Archivo | Qué hace |
|---|---|
| `workflows/obtener-token-servicio.json` | Sub-workflow reutilizable: hace login contra el backend con la cuenta de servicio y devuelve el `accessToken`. Pensado para que los futuros flujos (chat conversacional, agendamiento) lo reutilicen también, tal como pide la sección 15. |
| `workflows/recordatorio-citas-whatsapp.json` | Workflow principal: cron diario → token → `GET /citas/proximas` → por cada cita, obtiene el teléfono del paciente → envía WhatsApp → registra el resultado con `POST /notificaciones/callback`. |
| `workflows/error-workflow.json` | Captura fallos de cualquier workflow y deja los datos listos para notificar al equipo técnico (sección 15: "nunca fallar en silencio"). El canal final (email/Slack/Telegram) queda como placeholder porque depende de qué use la fundación. |

Estos JSON fueron escritos a mano siguiendo el esquema de exportación de n8n; no se probaron contra una instancia en vivo en este entorno (no había Docker disponible). Al importarlos, abre cada nodo una vez para confirmar que los parámetros se vean bien antes de activar el workflow — es una verificación de 2 minutos y evita sorpresas si tu versión de n8n difiere ligeramente.

## Requisitos previos

1. El sistema corriendo (`docker compose up`), con al menos un login exitoso como admin.
2. Una cuenta de WhatsApp Business en Meta for Developers, con:
   - Un **Phone Number ID** y un **Access Token** (Meta Cloud API).
   - Una plantilla de mensaje aprobada llamada `recordatorio_cita`, con 3 variables de cuerpo: `{{1}}` nombre del paciente, `{{2}}` fecha, `{{3}}` hora. Meta exige plantillas pre-aprobadas para mensajes que la empresa inicia (no es una limitación del sistema, es una regla de WhatsApp Business API).
   - Alternativa: si la fundación prefiere Twilio (también contemplado en la sección 4 del prompt), reemplaza el nodo "Enviar recordatorio WhatsApp" por la llamada equivalente a la API de Twilio; el resto del flujo no cambia.

## Paso 1 — Crear la cuenta de servicio

Los workflows llaman a la API igual que cualquier usuario (sección 5: "n8n... consume la propia API REST del backend, autenticada con un token de servicio"). Créala desde el frontend (**Usuarios y roles**, solo visible como ADMIN) o vía API:

```bash
curl -X POST http://localhost:8080/api/v1/usuarios \
  -H "Authorization: Bearer <tu-token-de-admin>" \
  -H "Content-Type: application/json" \
  -d '{"nombres":"Servicio n8n","email":"n8n@threepartners.org","password":"<la-misma-de-ONCOLOGIA_SERVICE_PASSWORD>","rol":"RECEPCIONISTA"}'
```

El email/password deben coincidir con `ONCOLOGIA_SERVICE_EMAIL` / `ONCOLOGIA_SERVICE_PASSWORD` en el `.env` de la raíz (ya generados con un valor por defecto; cámbialos si quieres).

## Paso 2 — Completar las variables de entorno

En el `.env` de la raíz, completa (o confirma):

```
ONCOLOGIA_SERVICE_EMAIL=n8n@threepartners.org
ONCOLOGIA_SERVICE_PASSWORD=...
WHATSAPP_PHONE_NUMBER_ID=...
WHATSAPP_ACCESS_TOKEN=...
```

`docker-compose.yml` ya las pasa al contenedor de n8n. Si corres n8n fuera de Docker, expórtalas en el entorno donde corra el proceso de n8n.

## Paso 3 — Importar los workflows

En la UI de n8n (**Workflows → Import from File**), en este orden:

1. `obtener-token-servicio.json`
2. `recordatorio-citas-whatsapp.json`
3. `error-workflow.json`

Al importar el workflow principal, abre el nodo **"Obtener token de servicio"**: si el desplegable del sub-workflow no lo resuelve automáticamente (puede pasar porque el ID interno cambia entre instancias de n8n), selecciona manualmente "Oncologia - Obtener Token de Servicio" de la lista — es un ajuste de una sola vez.

## Paso 4 — Configurar el Error Workflow

En el workflow **"Oncologia - Recordatorio de Citas (WhatsApp)"** → Settings (ícono de engranaje) → **Error Workflow** → selecciona "Oncologia - Error Workflow". Antes de eso, abre `error-workflow.json` y reemplaza el nodo placeholder "Conectar canal de alertas..." por el nodo real que la fundación quiera usar (Send Email, Slack, Telegram, etc.), usando los campos que ya vienen formateados (`workflow`, `nodoQueFallo`, `mensajeError`, `urlEjecucion`).

## Paso 5 — Probar y activar

1. Con el workflow principal abierto, click en **"Execute Workflow"** para probarlo manualmente contra una cita real (crea una cita de prueba con fecha dentro de las próximas 24-48h primero).
2. Verifica que quedó el registro consultando directamente la tabla `notificacion` (no hay un endpoint GET de listado; el backend solo expone el `POST /notificaciones/callback` que este workflow usa para escribir el resultado).
3. Activa el workflow (toggle **Active** arriba a la derecha). A partir de ahí corre solo, todos los días a las 8:00 a. m.

## Qué falta más allá de esto

Los otros dos flujos de la sección 15 (chat conversacional y agendamiento vía IA) **ya no viven en n8n**: se implementaron directamente en el backend (`ChatbotOrquestadorUseCase` + `GeminiRestClientAdapter`, ver el README raíz, sección "Chatbot con Gemini"). n8n ya no orquesta la conversación; `ChatbotWebhookController` (`POST /api/v1/chatbot/webhook/conversaciones`) queda disponible solo como bitácora si en el futuro se agrega otro canal externo (ej. WhatsApp conversacional) que necesite reportar interacciones desde fuera del propio backend.

## Pendiente: workflows de correo del portal de autoservicio de pacientes

El backend ya llama a dos webhooks de n8n para el flujo de registro de pacientes (sección 7), pero **los workflows en sí todavía no existen** — requieren credenciales SMTP/proveedor de correo (SendGrid, SES, Gmail API, etc.) que dependen de la fundación, igual que WhatsApp arriba. No se fabricó un workflow falso: se documenta el contrato exacto para que se puedan crear siguiendo el mismo patrón que `recordatorio-citas-whatsapp.json`.

| Ruta | Cuándo se dispara | Payload (JSON) |
|---|---|---|
| `POST /webhook/notificaciones/verificacion-email` | Al registrarse una nueva cuenta de paciente (`POST /api/v1/auth/registro`) | `{"email": "...", "nombre": "...", "urlVerificacion": "http://.../auth/verificar-email?token=..."}` |
| `POST /webhook/notificaciones/recuperar-password` | Al solicitar recuperación de contraseña (`POST /api/v1/auth/recuperar-password`) | `{"email": "...", "nombre": "...", "urlRestablecer": "http://.../auth/restablecer-password?token=..."}` |

Ambas llamadas llevan el header `X-Webhook-Secret` con el mismo `N8N_WEBHOOK_SECRET` usado en el resto de la integración (verificar con `WebhookSecretValidator` si se agrega un endpoint que reciba el resultado del envío, siguiendo el mismo patrón que `NotificacionController`). El token dentro de la URL es de un solo uso y expira (24h para verificación, 1h para reset): el workflow solo necesita enviar el correo con el enlace, no validar nada.

Si el envío falla silenciosamente en el backend (`N8nNotificationAdapter.dispararWorkflow` solo loguea el error y no reintenta), considera agregar reintentos dentro del propio workflow de n8n en vez de en el backend.
