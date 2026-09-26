# :mag: 05 - YARA

## :dart: Objetivo

En esta etapa incorporamos **YARA** al laboratorio para realizar una búsqueda basada en patrones sobre archivos del host.

La hipótesis de la práctica es:

> El host que generó la alerta de Threat Intelligence puede contener archivos con indicadores asociados al artefacto simulado **PurpleWolf**.

Para comprobarlo vamos a:

1. Instalar YARA.
2. Crear el directorio donde almacenaremos las reglas.
3. Crear una regla YARA.
4. Crear archivos de prueba.
5. Realizar un control negativo.
6. Detectar el artefacto simulado.
7. Crear un script que ejecute las búsquedas.
8. Registrar los resultados como eventos JSON.
9. Crear una regla de Wazuh para detectar los resultados de YARA.
10. Validar la regla con `wazuh-logtest`.
11. Ejecutar una búsqueda real.
12. Automatizar la ejecución mediante un `systemd timer`.

> :warning: Los archivos utilizados en esta práctica son **artefactos de texto inofensivos** creados exclusivamente para el laboratorio. No estamos utilizando malware real.

El README original plantea esta etapa como parte de la investigación posterior a Threat Intelligence.

---

# :building_construction: 1. Arquitectura de YARA

En esta etapa, el flujo es diferente al de Suricata.

Suricata analiza **tráfico de red**:

```text
Attacker
   ↓
DVWA
   ↓
Suricata
   ↓
eve.json
   ↓
Wazuh Agent
```

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

YARA permite definir una regla que describe determinados patrones que queremos encontrar dentro de archivos.

Nuestra regla buscará cuatro cadenas:

```text
PurpleWolf
172.30.0.20
CYBERSOC-LAB
PurpleWolf-C2
```

Pero la regla **no exige las cuatro**.

La condición será:

```text
3 of them
```

Por lo tanto:

```text
4 coincidencias → MATCH
3 coincidencias → MATCH
2 coincidencias → NO MATCH
1 coincidencia  → NO MATCH
0 coincidencias → NO MATCH
```

El README utiliza precisamente esta condición para el artefacto simulado.

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

Y:

```bash
command -v yara
```

El segundo comando permite comprobar dónde está instalado el ejecutable.

---

# :file_folder: 4. Crear los directorios de trabajo

Crear el directorio donde almacenaremos las reglas:

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

No son los patrones que YARA busca.

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

Por ejemplo:

```text
PurpleWolf
purplewolf
PURPLEWOLF
```

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

contiene tres patrones.

Por lo tanto:

```text
MATCH
```

El cuarto patrón no es obligatorio.

---

# :white_check_mark: 8. Validar la sintaxis de la regla

Antes de analizar archivos, comprobamos que la regla sea válida:

```bash
sudo yara \
  /var/ossec/etc/yara/rules/cybersoc_purplewolf.yar \
  /dev/null
```

Resultado esperado:

```text
sin errores
```

En este punto todavía **no estamos buscando un archivo real**.

Estamos comprobando que YARA pueda cargar correctamente la regla.

---

# :test_tube: 9. Crear un archivo normal

Primero hacemos un **control negativo**.

Crear:

```bash
echo "Archivo normal del laboratorio CyberSOC" |
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
```

Esto significa que:

```text
normal.txt
   ↓
YARA
   ↓
No encuentra 3 patrones
   ↓
NO MATCH
```

Este control es importante porque demuestra que la regla no está generando coincidencias sobre cualquier archivo.

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

El archivo contiene los cuatro patrones:

```text
CYBERSOC-LAB
PurpleWolf
172.30.0.20
PurpleWolf-C2
```

Por lo tanto:

```text
4 patrones encontrados
        ↓
3 requeridos
        ↓
MATCH
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

YARA necesita producir una salida que posteriormente pueda ser recolectada por Wazuh.

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

Antes de modificar la configuración, crear una copia de seguridad:

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

Estamos diciéndole al Wazuh Agent:

> El archivo `/var/log/cybersoc-yara.log` contiene eventos JSON que deben ser recolectados.

En esta etapa **no configuramos FIM**.

La configuración de:

```xml
<syscheck>
```

y la monitorización de:

```text
/opt/cybersoc-hunting/evidence
```

corresponderán a `06-fim.md`.

Esto respeta la separación que hicimos para nuestra documentación, aunque el README original combina ambas configuraciones en un mismo paso.

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

El README utiliza un script para recorrer los archivos de evidencia, ejecutar YARA y transformar los resultados en eventos JSON.

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

> **Nota sobre el comando:** se conserva la sintaxis del README de referencia para mantener la implementación alineada con la fuente.

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

Si no devuelve ninguna salida, la sintaxis Bash es válida.

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

y ejecutará YARA contra cada a
