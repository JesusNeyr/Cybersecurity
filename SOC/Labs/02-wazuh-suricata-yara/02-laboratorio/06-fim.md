# :mag: 06 - File Integrity Monitoring (FIM)

## :dart: Objetivo

En esta etapa configuramos **File Integrity Monitoring (FIM)** mediante el módulo `syscheck` de Wazuh.

El objetivo es detectar cambios en los archivos ubicados dentro de:

```text
/opt/cybersoc-hunting/evidence
```

En el laboratorio, este directorio contiene los artefactos utilizados durante la investigación con YARA.

FIM permitirá detectar eventos como:

* creación de archivos;
* modificaciones;
* cambios en atributos o contenido;
* eliminación de archivos.

La finalidad es generar **telemetría sobre cambios en el host** que posteriormente pueda ser utilizada durante Threat Hunting.

> :warning: FIM no ejecuta YARA. Son mecanismos independientes. El `systemd timer` ejecuta YARA periódicamente, mientras que FIM monitoriza los cambios en el directorio configurado.

---

# :building_construction: 1. Flujo de FIM

El funcionamiento dentro de nuestro laboratorio es:

```text
                 VM2 — CyberRange
                        │
                        ↓
        /opt/cybersoc-hunting/evidence
                        │
                        ↓
                  Wazuh Agent
                        │
                 ┌──────┴──────┐
                 │             │
                 ↓             ↓
                FIM           YARA
              syscheck      systemd timer
                 │             │
                 ↓             ↓
             Evento FIM    Resultado YARA
                 │             │
                 ↓             ↓
              Wazuh Manager
                 │
                 ↓
           Wazuh Dashboard
```

La diferencia fundamental es:

```text
FIM
 │
 └── ¿Cambió este archivo?

YARA
 │
 └── ¿Este archivo contiene los patrones que estoy buscando?
```

Por lo tanto:

```text
Cambio de archivo ≠ ejecución de YARA
```

---

# :brain: 2. ¿Qué es FIM?

**File Integrity Monitoring** es un mecanismo utilizado para supervisar la integridad de archivos y detectar modificaciones.

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

Esto proporciona una evidencia diferente de la que obtenemos con YARA.

---

# :wrench: 3. Configurar FIM en Wazuh Agent

Esta configuración se realiza en:

```text
VM2 — CyberRange
```

El archivo de configuración del agente es:

```text
/var/ossec/etc/ossec.conf
```

Antes de modificarlo, crear una copia de seguridad:

```bash
sudo cp -p /var/ossec/etc/ossec.conf \
  "/var/ossec/etc/ossec.conf.bak-fim-$(date +%Y%m%dT%H%M%S)"
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

El README indica que esta ruta debe añadirse **una sola vez**, conservando las demás opciones existentes del bloque `syscheck`.

---

# :page_facing_up: 6. Mantener la configuración del log YARA

El mismo `ossec.conf` también necesita conservar la recolección del log generado por YARA:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/cybersoc-yara.log</location>
</localfile>
```

Esto **no es FIM**.

Son dos configuraciones diferentes dentro del mismo archivo:

```text
ossec.conf
    │
    ├── <syscheck>
    │       ↓
    │      FIM
    │
    └── <localfile>
            ↓
       Log de YARA
```

Esto es importante porque ambas fuentes de telemetría posteriormente llegarán a Wazuh, pero representan evidencias diferentes.

---

# :white_check_mark: 7. Validar la configuración

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
```

Comprobar el servicio:

```bash
sudo systemctl status wazuh-agent --no-pager
```

Debe aparecer:

```text
active (running)
```

---

# :satellite: 9. Comprobar la conexión con Wazuh Manager

Revisar el log del agente:

```bash
sudo grep -Ei \
  'connected|syscheck|cybersoc-yara|error' \
  /var/ossec/logs/ossec.log | tail -30
```

Debemos poder confirmar:

```text
Connected to the server
```

y observar actividad relacionada con:

```text
syscheck
```

El README indica que después del reinicio debemos esperar a que finalice el **escaneo FIM inicial** antes de realizar la prueba con un archivo nuevo.

---

# :hourglass_flowing_sand: 10. Esperar el escaneo inicial

El primer arranque de FIM puede generar actividad mientras Wazuh establece el estado inicial de los archivos.

Por eso:

```text
Inicio del agente
       ↓
Escaneo inicial FIM
       ↓
Establecimiento del estado inicial
       ↓
FIM listo para detectar cambios posteriores
```

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

Por lo tanto:

```text
Archivo nuevo
      ↓
Directorio monitorizado
      ↓
Wazuh Syscheck
      ↓
Evento FIM
```

La evidencia que buscamos es la ruta del archivo:

```text
syscheck.path
```

---

# :desktop_computer: 13. Verificar el agente desde VM1

Ahora pasamos a:

```text
VM1 — CyberSOC
```

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

# :package: 14. Instalar `jq` en VM1

El README utiliza `jq` para consultar las alertas JSON del Manager.

Si todavía no está instalado:

```bash
sudo apt install -y jq
```

Verificar:

```bash
jq --version
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

Conceptualmente:

```text
VM2
 │
 │ crea archivo
 ↓
/opt/cybersoc-hunting/evidence/fim-test-...
 │
 ↓
Wazuh Syscheck
 │
 ↓
Wazuh Agent
 │
 ↓
Wazuh Manager
 │
 ↓
alerts.json
 │
 ↓
syscheck.path
```

El README especifica que, si la alerta todavía no llegó, debemos esperar unos segundos y repetir la consulta.

---

# :arrows_counterclockwise: 17. Prueba de modificación

La primera prueba demuestra:

```text
archivo nuevo → FIM
```

También podemos demostrar que FIM registra una modificación.

Modificar el archivo creado:

```bash
printf 'FIM MODIFICATION TEST\n' |
  sudo tee -a "$FIM_FILE"
```

Esto cambia el contenido del archivo.

El mecanismo esperado es:

```text
Archivo existente
      ↓
Contenido modificado
      ↓
Wazuh Syscheck
      ↓
Nuevo evento FIM
```

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

# :mag: 18. FIM no ejecuta YARA

Este punto debe quedar completamente claro.

Si ejecutamos:

```bash
printf 'FIM MODIFICATION TEST\n' |
  sudo tee -a "$FIM_FILE"
```

FIM detecta:

```text
CAMBIO DE ARCHIVO
```

Pero FIM **no hace automáticamente**:

```text
Cambio
  ↓
YARA
```

La ejecución de YARA corresponde al mecanismo que documentamos en `05-yara.md`:

```text
systemd timer
      ↓
cybersoc-yara.service
      ↓
cybersoc-yara-scan.sh
      ↓
YARA
```

Por lo tanto tenemos dos mecanismos paralelos:

```text
                    Archivo
                       │
              ┌────────┴────────┐
              │                 │
              ↓                 ↓
             FIM               YARA
              │                 │
      detecta cambios      busca patrones
              │                 │
              ↓                 ↓
         evento FIM        yara_match
```

Esta separación está indicada explícitamente en el README original.

---

# :link: 19. Relación entre FIM y YARA

Aunque funcionan de forma independiente, ambas fuentes pueden referirse al **mismo archivo**.

Por ejemplo:

```text
/opt/cybersoc-hunting/evidence/purplewolf_update.dat
```

YARA puede determinar:

```text
CYBERSOC_PurpleWolf_Artifact
        ↓
MATCH
```

Mientras FIM puede determinar:

```text
purplewolf_update.dat
        ↓
fue creado/modificado
```

Por lo tanto:

```text
                    MISMO ARCHIVO
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
             FIM                   YARA
              │                     │
       ¿Cambió el archivo?   ¿Tiene patrones?
              │                     │
              ↓                     ↓
          Evidencia 1           Evidencia 2
```

Esta relación será especialmente útil durante `07-threat-hunting.md`.

---

# :test_tube: 20. Comprobar FIM junto con YARA

El README realiza una prueba final donde un mismo archivo produce evidencia de ambos mecanismos.

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

Esto permite observar que ambas tecnologías están analizando el mismo objeto desde perspectivas diferentes.

```text
                 Archivo
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
         FIM                 YARA
          │                   │
       Cambio              Contenido
          │                   │
          ↓                   ↓
   syscheck.path          data.file
```

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

El README utiliza precisamente `rule.groups:syscheck` para consultar los eventos FIM.

---

# :mag: 23. Filtrar por archivo

También podemos utilizar la ruta completa del archivo como referencia.

Por ejemplo:

```text
agent.name:"cyberrange-suricata"
```

combinado con la búsqueda de:

```text
syscheck.path
```

y la ruta:

```text
/opt/cybersoc-hunting/evidence/purplewolf_final-...
```

La ventaja de utilizar un nombre de archivo concreto es que evita mezclar eventos de diferentes pruebas.

---

# :arrows_counterclockwise: 24. Flujo completo de FIM

La etapa queda resumida de esta manera:

```text
                  VM2
                   │
                   ↓
       /opt/cybersoc-hunting/evidence
                   │
                   ↓
            Wazuh Syscheck
                   │
            ¿Cambio detectado?
                   │
                  SÍ
                   │
                   ↓
             Wazuh Agent
                   │
                   ↓
            Wazuh Manager
                   │
                   ↓
             alerts.json
                   │
                   ↓
            Wazuh Dashboard
```

Mientras YARA sigue su propio flujo:

```text
                  VM2
                   │
                   ↓
       /opt/cybersoc-hunting/evidence
                   │
                   ↓
          systemd timer
                   │
                   ↓
             YARA scan
                   │
              ¿Match?
                   │
                  SÍ
                   │
                   ↓
        cybersoc-yara.log
                   │
                   ↓
             Wazuh Agent
                   │
                   ↓
            Wazuh Manager
```

---

# :brain: 25. FIM vs. YARA

| Característica | FIM                    | YARA                      |
| -------------- | ---------------------- | ------------------------- |
| Objetivo       | Detectar cambios       | Buscar patrones           |
| Tecnología     | Wazuh Syscheck         | YARA                      |
| Entrada        | Archivos monitorizados | Archivos analizados       |
| Disparador     | Cambio detectado       | Ejecución de YARA         |
| Automatización | `syscheck`             | `systemd timer`           |
| Resultado      | Evento FIM             | `yara_match`              |
| Pregunta       | ¿Cambió el archivo?    | ¿Contiene estos patrones? |
| Regla Wazuh    | Reglas `syscheck`      | Regla `100600`            |

La diferencia fundamental es:

```text
FIM → INTEGRIDAD

YARA → CONTENIDO
```

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

FIM nos indica que hubo un cambio.

Para saber si el contenido presenta determinados indicadores utilizamos YARA.

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

No existe una regla en este laboratorio que automáticamente diga:

```text
TI + FIM + YARA
       ↓
   Incidente
```

La correlación entre esas evidencias será parte del análisis de:

```text
07-threat-hunting.md
```

El README también aclara que TI y YARA son alertas independientes y que el laboratorio no crea una regla que las correlacione automáticamente.

---

# :white_check_mark: 28. Checklist

```text
[ ] El bloque <syscheck> está habilitado
[ ] <disabled> está configurado en no
[ ] <scan_on_start> está configurado en yes
[ ] /opt/cybersoc-hunting/evidence está monitorizado
[ ] realtime="yes" está configurado
[ ] check_all="yes" está configurado
[ ] La ruta aparece una sola vez en <syscheck>
[ ] wazuh-agentd -t no presenta errores
[ ] wazuh-syscheckd -t no presenta errores
[ ] wazuh-logcollector -t no presenta errores
[ ] Wazuh Agent está active (running)
[ ] El agente aparece Active en el Manager
[ ] Finalizó el escaneo FIM inicial
[ ] Se creó un archivo nuevo en evidence
[ ] Wazuh detectó el archivo nuevo
[ ] El evento contiene syscheck.path
[ ] Se realizó una prueba de modificación
[ ] Wazuh registró el cambio del archivo
[ ] Se verificó FIM en alerts.json
[ ] Se verificó FIM en Wazuh Dashboard
[ ] Se comprende que FIM no ejecuta YARA
[ ] Se comprende que YARA y FIM generan evidencias independientes
```

---

# :mortar_board: 29. Resultado de aprendizaje

Al finalizar esta etapa debemos poder explicar:

```text
¿Qué hace FIM?
        ↓
Monitoriza archivos y detecta cambios.

¿Qué hace YARA?
        ↓
Busca patrones dentro de archivos.

¿FIM ejecuta YARA?
        ↓
NO.

¿Cómo se ejecuta YARA?
        ↓
Mediante el mecanismo automatizado del laboratorio:
systemd timer → script → YARA.

¿Pueden analizar el mismo archivo?
        ↓
SÍ.

¿Generan la misma evidencia?
        ↓
NO.
```

El modelo mental final es:

```text
                  HOST
                   │
                   ↓
               ARCHIVO
                   │
          ┌────────┴────────┐
          │                 │
          ↓                 ↓
         FIM               YARA
          │                 │
       CAMBIO            PATRONES
          │                 │
          ↓                 ↓
     SYS_CHECK          YARA MATCH
          │                 │
          └────────┬────────┘
                   ↓
             WAZUH MANAGER
                   │
                   ↓
              INVESTIGACIÓN
                   │
                   ↓
          07-threat-hunting
```

> **Idea clave:** FIM nos dice **que un archivo cambió**; YARA nos ayuda a determinar **si el contenido coincide con determinados patrones**. Ambos generan evidencia independiente que posteriormente puede ser analizada durante Threat Hunting.
