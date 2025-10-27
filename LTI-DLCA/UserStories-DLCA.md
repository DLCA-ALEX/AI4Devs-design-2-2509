# User Stories – AS (LTI ATS v1)

> Fecha: 2025-10-27 · Autor: AS · Versión: 1.0

## 1. Introducción
Este documento recoge **User Stories**, **Backlog priorizado**, **tickets técnicos** y **estimaciones** para iniciar la implementación del **LTI ATS v1**. Se basa en el PRD previamente definido (módulos: multiposting, parsing + búsqueda semántica, pipeline y colaboración, entrevistas + scorecards, automatizaciones, reportes y cumplimiento).

**Objetivos de esta entrega:**
- Alinear alcance inicial del MVP con foco en reducir *time-to-hire* y aumentar la automatización útil.
- Proveer un backlog priorizado con criterio WSJF y MoSCoW (referencia).
- Bajar a tickets una historia clave con criterios de aceptación y estimaciones.

---

## 2. User Stories detalladas (plantilla INVEST)
Tabla resumen (ver detalle más abajo):

| ID | Título | Rol | Funcionalidad | Valor | Prioridad (MoSCoW) |
|----|--------|-----|---------------|-------|---------------------|
| US-01 | Crear vacante y multiposting | Reclutador | Publicar en múltiples canales con tracking | Alto (↑ candidatos) | Must |
| US-02 | Ingesta/Parsing + dedupe + embeddings | Reclutador | Centralizar CVs y habilitar búsqueda semántica | Muy alto (↑ velocidad screening) | Must |
| US-03 | Pipeline Kanban + @mentions | Hiring Manager | Mover etapas, comentar, tareas | Alto (↓ fricción colaborativa) | Must |
| US-04 | Entrevistas integradas a calendario | Reclutador | Proponer horarios, agendar, no-show tracking | Medio/Alto | Should |
| US-05 | Scorecards estandarizados + agregación | Entrevistador | Evaluar con rúbrica y consolidar score | Alto (↑ calidad decisión) | Should |
| US-06 | Automatizaciones no‑code + auditoría | Reclutador | Reglas por evento para mover/etiquetar/notificar | Muy alto (↑ productividad) | Must |

A continuación, cada User Story con la plantilla completa.

### US-01 — Crear vacante y multiposting
**Como** Reclutador **quiero** crear una vacante desde plantilla y publicarla en múltiples canales **para** atraer candidatos rápidamente con trazabilidad por fuente.

**Criterios de aceptación (Gherkin):**
- **Given** que tengo permisos y una vacante en estado *draft*, **When** selecciono canales y publico, **Then** se crean registros `JobPosting` por canal con `url`, `status=published`, `posted_at` y UTM de tracking.
- **Given** un canal con error temporal, **When** publico, **Then** el sistema reintenta hasta 3 veces y deja `AutomationRun`/log de error.
- **Given** que edito el JD, **When** republique, **Then** se versiona el contenido y se actualiza el `external_ref` si aplica.

**Definition of Done:**
- Endpoint REST/GraphQL de publicación por canal con validaciones y colas de reintento.
- UI con selector de canales y estado por publicación.
- Auditoría en `AuditLog` y métricas por fuente.

**Prioridad:** Must  
**Dependencias:** RBAC básico, Catálogo de canales, Plantillas JD.  
**Riesgos:** Reglas y límites de APIs externas; rechazo de contenido por políticas.  
**KPIs:** *Time-to-publish*, candidatos por fuente, CTR por canal.

---

### US-02 — Ingesta/Parsing + dedupe + embeddings
**Como** Reclutador **quiero** importar CVs desde email/archivo/formulario y deduplicarlos **para** buscar semánticamente y evitar duplicados.

**Criterios de aceptación (Gherkin):**
- **Given** un CV subido o recibido por email, **When** se procesa, **Then** se genera `Resume` con `parsed_json`, `text`, `language`, `hash` único y se vincula a `Candidate`.
- **Given** un candidato existente, **When** el hash o `dedupe_key` coincide, **Then** se marca como potencial duplicado y se ofrece *merge* asistido.
- **Given** un `Resume` parseado, **When** se indexa, **Then** se generan *embeddings* y queda consultable por vector kNN.
- **Given** una búsqueda semántica “backend senior TypeScript”, **When** consulto, **Then** obtengo ranking híbrido (vector + BM25) con *highlights* y filtros por etapa/ubicación.

**Definition of Done:**
- Worker de parsing (PDF/DOCX), normalización y dedupe.
- Servicio de embeddings y vector store (`pgvector`).
- API de búsqueda híbrida con paginación y métricas de calidad (recall@k).

**Prioridad:** Must  
**Dependencias:** Object storage, Vector store, ETL parsing.  
**Riesgos:** Calidad de parsing; costos de embeddings; sesgos en ranking.  
**KPIs:** tasa de parsing correcto, duplicates detectados, recall@k, latencia P95.

---

### US-03 — Pipeline Kanban + @mentions y tareas
**Como** Hiring Manager **quiero** arrastrar candidatos entre etapas, comentar con *@mentions* y asignar tareas **para** coordinar al equipo con SLA visibles.

**Criterios de aceptación (Gherkin):**
- **Given** permisos, **When** arrastro un candidato de *screening* a *interview*, **Then** se registra `Stage.Changed` con SLA de la nueva etapa.
- **Given** un comentario con *@mention @usuario*, **When** publico, **Then** el usuario mencionado recibe notificación y queda `Note` + `Mention` en auditoría.
- **Given** una tarea, **When** la marco como *done*, **Then** se registra en `Task` y ajustan métricas de SLA.

**Definition of Done:**
- UI Kanban con DnD y contadores por etapa.
- Notificaciones y mentions con permisos.
- Reporte básico de SLA por etapa y equipo.

**Prioridad:** Must  
**Dependencias:** RBAC, Eventos de dominio, Notificaciones.  
**Riesgos:** Contención de concurrencia al mover etapas; spam de notificaciones.  
**KPIs:** tiempo por etapa, % tareas a tiempo, cumplimiento SLA p95.

---

### US-04 — Entrevistas integradas a calendario
**Como** Reclutador **quiero** proponer horarios y agendar entrevistas con Google/Microsoft **para** reducir *no‑shows* y conflictos.

**Criterios de aceptación (Gherkin):**
- **Given** disponibilidad del equipo, **When** envío propuesta, **Then** el candidato recibe slots y al confirmar se crea `Interview` con `status=scheduled` y `conferencing_link`.
- **Given** un *no‑show*, **When** registro el evento, **Then** se ofrece reprogramar y se actualiza métrica `no_show_rate`.
- **Given** conflictos de agenda, **When** intento agendar, **Then** el sistema sugiere slots alternos.

**Definition of Done:** Integración OAuth con Calendar, verificación de conflictos, plantillas de correo, registro de métricas.  
**Prioridad:** Should  
**Dependencias:** Integraciones Calendar, Mensajería/email.  
**Riesgos:** Scopes OAuth; zonas horarias.  
**KPIs:** tasa de *no‑show*, latencia de programación, tiempo medio hasta entrevista.

---

### US-05 — Scorecards estandarizados + agregación
**Como** Entrevistador **quiero** completar scorecards con rúbricas y pesos **para** comparar candidatos objetivamente y consolidar un *overall score*.

**Criterios de aceptación (Gherkin):**
- **Given** una entrevista, **When** envío feedback, **Then** se valida completitud de rúbrica y se actualiza `Application.score` ponderado.
- **Given** feedback tardío, **When** pasan 24h, **Then** se envía recordatorio automático al entrevistador.

**Definition of Done:** Rúbricas configurables, validación de completitud, cálculo de score y vista comparativa.  
**Prioridad:** Should  
**Dependencias:** Entrevistas, Modelo de `Scorecard`/`Feedback`.  
**Riesgos:** Sesgos en rúbricas; baja adopción.  
**KPIs:** cobertura de scorecards, latencia de feedback, correlación score↔oferta.

---

### US-06 — Automatizaciones no‑code + auditoría
**Como** Reclutador **quiero** crear reglas *no‑code* (triggers/conditions/actions) **para** etiquetar, enviar emails o mover etapas con auditoría.

**Criterios de aceptación (Gherkin):**
- **Given** una regla activa, **When** ocurre el trigger (p. ej., *Application.Created*), **Then** se evalúan condiciones y se ejecutan acciones con `AutomationRun` y logs.
- **Given** una acción fallida, **When** sucede, **Then** hay reintentos con *backoff* y alerta visible.

**Definition of Done:** Motor de reglas, UI declarativa, auditoría y métricas de ejecuciones.  
**Prioridad:** Must  
**Dependencias:** Bus de eventos, Mensajería, Auditoría.  
**Riesgos:** Cascadas de automaciones; bucles.  
**KPIs:** % acciones automatizadas, fallos/reintentos, tiempo ahorrado.

---

## 3. Product Backlog priorizado (WSJF + MoSCoW)
**Metodología:** WSJF = (Business Value + Time Criticality + Risk Reduction/OE) / Job Size. Escala relativa (1–8). Se incluye etiqueta MoSCoW como referencia.

| ID | Épica | Título | BV | TC | RR/OE | Job Size | WSJF | MoSCoW | Justificación |
|----|------|--------|----|----|-------|----------|------|--------|---------------|
| US-02 | Search/Matching | Ingesta/Parsing + dedupe + embeddings | 8 | 6 | 6 | 5 | **4.0** | Must | Desbloquea búsqueda y screening; impacto directo en TTH |
| US-03 | Pipeline/Colab | Pipeline Kanban + @mentions | 7 | 6 | 5 | 5 | **3.6** | Must | Visibilidad y coordinación inmediata |
| US-06 | Automation | Reglas no‑code + auditoría | 7 | 5 | 6 | 5 | **3.6** | Must | Ahorro de tiempo sostenido y control |
| US-01 | Multiposting | Crear vacante y multiposting | 6 | 6 | 4 | 5 | **3.2** | Must | Aumenta *top of funnel* y tracking |
| US-04 | Entrevistas | Integración calendario | 6 | 4 | 4 | 5 | **2.8** | Should | Reduce *no‑shows* y tiempos de agenda |
| US-05 | Scorecards | Rúbricas + agregación | 5 | 4 | 5 | 5 | **2.8** | Should | Mejora calidad de decisión |

> **Orden recomendado de implementación (sprints 1–3):** US‑02 → US‑03 → US‑06 → US‑01 → US‑04 → US‑05.

---

## 4. Tickets técnicos (User Story seleccionada: US‑02)
**US‑02: Ingesta/Parsing + dedupe + embeddings**

**Meta:** Centralizar CVs, evitar duplicados y habilitar búsqueda semántica (kNN + BM25).  
**Suposiciones:** PostgreSQL + `pgvector`, object storage, workers asíncronos, colas (Redis).  
**Estimación global US‑02:** 21 SP (Fibonacci).

### T-01 — Esquema de datos y migraciones
- **Descripción:** Crear tablas `Resume`, ampliar `Candidate`, índices (full-text, trigram, vector), constraints de `hash` único.
- **Criterios de aceptación:** migraciones aplican; índices presentes; seeds mínimas.
- **Estimación:** 3 SP
- **Asignación:** Back

### T-02 — Servicio de parsing (PDF/DOCX) + normalización
- **Descripción:** Worker que extrae texto y `parsed_json` (secciones: educación, experiencia, skills, idiomas).
- **Criterios de aceptación:** parsing ≥ 85% casos de prueba; manejo de errores; unit tests.
- **Estimación:** 5 SP
- **Asignación:** Back

### T-03 — Dedupe y *merge* asistido
- **Descripción:** Generar `dedupe_key` con combinación de email + nombre + similitud; flujo de *merge* con previsualización.
- **Criterios de aceptación:** umbral de similitud configurable; registro en `Application.Deduped`.
- **Estimación:** 3 SP
- **Asignación:** Back

### T-04 — Embeddings + indexación en `pgvector`
- **Descripción:** Servicio para generar embeddings de `Resume.text` y consulta kNN top‑k.
- **Criterios de aceptación:** latencia de indexación < 2s/doc en P95; reintentos; métricas.
- **Estimación:** 5 SP
- **Asignación:** Back/Infra

### T-05 — API de búsqueda híbrida (vector + BM25) con filtros
- **Descripción:** Endpoint `/search/candidates` con query parser, re‑rank y *highlights*; filtros por etapa/ubicación.
- **Criterios de aceptación:** resultados relevantes en casos de prueba; paginación; contrato API documentado.
- **Estimación:** 3 SP
- **Asignación:** Back

### T-06 — UI de carga y resultados semánticos
- **Descripción:** Pantalla para subir CVs y vista de resultados con *highlights*, chips de filtros y paginación.
- **Criterios de aceptación:** UX fluida; estados de carga/empty/error; accesibilidad AA.
- **Estimación:** 2 SP
- **Asignación:** Front/UX

**Dependencias cruzadas:** T‑01 → (T‑02, T‑04); T‑04 → T‑05; T‑02 → T‑03; T‑05 → T‑06.

---

## 5. Estimaciones
**Metodología:** Fibonacci en **Story Points** (SP) para tickets; referencia de capacidad 1 sprint ≈ 30–35 SP del equipo.  
**Suma SP US‑02:** 3 + 5 + 3 + 5 + 3 + 2 = **21 SP**.  
**Estimación macro por US (aprox.):**
- US‑02: 21 SP
- US‑03: 13 SP
- US‑06: 13 SP
- US‑01: 8 SP
- US‑04: 8 SP
- US‑05: 8 SP

> **Total MVP (sprints 1–3):** ~71–75 SP.

---

## 6. Conclusiones
- Priorizar **US‑02** primero maximiza impacto en *time‑to‑hire* al habilitar el *screening* eficiente.
- WSJF ayudó a balancear valor y tamaño, evitando sesgos por “features llamativas” pero costosas.
- La descomposición en tickets separa preocupaciones (datos, parsing, vector, API, UI) y facilita flujos paralelos Back/Front/Infra.

---

## 7. Prompts utilizados y análisis
Incluye los prompts completos en `prompts.md`. Resumen:
- **Prompt Maestro (este documento)**: obtuvo cobertura completa (US, Backlog, Tickets, Estimaciones). **Más efectivo** por especificidad y estructura.
- **Prompt Técnico (Search/Parsing)**: mejor para bajar a tickets y criterios técnicos detallados.
- **Prompt Estratégico (WSJF)**: útil para ordenar el backlog con foco en negocio.

**Por qué fue efectivo el Prompt Maestro:** define claramente *rol*, *contexto*, *plantillas*, *métricas* y *artefactos esperados*, reduciendo ambigüedad y *hallucinations* y mejorando consistencia de formato.

---

## 8. Anexos (glosario breve)
- **WSJF:** método de priorización por costo de retraso / tamaño.
- **SP (Story Points):** unidad relativa de esfuerzo.
- **kNN:** *k*-Nearest Neighbors para búsqueda vectorial.
- **BM25:** algoritmo clásico de ranking textual.
