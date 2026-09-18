# Cybersecurity

Repositorio personal orientado al desarrollo de conocimientos y habilidades prácticas en **Cybersecurity, SOC Operations, SIEM**.

El objetivo de este repositorio es documentar algunos conocimientos adquiridos y prácticas en laboratorios realizados.

---

## Áreas principales

### SOC Operations

Conceptos y procedimientos fundamentales para el trabajo de un SOC Analyst:

* SOC y funciones de un analista
* SIEM
* EDR / XDR
* SOAR
* IOC / TTP
* Alert Triage
* Incident Response
* Threat Intelligence
* MITRE ATT&CK

### Network Security

Fundamentos de redes necesarios para el análisis de eventos de seguridad:

* TCP/IP
* TCP / UDP
* DNS
* HTTP / HTTPS
* Protocolos frecuentes
* Puertos comunes

### Detection & Monitoring

Tecnologías y técnicas utilizadas para detectar y analizar actividad sospechosa:

* File Integrity Monitoring (FIM)
* Data Loss Prevention (DLP)
* YARA
* IDS / IPS
* Threat Intelligence
* Correlation Rules
* Log Analysis

### Query & Investigation

Práctica con herramientas y lenguajes utilizados para investigar eventos:

* Wazuh
* Splunk
* SPL
* consultas y filtros de eventos
* análisis de logs
* investigación de alertas

---

## SOC Cases

Casos prácticos desarrollados a partir de escenarios de investigación de seguridad.

Cada caso busca documentar el proceso completo:

```text
Alert
  ↓
Initial Triage
  ↓
Investigation
  ↓
Evidence
  ↓
Threat Intelligence
  ↓
MITRE ATT&CK
  ↓
Analysis
  ↓
Conclusion
```

Actualmente esta sección se encuentra en desarrollo.

---

## Labs

Laboratorios prácticos utilizados para implementar y comprobar los conceptos estudiados.

### Wazuh + Tailscale

Implementación de un entorno Wazuh utilizando agentes y conectividad mediante Tailscale.

[Ver laboratorio →](./cybersecurity/SOC/Labs/01-Wazuh-tailscale/)

### Wazuh SOC Lab

Laboratorio orientado a la construcción de un entorno SOC utilizando Wazuh, Suricata, Threat Intelligence y diferentes mecanismos de detección.

[Ver laboratorio →](./cybersecurity/SOC/Labs/02-wazuh-soc-lab/)

### Splunk

Laboratorios orientados al análisis de logs, búsquedas, consultas SPL, detección e investigación.

[Ver laboratorio →](./cybersecurity/SOC/Labs/03-splunk/)

---

## Documentation

La documentación se organiza separando:

**conocimientos**

Conceptos y fundamentos necesarios para comprender las tecnologías y procesos.

**Labs**

Implementaciones prácticas donde esos conceptos son utilizados.

**SOC Cases**

Investigaciones completas sobre incidentes o alertas.

**Playbooks**

Procedimientos estructurados para responder ante determinados tipos de alertas.

---

## Tools

Actualmente se trabaja con herramientas como:

* Wazuh
* Splunk
* Suricata
* YARA
* Tailscale
* Docker

La lista se irá ampliando a medida que se incorporen nuevas tecnologías al laboratorio.

---

## Repository Status

Este repositorio se encuentra en desarrollo.

