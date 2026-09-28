# :mag: 06 - File Integrity Monitoring (FIM)

## :dart: Objetivo

Configuramos **File Integrity Monitoring (FIM)** mediante el módulo `syscheck` de Wazuh.

El objetivo es detectar cambios en los archivos ubicados dentro de:

```text
/opt/cybersoc-hunting/evidence
```

Este directorio contiene los artefactos utilizados durante la investigación con YARA.

FIM permitirá detectar eventos como:

* creación de archivos;
* modificaciones;
* cambios en atributos o contenido;
* eliminación de archivos.

La finalidad es generar **telemetría sobre cambios en el host** que posteriormente pueda ser utilizada durante Threat Hunting.

---

# :brain: 1. FIM

En nuestro laboratorio, Wazuh Agent utiliza `syscheck` para monitorizar:

```text
/opt/cybersoc-hunting/evidence
```

Cuando aparece un archivo nuevo, Wazuh puede generar un evento indicando que detectó un cambio.

El flujo es:

```text
Archivo nuevo
     ↓
Wazuh Syscheck
     ↓
Evento FIM
     ↓
Wazuh Manager
     ↓
Alerta
```
---

# :wrench: 3. Configurar FIM en Wazuh Agent

Esta configuración se realiza en:

```text
VM2 — CyberRange
```

---

# :gear: 4. Configurar `syscheck`

Abrir:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Buscar el bloque:

```xml
<syscheck>
```

Dentro de este bloque debemos asegurarnos de que:

```xml
<disabled>no</disabled>
```

y:

```xml
<scan_on_start>yes</scan_on_start>
```

estén configurados.

La configuración indica que:

* `disabled=no` → FIM está habilitado.
* `scan_on_start=yes` → Wazuh realiza un escaneo inicial cuando inicia el agente.

---

# :file_folder: 5. Agregar el directorio monitorizado

Dentro del bloque `<syscheck>` agregar:

```xml
<directories realtime="yes" check_all="yes">/opt/cybersoc-hunting/evidence</directories>
```

La configuración queda conceptualmente:

```xml
<syscheck>
    ...
    <disabled>no</disabled>
    <scan_on_start>yes</scan_on_start>
    ...
    <directories realtime="yes" check_all="yes">/opt/cybersoc-hunting/evidence</directories>
    ...
</syscheck>
```

### ¿Qué significa?

```text
realtime="yes"
```

Solicita monitorización en tiempo real del directorio.

```text
check_all="yes"
```

Hace que Wazuh realice las comprobaciones de integridad configuradas para los archivos monitorizados.

La ruta:

```text
/opt/cybersoc-hunting/evidence
```

es el directorio que queremos vigilar.

---


# :white_check_mark: 6. Validar la configuración

Antes de reiniciar el agente, validar los tres componentes:

```bash
sudo /var/ossec/bin/wazuh-agentd -t &&
sudo /var/ossec/bin/wazuh-syscheckd -t &&
sudo /var/ossec/bin/wazuh-logcollector -t
```

El objetivo es que las tres validaciones finalicen correctamente.

Estamos comprobando:

```text
wazuh-agentd
     ↓
Configuración general

wazuh-syscheckd
     ↓
Configuración FIM

wazuh-logcollector
     ↓
Recolección de logs
```

> No conviene reiniciar el agente si alguna de estas validaciones devuelve errores.

---

# :arrows_counterclockwise: 8. Reiniciar Wazuh Agent

Si las validaciones fueron correctas:

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent --no-pager
# active (running)
```

---

# :satellite: 9. Comprobar la conexión con Wazuh Manager

Revisar el log del agente:

```bash
sudo grep -Ei \
  'connected|syscheck|cybersoc-yara|error' \
  /var/ossec/logs/ossec.log | tail -30
```

Después del reinicio debemos esperar a que finalice el **escaneo FIM inicial** antes de realizar la prueba con un archivo nuevo.

---

# :hourglass_flowing_sand: 10. Esperar el escaneo inicial

El primer arranque de FIM puede generar actividad mientras Wazuh establece el estado inicial de los archivos.

No debemos utilizar el escaneo inicial como nuestra evidencia principal.

La prueba que nos interesa es crear **un archivo nuevo después de que el escaneo inicial haya terminado**.

---

# :test_tube: 11. Crear un archivo de prueba FIM

Una vez finalizado el escaneo inicial, crear un archivo nuevo en VM2:

```bash
FIM_FILE="/opt/cybersoc-hunting/evidence/fim-test-$(date -u +%Y%m%dT%H%M%S)-$.txt"

printf 'FIM TEST\n' | sudo tee "$FIM_FILE"
```

Mostrar la ruta:

```bash
printf 'Copia esta ruta en VM 1: %s\n' "$FIM_FILE"
```

El archivo tendrá una ruta similar a:

```text
/opt/cybersoc-hunting/evidence/fim-test-20260926T010000-12345.txt
```

---

# :mag: 12. ¿Qué debería detectar FIM?

El archivo acaba de aparecer dentro de:

```text
/opt/cybersoc-hunting/evidence
```

La evidencia que buscamos es la ruta del archivo:

```text
syscheck.path
```

---

# :desktop_computer: 13. Verificar el agente desde VM1

Primero comprobar que el agente esté conectado:

```bash
cd /opt/wazuh-docker/single-node

sudo docker compose exec -T \
  wazuh.manager \
  /var/ossec/bin/agent_control -lc
```

Debe aparecer:

```text
cyberrange-suricata    Active
```

---

# :mag: 15. Buscar el evento FIM

En VM1, conservar la ruta obtenida anteriormente:

```bash
read -r -p \
  "Pega la ruta FIM_FILE de VM 2: " \
  FIM_FILE
```

Consultar `alerts.json`:

```bash
sudo docker compose exec -T \
  wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg file "$FIM_FILE" \
  'select(
    .agent.name=="cyberrange-suricata"
    and .syscheck.path==$file
  )' |
  tail -3
```

La condición principal de búsqueda es:

```text
agent.name
      =
cyberrange-suricata
```

y:

```text
syscheck.path
      =
ruta del archivo creado
```

---

# :white_check_mark: 16. Resultado esperado

Debemos encontrar una alerta FIM asociada con:

```text
cyberrange-suricata
```

y:

```text
/opt/cybersoc-hunting/evidence/fim-test-...
```
---

# :arrows_counterclockwise: 17. Prueba de modificación

La primera prueba demuestra:

```text
archivo nuevo → FIM
```

Modificar el archivo creado:

```bash
printf 'FIM MODIFICATION TEST\n' |
  sudo tee -a "$FIM_FILE"
```

Esto cambia el contenido del archivo.

Consultar nuevamente en VM1:

```bash
sudo docker compose exec -T \
  wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg file "$FIM_FILE" \
  'select(
    .agent.name=="cyberrange-suricata"
    and .syscheck.path==$file
  )' |
  tail -5
```

---


# :test_tube: 20. Comprobar FIM junto con YARA

En VM2, el archivo utilizado para la prueba automática de YARA será:

```text
/opt/cybersoc-hunting/evidence/purplewolf_final-...
```

El timer de YARA lo analiza automáticamente.

Al mismo tiempo, la creación del archivo debe generar un evento FIM porque el directorio está monitorizado.

En VM1 podemos consultar FIM mediante:

```bash
sudo docker compose exec -T \
  wazuh.manager \
  cat /var/ossec/logs/alerts/alerts.json |
  jq -c --arg file "$FINAL_FILE" \
  'select(
    .agent.name=="cyberrange-suricata"
    and .syscheck.path==$file
  )' |
  tail -3
```

---

# :bar_chart: 21. Comparar FIM y YARA sobre el mismo archivo

Para el mismo:

```text
FINAL_FILE
```

podemos tener:

### Evidencia FIM

```text
syscheck.path
      ↓
/opt/cybersoc-hunting/evidence/purplewolf_final-...
```

### Evidencia YARA

```text
data.file
      ↓
/opt/cybersoc-hunting/evidence/purplewolf_final-...
```

Esto permite observar que ambas están analizando el mismo objeto desde perspectivas diferentes.


---

# :bar_chart: 22. Verificación en Wazuh Dashboard

Acceder a:

```text
https://192.168.56.10
```

Navegar hasta:

```text
Threat intelligence
        ↓
Threat Hunting
```

Para consultar FIM:

```text
agent.name:"cyberrange-suricata" AND rule.groups:syscheck
```

El objetivo es encontrar eventos correspondientes al agente:

```text
cyberrange-suricata
```

y al archivo que acabamos de crear.

---

# :warning: 26. Qué demuestra un evento FIM

Si observamos:

```text
syscheck.path =
/opt/cybersoc-hunting/evidence/purplewolf_final-...
```

podemos afirmar:

> Wazuh detectó un cambio relacionado con ese archivo monitorizado.

Pero no podemos concluir solamente a partir de FIM:

```text
FIM → Malware
```
---

# :mag: 27. Relación con las etapas anteriores

Hasta ahora tenemos diferentes fuentes de evidencia:

```text
Threat Intelligence
        │
        ↓
    IP / IOC
        │
        ↓
   Rule 100500


YARA
        │
        ↓
   Contenido
        │
        ↓
   Rule 100600


FIM
        │
        ↓
  Cambio de archivo
        │
        ↓
   Syscheck
```
---

> **Idea clave:** FIM nos dice **que un archivo cambió**;