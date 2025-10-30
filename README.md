# orchestrator-n8n

# Orquestador de Agentes de IA para Gestión de Incidentes (SOC)

Orquestador basado en **n8n** + utilidades JS/TS para **ingesta, triage (LLM/HITL) y respuesta** ante incidentes de ciberseguridad (mapeo MITRE, playbooks SOAR). Incluye laboratorio con **Wazuh Manager** y **Active Directory**, más un **CEC (Canonical Event Schema)** para estandarizar eventos.
![Flujo Preliminar monitoreo](./Orchestrator/docs/Screenshot%20%2833%29.png)


![Topología SOC](./Orchestrator/docs/Diagrama%20de%20seguridad%20de%20red%282%29.png)


## ✨ Objetivos
- Ingesta y normalización en CEC
- Triage automático con LLM + Human-in-the-Loop (HITL)
- Playbooks de respuesta (bloqueo IP, cuarentena, tickets)
- Métricas operativas (MTTD/MTTR, FPR) y trazabilidad

## 🧱 Arquitectura (resumen)
- **Agente de Monitoreo** (n8n): ingesta, normalización (CEC), deduplicación.
- **Agente de Análisis/Triage**: mapeo MITRE, router LLM vs fast-path.
- **Agente de Respuesta**: playbooks (firewall, tickets, notificaciones).
- **Orquestador/KPIs**: consolidación de métricas y auditoría.

> Ver detalles C1–C4 en [`docs/10-architecture-c4.md`](docs/10-architecture-c4.md).

## 🚀 Puesta en marcha (Docker)
```bash
cd docker
cp env.example .env
docker compose up -d
