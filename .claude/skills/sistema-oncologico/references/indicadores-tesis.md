# Indicadores de la tesis: definición operativa e implementación

Contenido:
1. Contexto metodológico que condiciona el diseño
2. Módulo de estudio (fases y participantes)
3. H1 — TPR, tiempo promedio de registro
4. H2 — TNS, tasa de ausentismo
5. H3 — NCA, nivel de consultas atendidas
6. Modelo de datos (migración propuesta)
7. Cálculo, endpoints y exportación para SPSS
8. Casos borde y tests obligatorios

---

## 1. Contexto metodológico que condiciona el diseño

- **Diseño**: preexperimental, un solo grupo, pretest–postest. Población 32, muestra 30 pacientes (no probabilístico por conveniencia).
- **Prueba**: Shapiro-Wilk y luego **Wilcoxon para muestras relacionadas** (α = 0,05). Wilcoxon compara *pares* (mismo paciente, antes y después). Por eso cada indicador necesita un **valor por paciente y por fase**, además del valor global.
- **Instrumentos (Anexo 2)**: ficha de registro de tiempos (hora inicio / hora fin), ficha de registro de ausentismo (asistió sí/no), ficha de registro del sistema (medio, consulta, resuelta sí/no). El sistema debe producir exactamente estas fichas, en digital, para ambas fases.
- **Situación pretest**: todo se llevaba en hojas de cálculo; las consultas llegaban por WhatsApp y llamadas; ausentismo del 65 %.
- **Ética (Ley 29733)**: solo participan pacientes con consentimiento informado; los datos exportados se anonimizan con un código de participante (`P01`…`P30`).

## 2. Módulo de estudio (fases y participantes)

Sin esto, ningún indicador es atribuible a la muestra ni al periodo correcto.

- `estudio_fase`: `PRETEST` y `POSTEST`, cada una con `fecha_inicio` y `fecha_fin` (y opcionalmente `estado` ABIERTA/CERRADA). Configurable por `ADMIN`/`INVESTIGADOR`.
- `participante_estudio`: vincula `paciente_id` con `codigo` (`P01`), `fecha_consentimiento`, `incluido` (bool) y `motivo_exclusion` (criterios de la tesis: suspende tratamiento, fallece, retira consentimiento, registros incompletos).
- Cada evento medible guarda la **fase explícitamente** al momento de insertarse (resuelta por un servicio `ResolverFaseEstudio` según la fecha del evento). Guardarla —en vez de deducirla luego— congela el dato aunque después se editen las fechas de la fase.
- Los indicadores se pueden pedir con tres alcances: `GLOBAL` (todo el sistema, para la gestión diaria), `MUESTRA` (solo participantes incluidos) y `POR_PARTICIPANTE` (para Wilcoxon). La tesis usa `MUESTRA` y `POR_PARTICIPANTE`.

## 3. H1 — TPR, tiempo promedio de registro

### Qué se mide exactamente

La fórmula del Anexo 1 es `TPR = Σ TRC / NCR` con *TRC = tiempo de registro de citas* y *NCR = número de citas registradas*, mientras que la dimensión se llama "Registro de pacientes". Ambas lecturas se reconcilian así:

- **Evento principal (`REGISTRO_CITA`)**: el proceso de registrar la atención de un paciente (buscarlo/identificarlo, validar o actualizar sus datos y dejar la cita registrada). Ocurre **varias veces por paciente**, en ambas fases, y por eso permite pares pretest/postest para Wilcoxon. Es el que coincide con la fórmula.
- **Eventos complementarios**: `REGISTRO_PACIENTE` (alta de la ficha, ocurre una sola vez por paciente → útil como descriptivo, no pareable) y `ACTUALIZACION_PACIENTE`.

> Decisión a confirmar con el asesor: si la tesis define TPR sobre el alta del paciente, no habrá pares por paciente (cada paciente se registra una vez). Recomendación: mantener la fórmula sobre citas registradas y aclarar en el texto que "registro de pacientes" abarca el registro de su atención. Cualquier cambio aquí debe reflejarse en el Anexo 1 y en la matriz de consistencia.

### Cómo se mide (postest, automático)

Patrón **sesión de medición**, sellado por el servidor:

1. Al abrir el formulario (intranet: "Nueva cita", "Nuevo paciente"; portal: "Completar perfil"; chatbot: inicio de una consulta de agendamiento) el frontend llama `POST /api/v1/mediciones/registro` con `{tipo, canal}`. El backend crea la fila con `inicio = now()` y devuelve `medicionId`.
2. El request de guardado (`POST /citas`, `POST /pacientes`) incluye `medicionId`. El caso de uso, **en la misma transacción** que persiste la cita/paciente, cierra la medición: `fin = now()`, `entidad_id`, `paciente_id`, `usuario_id` del operador.
3. Una medición sin cerrar después de 60 min (configurable) se marca `ABANDONADA` por un job y **no entra** al promedio (sí se reporta como dato descriptivo: tasa de abandono de formularios).
4. Validaciones: la medición pertenece al mismo usuario que guarda; no se reutiliza; si `fin - inicio` < 5 s se marca `SOSPECHOSA` para revisión (no se descarta automáticamente).
5. Eliminar `tiempoRegistroSegundos` de los DTO y de los formularios Angular (hoy se calcula con `Date.now()` en el navegador). La columna `paciente.tiempo_registro_segundos` queda obsoleta; se migra su contenido a `medicion_registro` con `origen = LEGADO` o se ignora, pero no se sigue alimentando.

Canales: `INTRANET`, `PORTAL`, `CHATBOT_WEB`, `TELEGRAM`, `MANUAL` (pretest). Los registros autoservicio (chatbot/portal) se reportan por separado, porque miden el tiempo del paciente y no del personal; el TPR de la hipótesis usa por defecto `INTRANET` + `MANUAL` (comparación justa: personal antes vs. personal después). Hacer el filtro de canal parametrizable en el endpoint.

### Cómo se mide (pretest, manual)

Pantalla "Ficha de registro de tiempos — Pretest" (rol `INVESTIGADOR`): participante, fecha, hora inicio, hora fin, tipo de registro, observación. Se guarda en la **misma tabla** con `canal = MANUAL`, `fase = PRETEST`, `capturado_por`. El cálculo es idéntico para ambas fases.

### Cálculo

```
TPR_min = AVG(duracion_segundos) / 60
          sobre medicion_registro
          donde estado = 'COMPLETADA' AND tipo = :tipo AND fase = :fase
            AND canal IN (:canales) AND [paciente en muestra]
```
Por participante: el mismo promedio agrupado por `paciente_id`.

## 4. H2 — TNS, tasa de ausentismo

### Qué se mide exactamente

`TNS = NI / (NI + NCC) × 100`

- `NI` (inasistencias): citas con desenlace `NO_ASISTIO` — el paciente no acudió **sin previo aviso**.
- `NCC` (citas cumplidas): citas con desenlace `ATENDIDA`.
- **No entran** al denominador: `CANCELADA` (hubo aviso), citas reprogramadas (la cita original pasa a `REPROGRAMADA` y la nueva es la que cuenta), citas futuras, citas `PROGRAMADA`/`CONFIRMADA` sin desenlace registrado.
- El periodo se filtra por **fecha de la cita**, no por fecha de creación.

> El código actual calcula `NO_ASISTIO / total de citas del periodo`; ese denominador incluye canceladas y futuras y subestima el ausentismo. Corregir en `IndicadoresRepositoryAdapter`.

### Cómo se registra el desenlace

- Recepción marca `ATENDIDA` (check-in) o `NO_ASISTIO` desde la agenda del día (intranet). Guardar `fecha_hora_desenlace` y `desenlace_registrado_por`.
- Job diario (con ShedLock) a las 23:00: citas del día aún `PROGRAMADA`/`CONFIRMADA` → bandeja "Citas pendientes de cierre". A las 48 h sin acción, se cierran como `NO_ASISTIO` con `cierre_automatico = true` (corregible con evento de corrección auditado). Así nunca queda un hueco que sesgue el indicador.
- Cancelación tardía: si la fundación quiere tratar como inasistencia una cancelación con menos de N horas de anticipación, que sea un parámetro (`ausentismo.cancelacion-tardia-horas`, por defecto desactivado) y que se reporte aparte. No cambiar la definición en silencio.

### Pretest

- Importación de las hojas de cálculo históricas (CSV/XLSX → `cita` con `origen = IMPORTACION_PRETEST`) o captura en la "Ficha de registro de ausentismo — Pretest" (participante, fecha de cita, asistió S/N).
- Las citas importadas no disparan recordatorios ni eventos de negocio.

### Métricas de apoyo (explican el *por qué* bajó el TNS)

No forman parte de la hipótesis, pero sostienen la discusión: % de citas con recordatorio entregado, % confirmadas por Telegram, TNS de citas con vs. sin recordatorio, tiempo entre agendamiento y cita (McMullen y Netland).

## 5. H3 — NCA, nivel de consultas atendidas

### Qué es una "consulta"

Una consulta es **una necesidad del usuario**, no un mensaje. Una conversación puede contener varias consultas y una consulta puede tomar varios mensajes.

- Se **abre** con el primer mensaje de una sesión (web o Telegram) o cuando Gemini detecta una intención accionable distinta a la de la consulta abierta.
- Se **cierra** con uno de estos resultados:

| Resultado | Cuándo |
|---|---|
| `RESUELTA_BOT` | La acción del catálogo se ejecutó con éxito (agendar, reprogramar, cancelar, confirmar, consultar citas) **o** una intención informativa recibió respuesta y el usuario la valoró 👍 o no repitió/reformuló la misma intención en los siguientes 10 min |
| `ESCALADA` | Intención `ESCALATE_TO_STAFF`, valoración 👎, error de negocio no recuperable o dos reformulaciones seguidas. Pasa a la **bandeja de consultas** del personal |
| `RESUELTA_PERSONAL` | Un trabajador atendió la consulta escalada y la marcó como resuelta |
| `NO_RESUELTA` | Escalada sin respuesta en 48 h, o cerrada por el personal como no resuelta, o abandonada sin resolución en una intención accionable |

Cada consulta guarda: `canal` (`CHATBOT_WEB`, `TELEGRAM`, `WHATSAPP`, `LLAMADA`, `PRESENCIAL`), `intencion`, `paciente_id` (si se identificó), `sesion_id`, `abierta_en`, `cerrada_en`, `resultado`, `resuelta_por_usuario_id`, `valoracion` (1/-1/null), `tiempo_primera_respuesta_ms`, `fase`.

### Cálculo

```
NCA      = (RESUELTA_BOT + RESUELTA_PERSONAL) / total consultas cerradas × 100   ← indicador de la hipótesis
NCA_auto = RESUELTA_BOT / total consultas cerradas × 100                          ← aporte específico del chatbot
```

Usar el NCA total para la hipótesis mantiene la comparación justa con el pretest (donde todo lo resolvía el personal) y el `NCA_auto` evidencia la automatización. Reportar ambos y el tiempo medio de primera respuesta. Las consultas abiertas al cierre del periodo se excluyen del denominador hasta cerrarse.

### Pretest

"Ficha de registro del sistema — Pretest": fecha, medio (WhatsApp/Llamada/Presencial), participante (si aplica), tipo de consulta, resuelta S/N. Se guarda en `consulta` con `fase = PRETEST`, `canal` correspondiente y `capturado_por`.

### Implementación en el chatbot actual

`ChatbotOrquestadorUseCase.procesar` hoy guarda cada mensaje en `conversacion_chatbot`. Añadir un `GestorConsultaService` que, en cada turno, decida abrir/continuar/cerrar la consulta en función de la intención y del resultado de `ejecutarAccion`. Para eso `ejecutarAccion` debe devolver un resultado tipado (`RespuestaChatbot(texto, ResultadoAccion)`: `EXITO`, `INFORMATIVA`, `REQUIERE_SESION`, `ERROR_NEGOCIO`, `ESCALAR`) en lugar de un `String`. Añadir botones 👍/👎 en el widget y en Telegram (`POST /chatbot/consultas/{id}/valoracion`).

## 6. Modelo de datos (migración propuesta `V5__modulo_estudio_indicadores.sql`)

```sql
CREATE TABLE estudio_fase (
    id            BIGSERIAL PRIMARY KEY,
    fase          VARCHAR(10) NOT NULL UNIQUE CHECK (fase IN ('PRETEST','POSTEST')),
    fecha_inicio  DATE NOT NULL,
    fecha_fin     DATE NOT NULL,
    estado        VARCHAR(10) NOT NULL DEFAULT 'ABIERTA',
    CHECK (fecha_fin >= fecha_inicio)
);

CREATE TABLE participante_estudio (
    id                    BIGSERIAL PRIMARY KEY,
    paciente_id           BIGINT NOT NULL UNIQUE REFERENCES paciente (id),
    codigo                VARCHAR(10) NOT NULL UNIQUE,          -- P01..P30
    fecha_consentimiento  DATE NOT NULL,
    incluido              BOOLEAN NOT NULL DEFAULT TRUE,
    motivo_exclusion      VARCHAR(40),
    observacion           TEXT
);

CREATE TABLE medicion_registro (
    id                 BIGSERIAL PRIMARY KEY,
    tipo               VARCHAR(30) NOT NULL,   -- REGISTRO_CITA | REGISTRO_PACIENTE | ACTUALIZACION_PACIENTE
    canal              VARCHAR(20) NOT NULL,   -- INTRANET | PORTAL | CHATBOT_WEB | TELEGRAM | MANUAL
    fase               VARCHAR(10),
    estado             VARCHAR(15) NOT NULL,   -- EN_CURSO | COMPLETADA | ABANDONADA | SOSPECHOSA
    usuario_id         BIGINT REFERENCES usuario (id),
    paciente_id        BIGINT REFERENCES paciente (id),
    entidad_id         BIGINT,
    inicio             TIMESTAMPTZ NOT NULL,
    fin                TIMESTAMPTZ,
    duracion_segundos  INTEGER GENERATED ALWAYS AS (EXTRACT(EPOCH FROM (fin - inicio))::INTEGER) STORED,
    capturado_por      BIGINT REFERENCES usuario (id),   -- solo MANUAL
    observacion        TEXT
);
CREATE INDEX idx_medicion_registro_calculo ON medicion_registro (tipo, fase, estado, canal);
CREATE INDEX idx_medicion_registro_paciente ON medicion_registro (paciente_id);

CREATE TABLE consulta (
    id                          BIGSERIAL PRIMARY KEY,
    canal                       VARCHAR(20) NOT NULL,
    sesion_id                   VARCHAR(100),
    paciente_id                 BIGINT REFERENCES paciente (id),
    intencion                   VARCHAR(40),
    resumen                     VARCHAR(300),
    fase                        VARCHAR(10),
    resultado                   VARCHAR(20),       -- null mientras esta abierta
    abierta_en                  TIMESTAMPTZ NOT NULL,
    cerrada_en                  TIMESTAMPTZ,
    tiempo_primera_respuesta_ms INTEGER,
    valoracion                  SMALLINT CHECK (valoracion IN (-1, 1)),
    resuelta_por_usuario_id     BIGINT REFERENCES usuario (id),
    capturado_por               BIGINT REFERENCES usuario (id)
);
CREATE INDEX idx_consulta_calculo ON consulta (fase, resultado, canal);
CREATE INDEX idx_consulta_sesion ON consulta (sesion_id);
ALTER TABLE conversacion_chatbot ADD COLUMN consulta_id BIGINT REFERENCES consulta (id);

ALTER TABLE cita
    ADD COLUMN origen                    VARCHAR(25) NOT NULL DEFAULT 'INTRANET', -- INTRANET|PORTAL|CHATBOT_WEB|TELEGRAM|IMPORTACION_PRETEST
    ADD COLUMN fase                      VARCHAR(10),
    ADD COLUMN fecha_creacion            TIMESTAMPTZ NOT NULL DEFAULT now(),
    ADD COLUMN fecha_hora_desenlace      TIMESTAMPTZ,
    ADD COLUMN desenlace_registrado_por  BIGINT REFERENCES usuario (id),
    ADD COLUMN cierre_automatico         BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN cita_origen_id            BIGINT REFERENCES cita (id);           -- trazabilidad de reprogramaciones

-- Correcciones: nunca UPDATE sobre mediciones cerradas; se registra el ajuste.
CREATE TABLE correccion_medicion (
    id            BIGSERIAL PRIMARY KEY,
    entidad       VARCHAR(30) NOT NULL,   -- MEDICION_REGISTRO | CONSULTA | CITA_DESENLACE
    entidad_id    BIGINT NOT NULL,
    campo         VARCHAR(40) NOT NULL,
    valor_previo  TEXT,
    valor_nuevo   TEXT,
    motivo        TEXT NOT NULL,
    usuario_id    BIGINT NOT NULL REFERENCES usuario (id),
    fecha         TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Añadir `REPROGRAMADA` a `EstadoCita` y el rol `INVESTIGADOR` a `Rol`. Proteger `medicion_registro` con un trigger que solo permita `UPDATE` de filas `EN_CURSO` (cerrarlas) y ningún `DELETE`.

## 7. Cálculo, endpoints y exportación para SPSS

Dominio: `domain/estudio/` con `FaseEstudio`, `ParticipanteEstudio`, `IndicadoresEstudio` (record con TPR, TNS, NCA, NCA_auto, n de cada uno) y un puerto `IndicadoresEstudioRepositoryPort`. Las fórmulas viven en el **dominio** (métodos estáticos puros, fáciles de testear: `Indicador.tns(ni, ncc)`), y el adaptador JPA solo aporta los conteos.

Endpoints (`/api/v1/estudio`, roles `ADMIN`, `INVESTIGADOR`):

| Método | Ruta | Uso |
|---|---|---|
| GET/PUT | `/fases` | Configurar periodos pretest/postest |
| GET/POST/PATCH | `/participantes` | Muestra, consentimiento, exclusiones |
| POST | `/mediciones/manual`, `/consultas/manual`, `/asistencias/manual` | Fichas del pretest |
| POST | `/importaciones/citas` | Importar hoja de cálculo histórica (CSV/XLSX con validación por fila) |
| GET | `/indicadores?fase=&alcance=` | TPR, TNS, NCA, NCA_auto con su n |
| GET | `/indicadores/comparativo` | Pretest vs postest + diferencia + % de mejora |
| GET | `/exportaciones/spss.xlsx` | Libro para SPSS |
| GET | `/exportaciones/fichas.xlsx` | Las 3 fichas del Anexo 2 llenas, por fase |

Libro `spss.xlsx` (Apache POI):
- Hoja `pareado`: una fila por participante incluido → `codigo, TPR_pre, TPR_post, TNS_pre, TNS_post, NCA_pre, NCA_post, n_registros_pre, n_registros_post, n_citas_pre, n_citas_post, n_consultas_pre, n_consultas_post`. Celdas vacías (no cero) cuando no hay datos en una fase.
- Hojas `detalle_tpr`, `detalle_tns`, `detalle_nca`: un evento por fila, con código de participante en vez de nombre/DNI.
- Hoja `diccionario`: definición de cada variable y la fórmula.

Vista opcional en el dashboard: Wilcoxon de rangos con signo con `commons-math3` (`WilcoxonSignedRankTest`) como **vista previa**; el análisis oficial sigue siendo SPSS. Mostrar siempre el n efectivo.

## 8. Casos borde y tests obligatorios

- División por cero → devolver `null` ("sin datos"), no 0 % (0 % de ausentismo es un resultado, "sin datos" es otro).
- Cita reprogramada dos veces → solo la última cuenta para TNS.
- Cita cancelada → fuera del TNS; cita importada del pretest → no genera recordatorios.
- Medición abierta y nunca cerrada → `ABANDONADA`, fuera del TPR.
- Medición cerrada por otro usuario o reutilizada → rechazo (`ValidacionDeNegocioException`).
- Consulta de visitante anónimo → cuenta para NCA global, no para el pareado.
- Consulta abierta al final de la fase → excluida hasta cerrarse.
- Paciente excluido del estudio a mitad del postest → fuera de `MUESTRA` y del pareado, pero sus datos no se borran.
- Evento justo en el límite de fecha entre fases → la fase se decide por la fecha del evento con zona `America/Lima`.
- Usar `TIMESTAMPTZ` y `Clock` inyectable en los casos de uso para poder testear el tiempo.

Cada fórmula: test unitario de dominio. Cada consulta SQL de conteo: test de integración con Testcontainers sembrando un escenario conocido (p. ej., 10 citas: 6 atendidas, 3 no asistió, 1 cancelada → TNS = 33,3 %).
