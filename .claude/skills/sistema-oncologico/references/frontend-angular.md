# Frontend: Angular + Tailwind CSS 4

> Versión: el repositorio ya está en Angular 21 (compatible con todo lo que se pedía para Angular 20: standalone, signals, control flow `@if/@for`, `inject()`). No bajar de versión; si la tesis menciona Angular 20, basta con actualizar el texto.

## Dos aplicaciones, una librería compartida

El sistema tiene dos públicos con necesidades distintas. Separarlos reduce el bundle del paciente, permite desplegarlos en dominios distintos (y restringir la intranet por red/IP) y evita que un cambio en la intranet rompa el portal.

```
frontend/                         (workspace Angular, `ng new --create-application=false` + migración)
├── projects/
│   ├── portal/                   público: inicio, información de la fundación, preguntas frecuentes,
│   │                             login/registro de paciente, portal del paciente (mis citas, mi perfil,
│   │                             vincular Telegram), widget del chatbot
│   ├── intranet/                 personal: dashboard de indicadores, agenda del día (check-in /
│   │                             inasistencia), pacientes, citas, tratamientos, bandeja de consultas,
│   │                             módulo de estudio (fases, participantes, fichas pretest, exportación),
│   │                             usuarios, auditoría
│   └── shared/                   librería: core (auth, interceptores, guards, modelos, cliente API),
│                                 ui (button, card, input, modal, tabs...), components (data-table,
│                                 pagination, stat-card...), pipes, validators, estilos/tema
```

Migración desde la app única actual: mover `src/app/shared` y `src/app/core` a `projects/shared`, `features/paciente-portal` + `features/auth` + `features/chatbot` a `portal`, el resto a `intranet`. Hacerlo **en una fase propia** sin cambiar comportamiento, compilando ambas apps al final; después seguir con las funcionalidades nuevas.

Si el tiempo de la tesis aprieta, la alternativa aceptable es una sola app con dos layouts (`PortalLayout`, `IntranetShell`) y rutas lazy separadas por prefijo (`/portal/**`, `/intranet/**`); la carpeta `shared` debe quedar igual de aislada para poder separar después.

## Convenciones (las que ya usa el código)

- Componentes standalone, `inject()` en lugar de constructor, estado con `signal`/`computed`, `input()`/`output()`; `ChangeDetectionStrategy.OnPush` en todos los componentes nuevos.
- Servicios por feature (`features/<feature>/<feature>.service.ts`) que devuelven `Observable` tipado con los modelos de `core/models`. URL base desde `environment.apiUrl`.
- Formularios reactivos tipados; validación en tres niveles: formato (validators), unicidad asíncrona contra la API (`documento-unico.validator.ts`), errores del backend mapeados al control.
- Rutas lazy con `loadComponent`, `authGuard` + `roleGuard(...)`. Nuevo rol `INVESTIGADOR` para el módulo de estudio.
- Interceptores funcionales: `jwtInterceptor` (renovación automática con refresh token) y `errorInterceptor` (toast).
- Gráficos con ApexCharts (`ng-apexcharts`), como el dashboard actual.
- Tests con Vitest.

## Tailwind 4 y diseño

- El tema está en `styles.css` con `@theme inline` y tokens semánticos oklch (`--primary`, `--muted`, `--success`, `--warning`, `--destructive`...). Modo oscuro redefiniendo las variables bajo `.dark`. Usa siempre las clases semánticas (`bg-card`, `text-muted-foreground`, `border-border`, `bg-success-soft`); nunca hex ni la paleta numérica de Tailwind, para que el modo oscuro y un futuro rebranding funcionen sin tocar componentes.
- Reutiliza `shared/ui` (button, card, input, select, modal, tabs, badge) y `shared/components` (data-table, page-header, stat-card, empty-state, skeleton-rows) antes de crear algo nuevo.
- Accesibilidad: el portal lo usan pacientes oncológicos, muchos adultos mayores. Texto base ≥ 16 px, contraste AA, botones grandes, foco visible, etiquetas en todos los campos, sin depender solo del color para estados.
- Mobile-first en el portal (la mayoría llegará desde el enlace de Telegram en el celular).

## Pantallas nuevas clave

### Intranet
- **Dashboard de indicadores (rediseño)**: tres tarjetas principales TPR (min), TNS (%), NCA (%) con selector de fase (Pretest / Postest / Comparativo) y alcance (Muestra / Global); cada tarjeta muestra el n y la variación vs. pretest. Debajo: tendencia semanal, desglose por canal, NCA vs NCA_auto, métricas de apoyo de recordatorios. Valores `null` se muestran como "Sin datos", nunca como 0.
- **Agenda del día (recepción)**: lista de citas de hoy con acciones grandes "Llegó" / "No asistió", indicador de confirmación por Telegram y recordatorios enviados.
- **Bandeja de consultas**: consultas escaladas por el chatbot, con historial de la conversación y acciones "Resolver" / "No resuelta".
- **Módulo de estudio**: fases, participantes (código, consentimiento, exclusión), las tres fichas de captura pretest, importación de hojas de cálculo con vista previa y errores por fila, exportación SPSS.

### Formularios que miden TPR
`paciente-form`, el formulario de nueva cita y `completar-perfil` deben:
1. Al abrirse en modo creación, llamar `POST /mediciones/registro` y guardar el `medicionId` en una signal.
2. Enviar `medicionId` al guardar.
3. Eliminar el cálculo actual con `Date.now()` y el campo `tiempoRegistroSegundos`.
Encapsularlo en un servicio `MedicionRegistroService` (`iniciar(tipo, canal)`) para no duplicar lógica.

### Portal
- Inicio público con información útil (horarios, servicios, preguntas frecuentes) y acceso al chatbot.
- Mis citas con confirmar / reprogramar / cancelar.
- **Vincular Telegram**: botón que obtiene el deep link `https://t.me/<bot>?start=<token>` y muestra QR + estado de vinculación.
- Widget de chatbot con botones 👍/👎 por respuesta y opción "Hablar con una persona".

## Comandos

```bash
cd frontend
npm start                      # app actual (tras la separación: ng serve portal / ng serve intranet)
npm test                       # Vitest
npm run build                  # build de producción; debe pasar sin warnings de presupuesto
```
