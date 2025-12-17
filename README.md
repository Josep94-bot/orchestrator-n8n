# orchestrator-n8n
## Orquestador de Agentes de IA para Gestión de Incidentes (SOC)

Prototipo de orquestación basado en **n8n** + utilidades **JS** para **ingesta, normalización (CEC), triage (LLM + HITL) y respuesta** ante incidentes de ciberseguridad (mapeo **MITRE ATT&CK**, playbooks y métricas como **MTTD/MTTR**).

> ⚠️ Proyecto orientado a desarrollo de prototipo de Laboratorio SOC. 

---

## 🧪 Laboratorio SOC (topología)
El prototipo se valida sobre un laboratorio conformado por un Cisco SG300-28 Small Bussiness con **Wazuh Manager**, **Active Directory**, **n8n**, endpoints Win11/Ubuntu y un IDS Suricata.

![Topología SOC](./Orchestrator/docs/Topología%20Laboratorio%20SOC.png)

---

## ✨ Objetivos
- **Ingesta y normalización** de eventos a un **CEC (Canonical Event Schema)**
- **Triage automático** con LLM + **Human-in-the-Loop (HITL)**
- **Playbooks de respuesta** (bloqueo IP, notificaciones, artefactos)
- **Métricas operativas** (MTTD/MTTR) 

---

## 🧱 Arquitectura (resumen)
El sistema está compuesto por **“agentes”** implementados como workflows en n8n, conectados entre sí mediante nodos **When Executed by Another Workflow** (llamadas tipo *tool*).  
El **evento CEC** es el “contrato” compartido: cada agente recibe un objeto evento, agrega contexto/decisiones y lo retorna enriquecido.

> Ver detalles C1–C4 en: [`docs/10-architecture-c4.md`](docs/10-architecture-c4.md)

---

## 🔌 Cómo se conectan los flujos y los agentes (end-to-end)
**Cadena principal (simplificada):**

1) **Ingesta (Wazuh → CEC)**  
2) **Monitoreo / enrutamiento** (deduplicación, decisión de triage)  
3) **Análisis (LLM + reglas + HITL opcional)**  
4) **Plan de respuesta** (qué hacer)  
5) **Ejecución de respuesta** (hacerlo mediante acciones con: firewall/tickets/notificaciones)  
6) **Cierre y métricas** (MTTD/MTTR)

---

## 🛰️ Agente de Monitoreo (ingesta + normalización CEC)
Este flujo recibe eventos (Webhook / Wazuh), asigna timestamps, normaliza al **CEC**, persiste en base de datos y dispara el “tool” de monitoreo para continuar el pipeline.

![Flujo Monitoreo](./Orchestrator/docs/monitorin.jpeg)

**Puntos clave del flujo:**
- `Ingesta Wazuh Event` → entrada (POST)
- `CEC Normalization ...` → transforma a CEC y prepara triage
- `INSERT cec_events` → persistencia del evento normalizado
- `Call 'tool_monitor_event'` → invoca el siguiente agente

---

## 🧠 Agente de Análisis / Triage
Este workflow se ejecuta “como herramienta” desde otros flujos. Evalúa si el evento requiere análisis profundo (LLM) o pasa por fast-path, registra métricas y notifica al SOC.

![Flujo Análisis/Triage](./Orchestrator/docs/analisis.jpeg)

**Lectura del diagrama:**
- `If route_to_analysis` decide entre:
  - **Passthrough** (no amerita análisis)  
  - **Ruta LLM** (análisis con `OpenAI Chat Model`)
- `ts_analysis_start` / `ts_analysis_end + tta` → trazabilidad y tiempos
- `INSERT fr_metrics(event)` → métricas (p. ej., FPR/feedback loop)
- Notificación a `SOC Team` (Telegram) + `Return final event`

---

## 🎛️ Orquestador (con n8n Workflow Tool )
Este agente actúa como “cerebro” de coordinación: recibe una solicitud (chat/comando), selecciona herramientas y encadena agentes (**Monitoring**, **Analysis**, **ResponsePlan**, **Execute**), y devuelve una salida al operador.

![Orquestador](./Orchestrator/docs/Orquestador.jpeg)

**Idea principal:** el orquestador *no ejecuta todo dentro de un único flujo gigante*, sino que **llama herramientas** (sub-workflows y workflow tools) para mantener:
- modularidad,
- observabilidad por etapa,
- reusabilidad (mismo “tool” para distintos disparadores).

---

## 🧩 Agente de Plan de Respuesta (ResponsePlan)
Genera un **plan** (acciones recomendadas, prioridad, justificación, mapeo MITRE, riesgos) y lo fusiona sobre el objeto evento CEC.

![ResponsePlan](./Orchestrator/docs/responseplan.jpeg)

**Salida típica:** `event.response_plan = { actions[], approvals, notes, mitre, confidence }`

---

## ⚡ Agente de Respuesta (ResponseExecute + HITL)
Ejecuta playbooks en función del plan: puede requerir aprobación humana (**HITL**), enrutar acciones (bloquear IP, notificar, generar artefactos) y consolidar resultados.

![Flujo Respuesta + HITL](./Orchestrator/docs/ResponseExecute.jpeg)

**Elementos visibles del flujo:**
- `HITL Decision` → decide si se requiere aprobación
- `Message HITL` → solicita confirmación
- `Switch Action Router` → enruta por tipo de acción (ej. bloquear IP)
- Integración Wazuh (`Get Token`, `Wazuh/firewall`)
- Notificaciones y artefactos
- `ts_response_end + mttr` → cierre de tiempos / `Return Final event`

---

## 🔧 “Tools” como sub-workflows (wrappers)
Para estandarizar el contrato de entrada/salida, varios tools usan el patrón:
1) `tool_input_unwrap` (normaliza input)
2) `Call <Workflow>` (ejecuta herramienta real)
3) `return_tool_output` (devuelve respuesta al orquestador)

Ejemplos:

**Wrapper de ResponsePlan**
![Wrapper ResponsePlan](./Orchestrator/docs/Response.jpeg)

**Wrapper de ResponseExecute**
![Wrapper ResponseExecute](./Orchestrator/docs/ResponseExecute.jpeg)

**Wrapper de Execute (ejecutor genérico)**
![Wrapper Execute](./Orchestrator/docs/Execute.jpeg)

---

## 🗂️ Estructura del repositorio (sugerida)
- `Orchestrator/workflows/` → exports de workflows n8n (agentes/tools)
- `Orchestrator/docs/` → capturas y diagramas (los `.jpeg/.png` de este README)
- `docs/10-architecture-c4.md` → arquitectura C4
- `src/` o `utils/` → utilidades JS/TS (helpers, CEC mapping, etc.)

---

## 📏 Métricas y trazabilidad
El prototipo instrumenta timestamps y persistencia para medir, por ejemplo:
- **MTTD**: desde `ts_ingest` hasta detección/triage útil
- **TTA**: tiempo hasta análisis/acción sugerida (`ts_analysis_*`)
- **MTTR**: desde inicio de respuesta hasta cierre (`ts_response_*`)
- **FPR / feedback**: eventos marcados como falsos positivos o re-clasificados

---

## 🚀 Cómo ejecutar (alto nivel)
1. Levanta **n8n** (docker o local).
2. Importa workflows desde `Orchestrator/workflows/`.
3. Configura credenciales/conexiones:
   - Webhook (Wazuh / input)
   - DB (para `cec_events`, `fr_metrics`, etc.)
   - Telegram (SOC / HITL)
   - API Wazuh (token + acciones)
4. Dispara un evento de prueba (Wazuh o mock) y revisa:
   - evento normalizado en DB,
   - notificación al SOC,
   - plan de respuesta,
   - ejecución (con o sin HITL),
   - métricas.

> Si quieres, agrega aquí un `.env.example` con variables tipo `WAZUH_URL`, `WAZUH_USER`, `WAZUH_PASS`, `TELEGRAM_TOKEN`, `DB_DSN`, etc.

---

## 📸 Créditos
Diagramas y capturas del laboratorio y flujos n8n incluidos en `Orchestrator/docs/`.



