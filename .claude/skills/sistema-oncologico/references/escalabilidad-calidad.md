# Escalabilidad, seguridad, operación y calidad

## Escalabilidad (qué ya está y qué falta)

| Aspecto | Estado | Acción |
|---|---|---|
| Backend sin estado (JWT) | ✔ | Permite varias réplicas detrás del proxy |
| Refresh token persistido | ✔ | — |
| Rate limit | ✔ en memoria (`RateLimitFilter`) | Mover a Redis (Bucket4j + Redis) para que funcione con varias réplicas |
| Jobs programados | ✘ | ShedLock para que corran una vez aunque haya N réplicas |
| Caché | Redis desplegado pero sin uso | `@Cacheable` en catálogos (médicos, especialidades) y en indicadores del dashboard (TTL 5 min, invalidar al cerrar mediciones) |
| Comunicación con n8n | Llamadas directas "dispara y olvida" | Patrón **outbox**: tabla `evento_saliente` escrita en la misma transacción y publicada por un job; nada se pierde si n8n está caído |
| Consultas de indicadores | Conteos por JPA | Índices compuestos (ver migraciones), consultas agregadas en SQL; si crece, vistas materializadas refrescadas por job |
| Frontend | App única | Dos apps (portal / intranet) servidas por Nginx como estáticos con caché agresiva de assets con hash |
| Archivos (documentos del paciente) | URL en BD | Almacenamiento de objetos S3-compatible (MinIO en local) con URLs firmadas |

## Seguridad y protección de datos (Ley 29733)

- Secretos solo en variables de entorno / gestor de secretos. `.env` real fuera del repositorio (ya en `.gitignore`); rotar el `JWT_SECRET` y la contraseña del admin semilla antes de cualquier despliegue.
- Consentimiento informado registrado en `participante_estudio` antes de incluir a un paciente en la muestra.
- Minimización: exportaciones con código de participante, mensajes de Telegram sin datos clínicos, prompts a Gemini sin DNI ni diagnóstico (solo el texto del usuario + contexto mínimo).
- Auditoría inmutable (ya existe): extenderla a exportaciones de datos, cambios de fase, correcciones de mediciones y vinculación de Telegram.
- Logs sin datos personales: enmascarar DNI, teléfono y correo en el `logback` (patrón de reemplazo) y nunca loguear cuerpos de request de pacientes.
- HTTPS obligatorio en producción, cabeceras de seguridad (CSP, HSTS, X-Frame-Options) en Nginx, CORS restringido a los dominios del portal y la intranet.
- Copias de seguridad diarias de PostgreSQL (`pg_dump` programado + retención de 30 días) y prueba de restauración documentada.
- Contraseñas: política mínima ya validada; bloqueo por intentos fallidos ya existe.

## Observabilidad

- Actuator ya expone `health` y `metrics`. Añadir `micrometer-registry-prometheus` y exponer `/actuator/prometheus` solo en red interna.
- Métricas de negocio con Micrometer: `onco_consultas_total{resultado,canal}`, `onco_recordatorios_total{estado}`, `onco_registro_duracion_seconds` (histograma). Útiles para ver los indicadores en tiempo casi real y detectar caídas de n8n o Gemini.
- Logs estructurados JSON (Spring Boot 3.4+ `logging.structured.format.console=ecs`) con `traceId`.
- Opcional: Prometheus + Grafana en `docker-compose.monitoring.yml`.

## Despliegue

```
Internet ──HTTPS──> Nginx/Caddy (TLS, cabeceras)
                     ├── portal.<dominio>      → estáticos portal
                     ├── intranet.<dominio>    → estáticos intranet (restringible por IP)
                     ├── api.<dominio>         → backend (1..N réplicas)
                     └── n8n.<dominio>         → n8n (webhook público solo /webhook/*, editor protegido)
Red interna: PostgreSQL (oncologia + n8n), Redis, n8n worker(s)
```

- `docker-compose.yml` para desarrollo y `docker-compose.prod.yml` con imágenes versionadas, healthchecks, límites de recursos y sin puertos de BD expuestos.
- Imágenes del backend con build multi-stage y JRE 21 (Temurin/Distroless), usuario no root.
- Perfiles Spring: `dev`, `test`, `prod`.

## CI (GitHub Actions)

1. Backend: `./mvnw verify` (tests + JaCoCo + ArchUnit).
2. Frontend: `npm ci && npm run lint && npm test && npm run build` para portal e intranet.
3. Build de imágenes Docker en `main` y escaneo de dependencias (Dependabot / `mvn dependency-check` / `npm audit`).
4. Validación de que los JSON de `n8n/workflows/` son JSON válido.

Requiere que el proyecto esté bajo **git** (hoy no lo está): primer paso del plan.

## Definición de "terminado" para cualquier tarea

- [ ] Compila y pasan `./mvnw test` y `npm test` (si no se pudo ejecutar algo, se dice explícitamente).
- [ ] Si toca un indicador: test de la fórmula con casos borde y test de integración del conteo.
- [ ] Migración Flyway nueva (no se editan las aplicadas) y `ddl-auto: validate` arranca sin errores.
- [ ] Endpoints documentados en OpenAPI (anotaciones `@Operation` donde no sea obvio) y protegidos por rol.
- [ ] Eventos de auditoría para acciones que cambian datos de pacientes o mediciones.
- [ ] UI con tokens del tema, modo oscuro correcto, usable en móvil (portal) y con estados de carga/vacío/error.
- [ ] Sin datos personales en logs, mensajes de Telegram ni prompts.
- [ ] Tarea marcada en `docs/PLAN_IMPLEMENTACION.md`; si cambia la operacionalización de un indicador, anotarlo en la sección "Impacto en la tesis".
