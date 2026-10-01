# Plan de implementación — Sistema web con chatbot para la gestión de pacientes

**Tesis**: *Sistema web con chatbot para la gestión de pacientes en una fundación oncológica privada de Lima, 2026* (Arnao Fretel, Dávila Anchante — UCV).
**Stack**: Java 21 · Spring Boot 3.5 · PostgreSQL 16 · Redis · Angular 21 · Tailwind CSS 4 · n8n · Telegram Bot API · Gemini.
**Guía técnica**: skill del proyecto en `.claude/skills/sistema-oncologico/` (leer `references/indicadores-tesis.md` antes de tocar cualquier métrica).

---

## 1. Objetivo del sistema, en una frase

Reducir el **tiempo promedio de registro (TPR)**, reducir la **tasa de ausentismo (TNS, línea base 65 %)** y aumentar el **nivel de consultas atendidas (NCA)**, y **medir los tres de forma automática, auditable y pareada por paciente** para contrastar H1, H2 y H3 con Wilcoxon (pretest vs. postest, n = 30).

| Indicador | Fórmula | Fuente de datos en el sistema |
|---|---|---|
| TPR (min) | Σ TRC / NCR | `medicion_registro` (sellado por el servidor; pretest por ficha manual) |
| TNS (%) | NI / (NI + NCC) × 100 | `cita` con desenlace `NO_ASISTIO` / `ATENDIDA` |
| NCA (%) | CA / TCR × 100 | `consulta` (chatbot web, Telegram; pretest: WhatsApp/llamadas por ficha manual) |

## 2. Diagnóstico del avance actual

**Lo que ya está bien construido** (se conserva):
- Backend hexagonal con JWT + refresh rotativo, bloqueo de cuenta, roles con `@PreAuthorize`, auditoría inmutable (trigger en BD), Flyway, paginación, OpenAPI, manejo de errores uniforme.
- Módulos de pacientes, citas (agendar, reprogramar, cancelar, confirmar, atendida, no asistió), tratamientos, documentos, usuarios.
- Portal de autoservicio del paciente (registro, verificación, recuperación, completar perfil, mis citas).
- Chatbot con Gemini: catálogo cerrado de intenciones, el backend ejecuta las acciones, modo degradado sin API key, historial por sesión.
- Frontend Angular standalone con signals, guards, interceptores, sistema de diseño con tokens Tailwind 4, modo oscuro, dashboard con ApexCharts.
- n8n: workflow de recordatorios por WhatsApp y sub-workflow de token.

**Brechas críticas para la tesis** (sin corregirlas, los resultados no son defendibles):

| # | Brecha | Impacto | Dónde |
|---|---|---|---|
| B1 | TPR se calcula en el navegador (`Date.now()`) y lo envía el cliente | Dato manipulable; no es "timestamp generado por el sistema" como declara el instrumento | `paciente-form.ts`, `completar-perfil.ts`, `PacienteRequestDto` |
| B2 | TPR se promedia sobre toda la tabla, sin periodo ni fase | No distingue pretest/postest | `PacienteJpaRepository.promedioTiempoRegistroSegundos` |
| B3 | TPR mide alta de paciente (1 vez por paciente) y la fórmula habla de citas registradas | Sin pares por paciente → no se puede aplicar Wilcoxon | Operacionalización |
| B4 | TNS usa como denominador **todas** las citas (incluye canceladas y futuras) | Subestima el ausentismo | `IndicadoresRepositoryAdapter` |
| B5 | "Consultas atendidas" = citas `ATENDIDA` | Mide otra cosa; H3 no tiene datos | `IndicadoresRepositoryAdapter` |
| B6 | No existe el concepto de consulta ni si fue resuelta | NCA imposible de calcular | Chatbot |
| B7 | No hay fases pretest/postest ni muestra de participantes | Indicadores no atribuibles a la muestra | — |
| B8 | No hay forma de cargar los datos del pretest en el sistema | Pretest y postest saldrían de fuentes distintas | — |
| B9 | No hay exportación pareada para SPSS | Análisis manual propenso a errores | — |

**Brechas técnicas**:
- El proyecto **no está bajo git**.
- `docker-compose.yml`: n8n en `EXECUTIONS_MODE=queue` sin worker ni PostgreSQL propio (los workflows no se ejecutarían); imagen `latest`; `N8N_BASIC_AUTH_*` obsoleto.
- Canal de recordatorios en WhatsApp (requiere plantillas aprobadas por Meta); se reemplaza por **Telegram**.
- Una sola app Angular para pacientes y personal.
- Sin ShedLock, sin outbox hacia n8n, rate limit en memoria.

## 3. Arquitectura objetivo

```
                ┌────────────── Portal (Angular) ──────────────┐   ┌──── Intranet (Angular) ────┐
 Paciente ────▶ │ inicio · mis citas · vincular Telegram · chat │   │ dashboard · agenda · bandeja│ ◀── Personal
                └───────────────────────┬──────────────────────┘   │ estudio · pacientes · citas │
                                        │ HTTPS /api/v1             └──────────────┬─────────────┘
                                        ▼                                          ▼
                     ┌──────────────────────────── Backend Spring Boot (hexagonal) ─────────────────────────┐
                     │ pacientes · citas · agenda · chatbot+consultas · estudio/indicadores · recordatorios │
                     │ auditoría · exportación SPSS · jobs (ShedLock)                                       │
                     └───────┬───────────────────────┬────────────────────────┬──────────────────────────────┘
                             │                       │ X-Webhook-Secret       │
                        PostgreSQL / Redis        n8n (router Telegram, recordatorios, alertas) ──▶ Telegram Bot API
                                                     │                                                 ▲
                                                  Gemini (solo interpretación) ◀── backend              │
 Paciente (celular) ───────────────────────────────────────────────────────────────────────────────────┘
```

## 4. Fases

Duraciones estimadas para una persona a tiempo parcial; ajústalas a tu cronograma (Tabla 4 de la tesis). **El orden importa**: la captura del pretest (Fase 1) debe estar lista lo antes posible, porque el pretest se mide *antes* de poner el sistema en producción.

### Fase 0 — Fundaciones (≈ 1 semana)
- [ ] `git init`, primer commit del estado actual, rama `main` protegida, ramas por fase.
- [ ] Corregir `docker-compose.yml`: n8n en modo `regular` (o worker + PostgreSQL para n8n), versión fija, `N8N_ENCRYPTION_KEY`, `GENERIC_TIMEZONE=America/Lima`, quitar basic auth obsoleto.
- [ ] Bean `Clock` con zona `America/Lima`; migrar timestamps nuevos a `TIMESTAMPTZ`.
- [ ] ShedLock + tabla `shedlock`.
- [ ] Test ArchUnit de capas hexagonales; JaCoCo.
- [ ] GitHub Actions: backend `verify`, frontend `test` + `build`.
- [ ] Rol `INVESTIGADOR` (y rol técnico `SERVICIO` para n8n).

**Criterio de aceptación**: `docker compose up` levanta todo; CI en verde.

### Fase 1 — Módulo de estudio y corrección de indicadores (≈ 2 semanas) — *núcleo de la tesis*
- [ ] Migración `V5__modulo_estudio_indicadores.sql` (fases, participantes, `medicion_registro`, `consulta`, columnas nuevas de `cita`, `correccion_medicion`, triggers de solo inserción).
- [ ] Dominio `estudio`: fórmulas puras TPR/TNS/NCA con tests de casos borde.
- [ ] Sesión de medición de registro: `POST /mediciones/registro`; `AgendarCitaUseCase` y `RegistrarPacienteUseCase` cierran la medición en su transacción; job de abandonadas.
- [ ] Eliminar `tiempoRegistroSegundos` del cliente y de los DTO (B1).
- [ ] Corregir TNS (B4) y reemplazar "consultas atendidas" por NCA real (B5).
- [ ] `EstadoCita.REPROGRAMADA` + `cita_origen_id`; la reprogramación cierra la cita original.
- [ ] API de fases y participantes.
- [ ] **Fichas pretest digitales** (tiempos, ausentismo, consultas) + importación CSV/XLSX de las hojas de cálculo históricas.
- [ ] `GET /estudio/indicadores` con `fase` y `alcance` (GLOBAL/MUESTRA/POR_PARTICIPANTE).

**Criterio de aceptación**: con un escenario sembrado conocido, los tres indicadores dan el valor esperado en tests de integración; el investigador puede empezar a capturar el pretest.

### Fase 2 — Agenda de recepción y desenlace de citas (≈ 1 semana) — *H2*
- [ ] `GET /citas/agenda?fecha=` y pantalla "Agenda del día" con "Llegó" / "No asistió" (registra `fecha_hora_desenlace`, `desenlace_registrado_por`).
- [ ] Bandeja de "citas pendientes de cierre" + job de cierre automático a las 48 h (`cierre_automatico = true`), corregible con auditoría.
- [ ] Medición de TPR en el formulario de nueva cita de la intranet.

**Criterio de aceptación**: ninguna cita pasada queda sin desenlace más de 48 h; el TNS del dashboard coincide con un cálculo manual.

### Fase 3 — Chatbot orientado a consultas (≈ 1,5 semanas) — *H3*
- [ ] `ejecutarAccion` devuelve un resultado tipado (`EXITO`, `INFORMATIVA`, `REQUIERE_SESION`, `ERROR_NEGOCIO`, `ESCALAR`).
- [ ] `GestorConsultaService`: abrir/continuar/cerrar consultas por sesión e intención; `conversacion_chatbot.consulta_id`.
- [ ] Valoración 👍/👎 en el widget; botón "Hablar con una persona" → `ESCALADA`.
- [ ] Bandeja de consultas en la intranet (resolver / no resuelta) + job de escaladas sin respuesta (48 h).
- [ ] Base de conocimiento de preguntas frecuentes de la fundación (horarios, requisitos, ubicación, servicios) inyectada como contexto a Gemini para que `GENERAL_QUERY` se resuelva con información real.
- [ ] Canal en el request del chatbot (`CHATBOT_WEB` / `TELEGRAM`).

**Criterio de aceptación**: cada conversación de prueba genera consultas con resultado correcto; NCA y NCA_auto visibles en el dashboard.

### Fase 4 — Telegram + n8n (≈ 2 semanas) — *H2 y H3*
- [ ] Crear el bot con @BotFather; credencial en n8n; túnel HTTPS para desarrollo.
- [ ] Migración `V6__telegram_recordatorios.sql` (`paciente.telegram_chat_id`, `recordatorio`).
- [ ] Vinculación: token de un solo uso (portal y desde la ficha del paciente en recepción, con QR) → deep link `t.me/<bot>?start=<token>`.
- [ ] `ProgramarRecordatoriosJob` (72 h / 24 h / 2 h), cancelación de recordatorios al cancelar/reprogramar.
- [ ] Endpoints `recordatorios/pendientes` (con `SKIP LOCKED`) y `resultado`; `accion-cita` (confirmar/cancelar/reprogramar con verificación de dueño).
- [ ] Workflows: `telegram-router`, `recordatorios-telegram`, `alertas-personal`; completar `error-workflow`. Exportarlos a `n8n/workflows/` y actualizar `n8n/README.md`.
- [ ] Chatbot por Telegram (texto libre → backend → respuesta + 👍/👎).
- [ ] Retirar o dejar como alternativa documentada el workflow de WhatsApp.

**Criterio de aceptación**: un paciente de prueba vincula su Telegram, recibe los recordatorios, confirma con un botón y la cita queda `CONFIRMADA`; un mensaje libre por Telegram genera una consulta contabilizada.

### Fase 5 — Frontend profesional: portal e intranet (≈ 2 semanas)
- [ ] Separar el workspace en `projects/portal`, `projects/intranet`, `projects/shared` sin cambiar comportamiento.
- [ ] Portal: inicio público, preguntas frecuentes, mis citas, vincular Telegram, chatbot con valoración; accesible y mobile-first.
- [ ] Intranet: dashboard de indicadores rediseñado (Pretest / Postest / Comparativo, n de cada indicador, "Sin datos" en vez de 0), agenda del día, bandeja de consultas, módulo de estudio.
- [ ] Cliente API generado desde OpenAPI (opcional, evita desalineación de modelos).

**Criterio de aceptación**: ambas apps compilan y se despliegan por separado; flujos principales probados en móvil y escritorio, claro y oscuro.

### Fase 6 — Exportación y análisis (≈ 1 semana)
- [ ] `GET /estudio/exportaciones/spss.xlsx` (hoja pareada por participante, detalle por indicador, diccionario de variables).
- [ ] `GET /estudio/exportaciones/fichas.xlsx` (las tres fichas del Anexo 2 llenas por fase).
- [ ] Vista comparativa con Wilcoxon preliminar (`commons-math3`), indicando que el análisis oficial es en SPSS.

**Criterio de aceptación**: el libro exportado se abre en SPSS sin transformaciones manuales.

### Fase 7 — Producción y robustez (≈ 1 semana)
- [ ] `docker-compose.prod.yml`, Nginx/Caddy con TLS y cabeceras, dominios separados.
- [ ] Rate limit en Redis, caché de catálogos e indicadores, outbox hacia n8n.
- [ ] Prometheus/métricas de negocio, logs JSON sin datos personales.
- [ ] Backups diarios de PostgreSQL + restauración probada.
- [ ] Cambiar credenciales semilla; activar `REGISTRO_PACIENTES_HABILITADO` cuando el workflow de correo exista.

### Fase 8 — Piloto (postest) y cierre (según cronograma de la tesis)
- [ ] Capacitación al personal y a los pacientes (vinculación de Telegram) — se apoya en TAM (utilidad y facilidad de uso percibidas).
- [ ] Cerrar fase PRETEST, abrir POSTEST con fechas fijas.
- [ ] Monitoreo semanal de los indicadores y de la calidad del dato (mediciones sospechosas, citas sin desenlace, consultas sin cerrar).
- [ ] Exportar, analizar en SPSS (Shapiro-Wilk, Wilcoxon) y redactar Resultados/Discusión.

## 5. Impacto en el documento de tesis (revisar con el asesor)

1. **TPR**: la fórmula dice "tiempo de registro de citas / número de citas registradas" y la dimensión "registro de pacientes". Se recomienda definir TPR sobre el registro de la atención (cita) para tener pares por paciente; el alta de paciente se reporta como descriptivo.
2. **NCA**: precisar qué es una consulta y qué es "atendida/resuelta" (ver `indicadores-tesis.md` §5); reportar también NCA_auto (solo chatbot).
3. **TNS**: precisar que canceladas con aviso y reprogramadas no cuentan.
4. **Definición operacional de la VI**: mencionar Telegram como canal de recordatorios y del chatbot (junto con Gemini y n8n, que ya figuran).
5. **Instrumento 03 postest**: el medio puede ser "Chatbot web" o "Telegram".
6. **Tecnologías**: Angular 21 (si el documento menciona 20).

## 6. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Pacientes sin Telegram o sin smartphone | Vinculación asistida en recepción; recordatorio telefónico registrado como canal `LLAMADA`; reportar cobertura |
| Pocas consultas en el postest (n pequeño) | Promover el bot en recepción; base de preguntas frecuentes útil; registrar también consultas presenciales |
| Personal no registra desenlaces | Agenda del día simple + cierre automático a 48 h + alerta diaria |
| Caída de Gemini o cuota agotada | Modo degradado existente + escalamiento a personal (cuenta como consulta, no se pierde) |
| n8n caído | Outbox + reintentos; los recordatorios quedan `PENDIENTE` y se envían al volver |
| Retraso del desarrollo frente al cronograma | Fases 1–4 son imprescindibles para la tesis; 5–7 pueden reducirse (p. ej., una sola app con dos layouts) |

## 7. Decisiones pendientes

- [ ] Confirmar con el asesor la operacionalización de TPR (cita vs. alta de paciente) y de NCA (total vs. solo chatbot).
- [ ] Fechas de las fases pretest y postest.
- [ ] ¿La fundación tratará las cancelaciones tardías como inasistencia? (por defecto, no).
- [ ] Dominio y hosting para producción (necesario para el webhook HTTPS de Telegram).
