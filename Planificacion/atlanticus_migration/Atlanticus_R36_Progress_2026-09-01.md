# Atlanticus — Estado de avance R3.6

**Fecha de corte:** 2026-09-01  
**Repositorio de decisiones:** `moragaga/atlanticus-decisions`  
**Estado:** checkpoint de trabajo previo al primer deploy

## 1. Objetivo de este archivo

Este documento resume el estado validado de la migración y productización de KPI dentro del repositorio canónico `atlanticus`, tomando `atlanticus-stage` como referencia histórica y manteniendo las decisiones arquitectónicas ya cerradas.

No reemplaza los contratos de cada módulo ni los gates autoritativos. Su propósito es dejar un punto de sincronización simple para continuar el trabajo entre sesiones.

## 2. Regla de versionado durante la iteración

Hasta completar el primer deploy, los packages estabilizados durante esta fase mantienen versión `1.0.0`.

Los identificadores del roadmap (`M-005B.3A`, `M-005B.3A.1`, etc.) representan incrementos de trabajo y no implican un bump de versión del package.

Después del primer deploy se retomará versionado incremental normal.

## 3. Estado KPI validado

### R3.6M-005A.1 — KPI Domain Clean Landing

**CERRADO / GREEN**

Layout canónico:

```text
scopes/ada/backend/kpis/
├── core
├── evaluation
└── persistence
```

Packages:

```text
ada-kpis-core==1.0.0
ada-kpis-evaluation==1.0.0
ada-kpis-persistence==1.0.0
```

Namespace funcional:

```text
ada.kpis.*
```

El directorio físico `backend` no forma parte del namespace Python.

### R3.6M-005A.2 — KPI Runtime

**CERRADO / GREEN**

Layout:

```text
scopes/ada/backend/processes/kpi-runtime
```

Package y entrypoint:

```text
ada-kpi-runtime-process==1.0.0
ada.processes.kpi_runtime
ada-kpi-runtime
```

Lifecycle validado:

```text
new operational watermark
→ evaluation
→ persistence
→ advance KPI committed watermark
```

Una segunda ejecución con la misma autoridad no genera trabajo nuevo.

`build_catalog()` permanece deliberadamente vacío e inyectable; este cierre no declara KPIs productivos configurados.

### R3.6M-005B.1 — KPI Delivery Domain Clean Landing

**CERRADO / GREEN**

Layout:

```text
scopes/ada/backend/kpis/delivery
```

Package:

```text
ada-kpis-delivery==1.0.0
ada.kpis.delivery
```

Responsabilidad:

- contratos puros de Delivery;
- proyección Latest;
- contrato Timeseries;
- routing KPI → destinos;
- revisiones determinísticas.

No contiene Cosmos, Job Runtime, filesystem, checkpoints, Web ni provisión de infraestructura.

Latest conserva el contrato:

```text
status
value_kind
value
```

Timeseries conserva:

```text
step_seconds = 120
window = (start_utc, end_utc]
exact timestamp match
missing exact point = null
```

### R3.6M-005B.2 — KPI Latest Delivery Runtime

**CERRADO / GREEN**

Layout:

```text
scopes/ada/backend/processes/kpi-delivery
```

Package y entrypoint:

```text
ada-kpi-delivery-process==1.0.0
ada.processes.kpi_delivery
ada-kpi-delivery
```

Lifecycle validado:

```text
process start
→ load KPI Delivery Configuration from Cosmos once
→ strict validation
→ freeze configuration
→ read KPI committed watermark
→ compare Delivery checkpoint
→ read persisted KpiEvaluationBatch
→ adapt Evaluation → Delivery
→ project Latest
→ publish Cosmos
→ commit Delivery checkpoint LAST
```

Decisiones relevantes:

- no polling de configuración durante el Job;
- `APPLICATION` y `KPI_RUNTIME_APPLICATION` son autoridades separadas;
- no se reconstruyó el antiguo `KpiLatestRepository` de Stage;
- se consume directamente `KpiEvaluationRepository`;
- publicación Cosmos es idempotente;
- checkpoint sólo avanza después de una publicación válida.

### R3.6M-005B.3A — KPI History Contract Clean Landing

**CERRADO / GREEN**

Layout:

```text
scopes/ada/backend/kpis/history
```

Package:

```text
ada-kpis-history==1.0.0
ada.kpis.history
```

Este módulo es la autoridad compartida del contrato durable que consumirán KPI Historian y Timeseries Delivery.

Datasets:

```text
kpis/history
kpis/error-history
```

Materialización:

```text
daily
partition = year / month / day
```

Identidad de History:

```text
timestamp_utc + key
```

Columnas:

```text
timestamp_utc
key
status
value_kind
value
parsed_value
```

Error History:

```text
timestamp_utc
key
error
```

Los valores se serializan como JSON compacto y determinístico antes de persistirse como texto.

La definición ya no se duplica entre productor Historian y consumidor Timeseries.

### R3.6M-005B.3A.1 — KPI History Authority Contract

**CERRADO / GREEN**

`ada-kpis-history` continúa en `1.0.0`.

Se agregó la autoridad compartida de Historian sin incorporar `atlanticus-state` al dominio.

Identidad lógica:

```text
namespace = ('kpis', 'history')
name = 'authority'
```

Payload:

```text
schema_version
watermark_utc
revision
```

La revisión se deriva determinísticamente del contrato History y del committed watermark.

KPI Historian será responsable de escribir esta autoridad; Timeseries Delivery podrá leer el mismo contrato sin depender del process Historian.

## 4. Arquitectura resultante

```text
KPI Runtime
    │
    │ persisted KpiEvaluationBatch
    │ KPI committed watermark
    ▼
KPI Historian
    │
    │ kpis/history
    │ kpis/error-history
    │ Historian authority
    ▼
KPI Timeseries Delivery
    │
    │ Timeseries projection
    ▼
Cosmos DB
    │
    ▼
Web
```

En paralelo:

```text
KPI Runtime
    │
    ▼
KPI Latest Delivery
    │
    ▼
Cosmos DB
    │
    ▼
Web
```

## 5. Principios preservados

- Backend contracts antes que frontend.
- `kpi_key` es la identidad única de KPI.
- Process roots componen capacidades; no se crean dependencias process → process.
- Sin clientes globales.
- Configuración explícita y modular.
- Sin nombres físicos de containers Cosmos hardcodeados cuando no están definidos.
- Publish/materialize primero; checkpoint/authority después.
- Reintentos idempotentes.
- Sin secretos en código o logs.
- `atlanticus-stage` es referencia histórica, no autoridad productiva.
- `atlanticus` es el repositorio canónico.
- Producción y espejo comentado mantienen equivalencia semántica.

## 6. Próximo incremento

### R3.6M-005B.3B — KPI Historian Runtime

**SIGUIENTE**

Scope previsto:

```text
scopes/ada/backend/processes/kpi-historian
ada-kpi-historian-process==1.0.0
ada.processes.kpi_historian
ada-kpi-historian
```

Lifecycle objetivo:

```text
read KPI committed watermark
→ read Historian authority
→ validate no regression
→ KpiEvaluationRepository.read_after(
     after=historian_committed,
     through=kpi_committed
   )
→ materialize eligible evaluations into daily Parquet
→ materialize errors into error-history
→ verify last durable batch reaches KPI committed
→ commit Historian authority LAST
```

Fronteras previstas:

```text
APPLICATION
→ propietario de Historian state + datasets

KPI_RUNTIME_APPLICATION
→ aplicación upstream propietaria de KPI persistence
```

El proceso deberá consumir `ada.kpis.history`; no redefinir dataset, schema, encoding ni authority identity.

## 7. Después de Historian

Orden recomendado:

```text
M-005B.3B
KPI Historian Runtime

M-005B.4
KPI Timeseries Delivery Runtime

después
integración Web sobre contratos backend cerrados
```

Timeseries Delivery no debe comenzar antes de disponer de una autoridad Historian limpia y validada.

## 8. Estado de cierre

```text
M-005A.1     GREEN
M-005A.2     GREEN
M-005B.1     GREEN
M-005B.2     GREEN
M-005B.3A    GREEN
M-005B.3A.1  GREEN
M-005B.3B    NEXT
M-005B.4     PENDING
```

Este archivo representa el checkpoint de avance vigente al 2026-09-01.
