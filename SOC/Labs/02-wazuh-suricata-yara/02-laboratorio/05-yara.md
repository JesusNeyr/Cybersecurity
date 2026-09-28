# :mag: 05 - YARA

## :dart: Objetivo

En esta etapa incorporamos **YARA** al laboratorio para realizar una búsqueda basada en patrones sobre archivos del host.

La hipótesis de la práctica es:

> El host que generó la alerta de Threat Intelligence puede contener archivos con indicadores asociados al artefacto simulado **PurpleWolf**.

---

# :building_construction: 1. Arquitectura de YARA

YARA analiza **archivos del host**:

```text
Archivos
   ↓
Regla YARA
   ↓
Match / No Match
   ↓
Script de búsqueda
   ↓
cybersoc-yara.log
   ↓
Wazuh Agent
   ↓
Wazuh Manager
   ↓
Regla 100600
   ↓
Dashboard
```

La arquitectura de esta etapa queda:

```text
                    VM2 — CyberRange
┌──────────────────────────────────────────────────┐
│                                                  │
│  /opt/cybersoc-hunting/evidence/                 │
│                 │                                │
│                 ↓                                │
│       cybersoc_purplewolf.yar                    │
│                 │                                │
│                 ↓                                │
│             YARA scan                             │
│                 │                                │
│             Match                                │
│                 │                                │
│                 ↓                                │
│      cybersoc-yara-scan.sh                       │
│                 │                                │
│                 ↓                                │
│       /var/log/cybersoc-yara.log                 │
│                 │                                │
│                 ↓                                │
│          Wazuh Agent                             │
└─────────────────┼────────────────────────────────┘
                  │
                  ↓
        ┌─────────────────────┐
        │   Wazuh Manager     │
        │                     │
        │    Rule 100600      │
        │      Level 12       │
        └──────────┬──────────┘
                   │
                   ↓
             Dashboard
```

---

# :brain: 2. ¿Qué busca YARA?

Nuestra regla buscará cuatro patrones que definamos:

```text
PurpleWolf
172.30.0.20
CYBERSOC-LAB
PurpleWolf-C2
```

Pero la regla **no exige las cuatro**.

La condición será:

```text
3 of them 4
```

Por lo tanto:

```text
4 coincidencias → MATCH
3 coincidencias → MATCH
2 coincidencias → NO MATCH
1 coincidencia  → NO MATCH
0 coincidencias → NO MATCH
```
---

# :wrench: 3. Instalar YARA

Esta parte se realiza en **VM2 — CyberRange**.

```bash
sudo apt update
sudo apt install -y yara jq
```

### Verificar la instalación

```bash
yara --version
```

---

# :file_folder: 4. Crear los directorios de trabajo donde estara la regla

```bash
sudo mkdir -p /var/ossec/etc/yara/rules
```

Crear el directorio donde colocaremos los archivos que serán analizados:

```bash
sudo mkdir -p /opt/cybersoc-hunting/evidence
```

La estructura será:

```text
/var/ossec/etc/yara/
└── rules/
    └── cybersoc_purplewolf.yar

/opt/cybersoc-hunting/
└── evidence/
    ├── normal.txt
    └── purplewolf_update.dat
```

---

# :mag: 5. Crear la regla YARA

Crear el archivo:

```bash
sudo tee /var/ossec/etc/yara/rules/cybersoc_purplewolf.yar >/dev/null <<'EOF'
rule CYBERSOC_PurpleWolf_Artifact
{
    meta:
        description = "Artefactos simulados PurpleWolf del lab CyberSOC"
        author = "CyberSOC"
        severity = "high"

    strings:
        $campaign = "PurpleWolf" ascii nocase
        $c2       = "172.30.0.20" ascii
        $marker   = "CYBERSOC-LAB" ascii
        $agent    = "PurpleWolf-C2" ascii nocase

    condition:
        3 of them
}
EOF
```

---

# :mag_right: 6. Entender la regla

### Nombre de la regla

```text
CYBERSOC_PurpleWolf_Artifact
```

Este será el identificador que posteriormente aparecerá en los resultados de YARA.

### Metadatos

```yara
meta:
    description = "Artefactos simulados PurpleWolf del lab CyberSOC"
    author = "CyberSOC"
    severity = "high"
```

Estos datos describen la regla.

---

## :strings: 6.1 Cadenas buscadas

```yara
strings:
    $campaign = "PurpleWolf" ascii nocase
    $c2       = "172.30.0.20" ascii
    $marker   = "CYBERSOC-LAB" ascii
    $agent    = "PurpleWolf-C2" ascii nocase
```

Tenemos cuatro patrones:

| Variable    | Patrón          | Función                                |
| ----------- | --------------- | -------------------------------------- |
| `$campaign` | `PurpleWolf`    | Identifica la campaña simulada         |
| `$c2`       | `172.30.0.20`   | Identifica el C2 ficticio              |
| `$marker`   | `CYBERSOC-LAB`  | Identifica el contexto del laboratorio |
| `$agent`    | `PurpleWolf-C2` | Identifica el User-Agent ficticio      |

`nocase` hace que la comparación no distinga entre mayúsculas y minúsculas.

pueden coincidir con `$campaign`.

---

# :gear: 7. Condición de detección

La condición es:

```yara
condition:
    3 of them
```

Esto significa que YARA necesita encontrar **al menos tres de las cuatro cadenas definidas**.

Por ejemplo:

```text
CYBERSOC-LAB
Campaign: PurpleWolf
C2: 172.30.0.20
```

contiene tres patrones, de 4 hace match.

---

# :white_check_mark: 8. Validar la sintaxis de la regla

```bash
sudo yara \
  /var/ossec/etc/yara/rules/cybersoc_purplewolf.yar \
  /dev/null
```

Resultado esperado:

```text
sin errores
```
---

# :test_tube: 9. Crear un archivo normal

Primero hacemos un **control negativo**.

Crear:

```bash
echo "Un archivo de CyberSOC" |
  sudo tee /opt/cybersoc-hunting/evidence/normal.txt
```

Analizarlo:

```bash
sudo yara \
  /var/ossec/etc/yara/rules/cybersoc_purplewolf.yar \
  /opt/cybersoc-hunting/evidence/normal.txt
```

Resultado esperado:

```text
sin salida
Porque no matchea con las condiciones 
```

---

# :detective: 10. Crear el artefacto simulado

Ahora creamos el archivo que debe generar un `match`.

```bash
sudo tee /opt/cybersoc-hunting/evidence/purplewolf_update.dat >/dev/null <<'EOF'
CYBERSOC-LAB
Application: System Update Service
Campaign: PurpleWolf
C2: 172.30.0.20
User-Agent: PurpleWolf-C2
Status: ACTIVE
EOF
```

El archivo contiene los cuatro patrones
Por lo tanto:

```text
Debe existir un MATCH
```

---

# :mag: 11. Ejecutar YARA contra el artefacto

```bash
sudo yara -s \
  /var/ossec/etc/yara/rules/cybersoc_purplewolf.yar \
  /opt/cybersoc-hunting/evidence/purplewolf_update.dat
```

El parámetro:

```text
-s
```

hace que YARA muestre también las cadenas que produjeron la coincidencia.

Resultado esperado:

```text
CYBERSOC_PurpleWolf_Artifact
```

acompañado por las cadenas coincidentes.

---

# :hash: 12. Calcular el SHA-256

Guardar también el hash del archivo:

```bash
sha256sum \
  /opt/cybersoc-hunting/evidence/purplewolf_update.dat
```

El hash será utilizado posteriormente por el script de búsqueda para identificar exactamente el archivo analizado.

El flujo será:

```text
Archivo
   ↓
YARA Match
   ↓
Ruta del archivo
   +
Regla YARA
   +
SHA-256
   ↓
Evento JSON
```

---

# :page_facing_up: 13. Crear el log de resultados de YARA

Para que pueda ser recolectada por Wazuh.

Crear:

```bash
sudo touch /var/log/cybersoc-yara.log
```

Asignar propietario:

```bash
sudo chown root:wazuh /var/log/cybersoc-yara.log
```

Configurar permisos:

```bash
sudo chmod 640 /var/log/cybersoc-yara.log
```

El archivo será utilizado como:

```text
YARA
 ↓
cybersoc-yara.log
 ↓
Wazuh Agent
```

---

# :gear: 14. Configurar Wazuh Agent para leer el log YARA

Creamos un backup.

```bash
sudo cp -p /var/ossec/etc/ossec.conf \
  "/var/ossec/etc/ossec.conf.bak-yara-$(date +%Y%m%dT%H%M%S)"
```

Editar:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Antes del último:

```xml
</ossec_config>
```

agregar:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/cybersoc-yara.log</location>
</localfile>
```

### ¿Qué estamos haciendo?

Estamos diciéndole al Agent:

> El archivo `/var/log/cybersoc-yara.log` contiene eventos JSON que deben ser recolectados.

---

# :white_check_mark: 15. Validar el Wazuh Agent

Ejecutar:

```bash
sudo /var/ossec/bin/wazuh-agentd -t
```

Y:

```bash
sudo /var/ossec/bin/wazuh-logcollector -t
```

Ambas validaciones deben finalizar sin errores.

Después:

```bash
sudo systemctl restart wazuh-agent
```

Comprobar:

```bash
sudo systemctl status wazuh-agent --no-pager
```

Y revisar:

```bash
sudo grep -Ei \
  'connected|cybersoc-yara|error' \
  /var/ossec/logs/ossec.log | tail -30
```

---

# :computer: 16. Crear el script de búsqueda YARA

Crear:

```bash
sudo tee /usr/local/bin/cybersoc-yara-scan.sh >/dev/null <<'EOF'
#!/bin/bash
set -euo pipefail

RULES="/var/ossec/etc/yara/rules/cybersoc_purplewolf.yar"
TARGET="/opt/cybersoc-hunting/evidence"
LOG="/var/log/cybersoc-yara.log"
RUN_ID="yara-$(date -u +%Y%m%dT%H%M%S)-$"

while IFS= read -r -d '' FILE; do
    MATCHES=$(/usr/bin/yara "$RULES" "$FILE")
    [ -n "$MATCHES" ] || continue
    RULE=$(printf '%s\n' "$MATCHES" | awk 'NR==1 {print $1}')
    SHA256=$(sha256sum -- "$FILE" | awk '{print $1}')

    jq -cn \
      --arg event_type "yara_match" \
      --arg rule "$RULE" \
      --arg file "$FILE" \
      --arg sha256 "$SHA256" \
      --arg severity "high" \
      --arg run_id "$RUN_ID" \
      --arg scanned_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      '{event_type:$event_type,yara_rule:$rule,file:$file,
        sha256:$sha256,severity:$severity,run_id:$run_id,
        scanned_at:$scanned_at}' >> "$LOG"
done < <(find "$TARGET" -type f -print0)

printf 'RUN_ID=%s\n' "$RUN_ID"
EOF
```
---

# :lock: 17. Configurar permisos del script

```bash
sudo chown root:root \
  /usr/local/bin/cybersoc-yara-scan.sh
```

```bash
sudo chmod 750 \
  /usr/local/bin/cybersoc-yara-scan.sh
```

---

# :white_check_mark: 18. Validar la sintaxis Bash

```bash
sudo bash -n \
  /usr/local/bin/cybersoc-yara-scan.sh
```

Si no devuelve nada todo esta ok.
---

# :test_tube: 19. Ejecutar el script manualmente

Ejecutar:

```bash
sudo /usr/local/bin/cybersoc-yara-scan.sh
```

El script recorrerá:

```text
/opt/cybersoc-hunting/evidence
```

y ejecutará YARA contra cada archivo.

---

# :page_facing_up: 20. Comprobar el evento generado

Después de ejecutar el script:

```bash
sudo tail -1 /var/log/cybersoc-yara.log | jq .
```

El resultado esperado es un JSON con información similar a:

```json
{
  "event_type": "yara_match",
  "yara_rule": "CYBERSOC_PurpleWolf_Artifact",
  "file": "/opt/cybersoc-hunting/evidence/purplewolf_update.dat",
  "sha256": "...",
  "severity": "high",
  "run_id": "...",
  "scanned_at": "..."
}
```

Este evento es el puente entre **YARA** y **Wazuh**.

---

# :arrows_counterclockwise: 21. Flujo YARA → Wazuh

Hasta este punto tenemos:

```text
Archivo
   ↓
YARA
   ↓
CYBERSOC_PurpleWolf_Artifact
   ↓
Script
   ↓
JSON
   ↓
/var/log/cybersoc-yara.log
   ↓
Wazuh Agent
   ↓
Wazuh Manager
```

Todavía falta que Wazuh tenga una regla capaz de interpretar este evento como una alerta de seguridad.

---

# :rotating_light: 22. Crear la regla Wazuh `100600`

La regla personalizada será:

```text
100600

nivel: 12
```

Primero comprobamos que el ID no exista:

```bash
cd /opt/wazuh-docker/single-node

sudo docker compose exec -T wazuh.manager \
  sh -c 'grep -R -n "id=\"100600\"" /var/ossec/etc/rules || true'
```

Si no existe, crear:

```bash
sudo docker compose exec -u 0 -T wazuh.manager \
  sh -c 'cat > /var/ossec/etc/rules/cybersoc_yara.xml' <<'EOF'
<group name="cybersoc_yara,">
  <rule id="100600" level="12">
    <decoded_as>json</decoded_as>
    <field name="event_type">^yara_match$</field>
    <field name="yara_rule">^CYBERSOC_PurpleWolf_Artifact$</field>
    <description>CYBERSOC - YARA detected PurpleWolf artifact</description>
    <group>yara,threat_hunting,malware_detection,</group>
  </rule>
</group>
EOF
```

---

# :brain: 23. Entender la regla `100600`

La regla requiere:

### Evento JSON

```text
event_type = yara_match
```

y:

```text
yara_rule = CYBERSOC_PurpleWolf_Artifact
```

La regla no busca directamente el archivo.

El script ya realizó la búsqueda con YARA.

Wazuh recibe el **resultado estructurado de esa búsqueda**.

---

# :lock: 24. Configurar permisos de la regla

```bash
sudo docker compose exec -u 0 -T wazuh.manager sh -c '
chown wazuh:wazuh /var/ossec/etc/rules/cybersoc_yara.xml
chmod 660 /var/ossec/etc/rules/cybersoc_yara.xml
'
```

---

# :white_check_mark: 25. Validar la configuración de Wazuh

```bash
sudo docker compose exec -T wazuh.manager \
  /var/ossec/bin/wazuh-analysisd -t
```

**Solo reiniciar el Manager si la validación termina sin errores.**

```bash
sudo docker compose restart wazuh.manager
```

Después comprobar:

```bash
sudo docker compose ps
```

El `wazuh.manager` debe estar levantado.

---

# :mag: 26. Validar la regla con `wazuh-logtest`

En VM1:

```bash
sudo docker compose exec -it \
  wazuh.manager \
  /var/ossec/bin/wazuh-logtest
```

Pegar:

```json
{"event_type":"yara_match","yara_rule":"CYBERSOC_PurpleWolf_Artifact","file":"/opt/cybersoc-hunting/evidence/purplewolf_update.dat","sha256":"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa","severity":"high","run_id":"logtest"}
```

Resultado esperado:

```text
id: '100600'
level: '12'
```

Esto demuestra que Wazuh reconoce correctamente el evento generado por YARA.

> `wazuh-logtest` prueba el decodificador y las reglas. No genera una alerta real en `alerts.json`.

Salir:

```text
Ctrl + C
```

---

# :detective: 27. Ejecutar una búsqueda YARA real

Modificar el artefacto para generar una nueva evidencia:

```bash
printf 'HUNT-ID: %s\n' \
  "$(date -u +%Y%m%dT%H%M%S)" |
  sudo tee -a \
  /opt/cybersoc-hunting/evidence/purplewolf_update.dat
```

Ejecutar:

```bash
sudo /usr/local/bin/cybersoc-yara-scan.sh
```

Verificar el último evento:

```bash
sudo tail -1 /var/log/cybersoc-yara.log | jq .
```

Y calcular nuevamente el hash:

```bash
sha256sum \
  /opt/cybersoc-hunting/evidence/purplewolf_update.dat
```

El SHA-256 que aparece en el JSON debe coincidir con el calculado sobre el archivo.

---

# :bar_chart: 29. Verificar la alerta real en Wazuh

En VM1:

```bash
cd /opt/wazuh-docker/single-node
```

El script muestra un:

```text
RUN_ID=...
```

Guardar ese valor y utilizarlo:

```bash
read -r -p \
  "Pega el valor RUN_ID de VM 2 (sin RUN_ID=): " \
  YARA_RUN
```

Buscar la alerta:

```bash
sudo docker compose exec -T wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg run "$YARA_RUN" 'select(
    .agent.name=="cyberrange-suricata" and
    .data.run_id==$run and
    .rule.id=="100600" and
    .rule.level==12
  )' |
  tail -1 |
  jq .
```

Resultado esperado:

```text
rule.id    = 100600
rule.level = 12
```

Además, deben aparecer datos como:

```text
data.file
data.sha256
data.run_id
```

---

# :alarm_clock: 30. Automatizar la búsqueda con `systemd`

```text
sudo /usr/local/bin/cybersoc-yara-scan.sh
```

Ahora crearemos un servicio `oneshot` y un `timer` para ejecutarlo periódicamente.


---

## 30.1 Crear el servicio

En VM2:

```bash
sudo tee /etc/systemd/system/cybersoc-yara.service >/dev/null <<'EOF'
[Unit]
Description=CyberSOC YARA Hunting Scan

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cybersoc-yara-scan.sh
EOF
```

---

## 30.2 Crear el timer

```bash
sudo tee /etc/systemd/system/cybersoc-yara.timer >/dev/null <<'EOF'
[Unit]
Description=CyberSOC periodic YARA hunting

[Timer]
OnBootSec=1min
OnUnitActiveSec=1min
AccuracySec=1s
Unit=cybersoc-yara.service

[Install]
WantedBy=timers.target
EOF
```

La configuración significa:

```text
Al iniciar el sistema
       ↓
esperar 1 minuto
       ↓
ejecutar YARA
       ↓
cada 1 minuto
       ↓
volver a ejecutar YARA
```

---

# :white_check_mark: 31. Validar los archivos `systemd`

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/cybersoc-yara.service \
  /etc/systemd/system/cybersoc-yara.timer
```

Si no aparecen errores:

```bash
sudo systemctl daemon-reload
```

Activar el timer:

```bash
sudo systemctl enable --now cybersoc-yara.timer
```

---

# :alarm_clock: 32. Comprobar el timer

```bash
systemctl list-timers --all cybersoc-yara.timer
```

Debemos encontrar:

```text
cybersoc-yara.timer
```

activo y con su próxima ejecución programada.

---

# :test_tube: 34. Prueba automática

```bash
FINAL_FILE="/opt/cybersoc-hunting/evidence/purplewolf_final-$(date -u +%Y%m%dT%H%M%S)-$.dat"

sudo tee "$FINAL_FILE" >/dev/null <<'EOF'
CYBERSOC-LAB
Campaign: PurpleWolf
C2: 172.30.0.20
User-Agent: PurpleWolf-C2
Final-Test: TRUE
EOF
```

Mostrar la ruta:

```bash
printf 'Copia esta ruta en VM 1: %s\n' "$FINAL_FILE"
```

Comprobar el timer:

```bash
systemctl list-timers --all cybersoc-yara.timer
```

### No ejecutar el script manualmente

Esperar aproximadamente un minuto.

---

# :scroll: 35. Revisar la ejecución automática

En VM2:

```bash
sudo journalctl \
  -u cybersoc-yara.service \
  -n 15 \
  --no-pager
```

Consultar el log:

```bash
sudo jq -c \
  --arg file "$FINAL_FILE" \
  'select(.file==$file)' \
  /var/log/cybersoc-yara.log |
  tail -1
```

Calcular el hash:

```bash
sha256sum "$FINAL_FILE"
```

Resultado esperado:

* el servicio finaliza sin errores;
* aparece un JSON correspondiente al archivo nuevo;
* el SHA-256 coincide;
* el timer continúa activo.

El servicio `oneshot` puede quedar como `inactive (dead)` después de ejecutarse correctamente. **Eso no significa que el timer esté detenido.**

---

# :mag: 36. Comprobar la alerta automática en Wazuh

En VM1:

```bash
cd /opt/wazuh-docker/single-node
```

Buscar la alerta correspondiente al archivo:

```bash
sudo docker compose exec -T wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg file "$FINAL_FILE" 'select(
    .agent.name=="cyberrange-suricata" and
    .data.file==$file and
    .rule.id=="100600" and
    .rule.level==12
  )' |
  tail -1 |
  jq .
```

El resultado esperado es:

```text
rule.id    = 100600
rule.level = 12
```

y:

```text
data.file = <ruta del archivo>
```

---

# :bar_chart: 37. Verificar en Wazuh Dashboard

Acceder:

```text
https://192.168.56.10
```

Navegar hasta:

```text
Threat intelligence
        ↓
Threat Hunting
```

Aplicar:

```text
agent.name:"cyberrange-suricata" AND rule.id:100600
```

Para una ejecución manual podemos utilizar:

```text
data.run_id
```

como filtro.

Para la prueba automática:

```text
data.file
```

con la ruta completa de `FINAL_FILE`.



---

# :brain: 38. Relación con Threat Intelligence

En Threat Intelligence:

```text
IP observada
     ↓
CDB
     ↓
IOC Match
     ↓
Rule 100500
```

En YARA:

```text
Archivo observado
     ↓
YARA
     ↓
Pattern Match
     ↓
Rule 100600
```

Son dos fuentes de evidencia diferentes:

```text
              Evidencias
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Red / IP              Host / Archivo
        ↓                   ↓
 Threat Intel             YARA
        ↓                   ↓
  Rule 100500            Rule 100600
```