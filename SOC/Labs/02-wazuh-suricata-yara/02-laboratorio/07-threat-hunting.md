# :mag: 7. Threat Hunting

## :dart: Objetivo

En esta etapa se realiza el **Threat Hunting** sobre las evidencias generadas durante el laboratorio.

La hipótesis planteada es que el host que generó una alerta de **Threat Intelligence** puede contener artefactos relacionados con **PurpleWolf**.

Para investigarlo se utilizan tres fuentes de evidencia:

* **Threat Intelligence (TI):** identifica el indicador de compromiso relacionado con una IP.
* **FIM:** registra cambios realizados sobre archivos del directorio de evidencias.
* **YARA:** busca patrones asociados a los artefactos de PurpleWolf.

El objetivo no es generar una nueva detección, sino **comparar y relacionar las evidencias obtenidas** para determinar qué ocurrió sobre el host.

---

## :mag: Flujo de Threat Hunting

El análisis parte de una alerta de Threat Intelligence y continúa con la búsqueda de evidencias en el host:

```text
                 Threat Intelligence
                         │
                         ▼
                Alerta sobre el host
                         │
                         ▼
              ┌─────────────────────┐
              │   Threat Hunting    │
              └─────────────────────┘
                    │           │
             ┌──────┘           └──────┐
             ▼                         ▼
            FIM                       YARA
             │                         │
             ▼                         ▼
      Cambios en archivos       Patrones encontrados
             │                         │
             └──────────┬──────────────┘
                        ▼
                Correlación manual
                        │
                        ▼
                 Evidencia final
```

> **Importante:** FIM y YARA funcionan de manera independiente.
> FIM no ejecuta YARA. El temporizador de `systemd` ejecuta las búsquedas YARA de forma independiente.

---

## :mag: 1. Hipótesis de Threat Hunting

La hipótesis del laboratorio es:

> El host que generó una alerta de Threat Intelligence contiene artefactos relacionados con PurpleWolf.

Para comprobar esta hipótesis se utilizan las evidencias generadas previamente:

| Fuente              | Qué aporta                                    |
| ------------------- | --------------------------------------------- |
| Threat Intelligence | Indicador asociado al host                    |
| FIM                 | Cambios realizados sobre archivos             |
| YARA                | Coincidencias con patrones de PurpleWolf      |
| Wazuh               | Centralización y visualización de los eventos |

El análisis busca determinar si existen evidencias relacionadas temporalmente y sobre los mismos archivos.

---

# :mag: 2. Analizar eventos de FIM

El primer paso es consultar los eventos generados por **File Integrity Monitoring**.

Desde **VM1**, dentro del directorio del Wazuh Docker:

```bash
cd /opt/wazuh-docker/single-node
```

Consultar los eventos FIM del agente `cyberrange-suricata`:

```bash
sudo docker compose exec -T wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg file "$FIM_FILE" 'select(
    .agent.name=="cyberrange-suricata" and .syscheck.path==$file
  )' | tail -3
```

El evento permite comprobar que Wazuh registró el archivo mediante **Syscheck/FIM**.

La información relevante incluye:

* agente que generó el evento;
* ruta del archivo;
* tipo de modificación;
* timestamp;
* información de integridad disponible.

### :bulb: Qué buscamos

En esta etapa no se determina que el archivo sea malicioso.

El objetivo es comprobar que **el archivo fue creado o modificado** y que ese cambio quedó registrado por FIM.

---

# :mag: 3. Analizar eventos de YARA

A continuación se analizan los eventos producidos por la búsqueda YARA.

La regla personalizada utilizada en el laboratorio es:

```text
rule.id:100600
```

Desde VM1 se pueden consultar directamente los eventos de YARA:

```bash
sudo docker compose exec -T wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c 'select(
    .agent.name=="cyberrange-suricata" and
    .rule.id=="100600" and
    .rule.level==12
  )' | tail -1 | jq .
```

La alerta permite identificar:

* el agente donde se encontró la coincidencia;
* el archivo analizado;
* la regla YARA asociada;
* el nivel de alerta;
* la información registrada por el script.

En este laboratorio, una alerta con:

```text
rule.id = 100600
```

corresponde a la detección generada por la integración de **YARA con Wazuh**.

---

# :mag: 4. Comparar FIM y YARA sobre el mismo archivo

Uno de los puntos importantes del Threat Hunting es comprobar si diferentes fuentes de evidencia apuntan al mismo artefacto.

Para ello, se utiliza el archivo generado durante la prueba final:

```bash
$FINAL_FILE
```

### Consultar FIM

```bash
sudo docker compose exec -T wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg file "$FINAL_FILE" 'select(
    .agent.name=="cyberrange-suricata" and .syscheck.path==$file
  )' | tail -3
```

### Consultar YARA

```bash
sudo docker compose exec -T wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg file "$FINAL_FILE" 'select(
    .agent.name=="cyberrange-suricata" and .data.file==$file
    and .rule.id=="100600" and .rule.level==12
  )' | tail -1 | jq .
```

La comparación permite comprobar que:

```text
                  Archivo
                     │
             ┌───────┴───────┐
             ▼               ▼
            FIM             YARA
             │               │
             ▼               ▼
       cambio detectado   patrón detectado
             │               │
             └───────┬───────┘
                     ▼
               misma evidencia
```

Esto es útil porque las dos herramientas responden preguntas diferentes:

* **FIM:** ¿el archivo cambió?
* **YARA:** ¿el contenido del archivo coincide con un patrón determinado?

Una misma evidencia puede producir ambos tipos de eventos.

---

# :mag: 5. Comparar con Threat Intelligence

El laboratorio también permite comparar las evidencias obtenidas con la alerta inicial de **Threat Intelligence**.

En Wazuh Dashboard se pueden utilizar las siguientes consultas.

### Threat Intelligence

```text
agent.name:"cyberrange-suricata" AND rule.id:100500
```

### YARA

```text
agent.name:"cyberrange-suricata" AND rule.id:100600
```

### FIM

```text
agent.name:"cyberrange-suricata" AND rule.groups:syscheck
```

De esta manera se pueden observar las tres fuentes de evidencia sobre el mismo agente.

---

# :mag: 6. Qué comparar

Al analizar los eventos, se deben comparar principalmente:

| Elemento  | Pregunta                                        |
| --------- | ----------------------------------------------- |
| Agente    | ¿Los eventos pertenecen al mismo host?          |
| Timestamp | ¿Cuándo ocurrió cada evento?                    |
| Ruta      | ¿Los eventos hacen referencia al mismo archivo? |
| Hash      | ¿Coincide la evidencia de integridad?           |
| Regla     | ¿Qué mecanismo produjo cada alerta?             |
| Indicador | ¿Qué evidencia aportó Threat Intelligence?      |

El análisis permite construir una línea temporal de lo ocurrido en el host.

Por ejemplo:

```text
TI
│
│  Alerta sobre el host
▼
Host investigado
│
├── FIM → registra cambios en archivos
│
└── YARA → encuentra patrones asociados
             │
             ▼
       Evidencia adicional
```

---

# :warning: 7. No existe una correlación automática entre TI y YARA

Un punto importante del laboratorio es que **Threat Intelligence y YARA son mecanismos independientes**.

Que un host genere una alerta de TI no significa automáticamente que YARA se ejecute sobre ese host.

Del mismo modo:

* una alerta de FIM no ejecuta YARA;
* una coincidencia YARA no significa automáticamente que exista un incidente;
* una alerta de TI no confirma por sí sola la presencia de un artefacto.

La relación entre estas evidencias se realiza durante el proceso de **Threat Hunting**.

```text
Threat Intelligence
        │
        ▼
     Evidencia
        │
        │       FIM
        │        │
        │        ▼
        └──► cambios
        │
        │       YARA
        │        │
        │        ▼
        └──► coincidencias
                 │
                 ▼
        Análisis del analista
```

El analista compara las evidencias para determinar si existe una relación entre ellas.

---

# :mag: 8. Visualización final en Wazuh

Desde **Wazuh Dashboard** se pueden realizar las tres búsquedas:

### YARA

```text
agent.name:"cyberrange-suricata" AND rule.id:100600
```

### FIM

```text
agent.name:"cyberrange-suricata" AND rule.groups:syscheck
```

### Threat Intelligence

```text
agent.name:"cyberrange-suricata" AND rule.id:100500
```

La comparación final debe centrarse en:

```text
┌───────────────────────────────┐
│ Threat Intelligence           │
│ rule.id: 100500               │
└───────────────┬───────────────┘
                │
                │ host / timestamp
                ▼
┌───────────────────────────────┐
│ FIM                           │
│ rule.groups: syscheck         │
└───────────────┬───────────────┘
                │
                │ archivo / timestamp
                ▼
┌───────────────────────────────┐
│ YARA                          │
│ rule.id: 100600               │
└───────────────────────────────┘
```

---

# :clipboard: 9. Resultado del Threat Hunting

Al finalizar el análisis se debe poder identificar:

* qué host generó la alerta inicial de Threat Intelligence;
* qué cambios de archivos registró FIM;
* qué archivos produjeron coincidencias YARA;
* cuándo ocurrió cada evento;
* qué hashes fueron registrados;
* qué evidencias pertenecen al mismo archivo;
* qué relación temporal existe entre los eventos.

El resultado es una **visión conjunta de la evidencia disponible en el host**, utilizando diferentes mecanismos de detección y monitoreo.

---

# :white_check_mark: 10. Checklist final

* [ ] Identificar el host asociado a la alerta de Threat Intelligence.
* [ ] Consultar los eventos FIM del agente `cyberrange-suricata`.
* [ ] Identificar los archivos modificados.
* [ ] Consultar las alertas YARA `100600`.
* [ ] Comparar las rutas de los archivos.
* [ ] Comparar timestamps.
* [ ] Comparar hashes cuando estén disponibles.
* [ ] Revisar las evidencias desde Wazuh Dashboard.
* [ ] Comparar TI, FIM y YARA.
* [ ] Documentar las relaciones encontradas.

---

# :dart: Resultado del laboratorio

Con esta etapa se completa el flujo de investigación:

```text
Threat Intelligence
        │
        ▼
Identificación del host
        │
        ▼
   Threat Hunting
      /       \
     /         \
   FIM        YARA
    │            │
    ▼            ▼
Cambios       Patrones
    │            │
    └──────┬─────┘
           ▼
     Correlación de
        evidencias
           │
           ▼
      Wazuh Dashboard
```

El laboratorio demuestra cómo diferentes fuentes de telemetría pueden utilizarse durante una investigación para construir contexto sobre un mismo host y sus archivos.

El Threat Hunting, en este escenario, consiste en **formular una hipótesis, buscar evidencias y contrastar los resultados de TI, FIM y YARA**.
