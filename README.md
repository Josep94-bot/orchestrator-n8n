# orchestrator-n8n

# Orquestador de Agentes de IA para Gestión de Incidentes (SOC)


Orquestador basado en **n8n** + utilidades JS/TS para **ingesta, triage (LLM/HITL) y respuesta** ante incidentes de ciberseguridad (mapeo MITRE, playbooks SOAR). Incluye laboratorio con **Wazuh Manager** y **Active Directory**, más un **CEC (Canonical Event Schema)** para estandarizar eventos.
![Flujo Preliminar monitoreo](./Orchestrator/docs/Screenshot%20%2833%29.png)

![Topología SOC](./Orchestrator/docs/Topología%20Laboratorio%20SOC.png)

![Flujo de trabajo](./Orchestrator/workflows/flujo%20trabajo.jpeg)



## ✨ Objetivos
- Ingesta y normalización en CEC
- Triage automático con LLM + Human-in-the-Loop (HITL)
- Playbooks de respuesta (bloqueo IP, cuarentena, tickets)
- Métricas operativas (MTTD/MTTR, FPR) y trazabilidad

## 🧱 Arquitectura (resumen)
- **Agente de Monitoreo y Análisis/Triage** (n8n): ingesta, normalización (CEC), deduplicación; mapeo MITRE, router LLM vs fast-path.
- **Agente de Respuesta**: playbooks (firewall, tickets, notificaciones).
- **Orquestador/KPIs**: consolidación de métricas y auditoría.

> Ver detalles C1–C4 en [`docs/10-architecture-c4.md`](docs/10-architecture-c4.md).


