# Sistema Web Oncologico — Fundacion Three Partners

Implementacion del prompt maestro (`prompt-sistema-oncologico.md`): gestion de pacientes oncologicos, citas, tratamientos, auditoria y chatbot, con arquitectura hexagonal en el backend y Angular + Tailwind en el frontend.

## Estructura

```
backend/    Spring Boot 3.5.16 (Java 21) — arquitectura hexagonal, JWT, auditoria, Flyway
frontend/   Angular 21 standalone + Tailwind CSS 4
n8n/        Workflows de recordatorios por WhatsApp (seccion 15, flujo 2) — ver n8n/README.md
docker-compose.yml   Orquesta backend, frontend, PostgreSQL, Redis y n8n
```

## Levantar todo con Docker Compose

```bash
cp .env.example .env   # ajustar secretos antes de produccion
docker compose up --build
```

- Frontend: http://localhost:4200
- Backend (Swagger UI): http://localhost:8080/swagger-ui.html
- n8n: http://localhost:5678 (usuario/clave definidos en `.env`)

Usuario administrador inicial (creado por la migracion `V2__seed_usuario_admin.sql`):

- Email: `admin@threepartners.org`
- Password: `Admin123!`

**Cambiar esta contraseña inmediatamente en un entorno real.**

## Desarrollo local sin Docker

### Backend

Requiere JDK 21 y una instancia de PostgreSQL 16 corriendo localmente (o vía `docker compose up postgres`).

```bash
cd backend
cp .env.example .env   # completar con valores reales; nunca se commitea
./mvnw spring-boot:run
```

`spring-dotenv` carga `backend/.env` automáticamente al iniciar (sin Docker no hace falta exportar las variables a mano). El archivo `.env` está excluido en `.gitignore`; solo `.env.example` (sin secretos reales) se versiona.

### Frontend

Requiere Node.js 20+.

```bash
cd frontend
npm install
npm start   # http://localhost:4200, apunta a http://localhost:8080 en desarrollo
```

## Pruebas

```bash
cd backend && ./mvnw test    # unitarias + integracion (Testcontainers, requiere Docker)
cd frontend && npm test      # Vitest
```

## Que cubre esta implementacion

Ver `prompt-sistema-oncologico.md` para el detalle completo del prompt original. En resumen:

- **Backend**: arquitectura hexagonal (`domain` / `application` / `infrastructure`), autenticacion JWT con refresh token persistido y revocable (logout real, rotacion en cada `/auth/refresh`), bloqueo de cuenta tras intentos fallidos, autorizacion por rol via `@PreAuthorize`, auditoria inmutable basada en eventos de dominio asincronos, modulos de pacientes/citas/tratamientos con validacion exhaustiva, portal de autoservicio de pacientes (registro, verificacion de correo, recuperacion de contrasena, completar ficha clinica), chatbot con Gemini embebido en el backend (ver mas abajo), puerto generico de dispositivos IoT (con adaptador REST y MQTT deshabilitado por defecto), webhooks para n8n y dispositivos autenticados con secreto compartido, catalogo de errores uniforme, paginacion en todos los listados, OpenAPI/Swagger.
- **Frontend**: componentes standalone con signals, interceptores JWT con renovacion automatica, guards de autenticacion y rol (incluido el portal de paciente: `mis-citas`, `mi-perfil`, `completar-perfil`), formularios reactivos con validacion en tres niveles (formato, unicidad asincrona contra la API, errores del backend), componentes UI reutilizables (`shared/ui`, `shared/components`), tema claro/oscuro persistente, panel de indicadores con ApexCharts, widget de chatbot embebido conectado al backend propio.
- **Fuera de alcance** (segun el propio prompt, seccion 3): drivers reales de dispositivos, HCE con HL7/FHIR, facturacion, telemedicina. El adaptador MQTT y el webhook de dispositivos quedan implementados como contrato funcional, no contra hardware real.

### Chatbot con Gemini (seccion 13)

El widget del navegador habla con `POST /api/v1/chatbot/mensaje` (backend propio), nunca directo con Gemini ni con n8n:

```
Angular (widget) -> Backend (ChatbotController) -> GeminiPort (interpreta intencion)
                                                  -> Backend decide y ejecuta con los
                                                     mismos casos de uso que el resto
                                                     del sistema (agendar/reprogramar/
                                                     cancelar/confirmar/consultar citas)
                                                  -> PostgreSQL
```

Gemini solo clasifica el mensaje en una intencion cerrada (`Intencion.java`) y extrae entidades (fecha, hora, especialidad, motivo) devolviendo JSON estructurado (`generationConfig.responseSchema`); nunca ejecuta acciones ni toca la base de datos. El endpoint es publico (permite preguntas generales de un visitante anonimo), pero si la peticion trae sesion iniciada, el chatbot identifica al paciente y puede gestionar sus propias citas.

Para activarlo, consigue una clave gratuita en <https://aistudio.google.com/apikey> y define `GEMINI_API_KEY` en `.env` (`GEMINI_MODEL` es opcional, por defecto `gemini-3.6-flash`). **Sin la clave el chatbot sigue funcionando** y responde en modo degradado ("no puedo procesar tu mensaje ahora, un miembro del equipo te ayudara"), nunca se cae la peticion ni se expone un error tecnico.

- **n8n**: el flujo de recordatorios por WhatsApp (seccion 15, flujo 2 — el que ataca directamente el ausentismo) esta implementado como workflows exportados en `n8n/workflows/`, listos para importar; ver `n8n/README.md` para la puesta en marcha completa (cuenta de servicio, credenciales de WhatsApp Business, plantilla de mensaje). Los workflows de envio de correo del portal de autoservicio (verificacion de cuenta, recuperacion de contrasena) tampoco estan incluidos: requieren credenciales SMTP/proveedor de correo propias de la fundacion; el contrato exacto que el backend ya invoca esta documentado en `n8n/README.md`.

### Registro de pacientes: apagado hasta tener SMTP (seccion 7)

`REGISTRO_PACIENTES_HABILITADO` (`.env`) esta en `false` por defecto. Mientras lo este, `POST /api/v1/auth/registro` responde 503 con un mensaje claro y **no crea ninguna fila en la base de datos** — evita cuentas huerfanas en `PENDIENTE_VERIFICACION` que nunca podrian activarse sin el correo de verificacion. El chatbot tambien lo respeta: si alguien le pide registrarse, responde que el registro estara disponible proximamente en vez de mandarlo a un formulario que va a fallar.

Cuando el workflow de correo (arriba) este listo, pon `REGISTRO_PACIENTES_HABILITADO=true` en `.env` y reinicia el backend — no requiere ningun otro cambio.
