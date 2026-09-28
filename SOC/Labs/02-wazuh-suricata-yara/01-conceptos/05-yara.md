# :mag: YARA

**YARA** es una herramienta utilizada para identificar archivos mediante **patrones definidos en reglas**.

Lo usamos como parte de **Threat Hunting** para buscar artefactos relacionados con la actividad simulada de PurpleWolf.

---

## :dart: ¿Qué hace YARA?

YARA analiza el contenido de archivos y comprueba si coincide con las condiciones definidas en una regla.

como funciona?

```text 
Archivo
   │
   ▼
 YARA
   │
   ▼
Regla YARA
   │
   ├── Match
   │
   └── Sin match
```

Un **match** nos dice que el archivo cumple con alguna condicion de las regla definidas

Esto no significa automáticamente que el archivo sea malware.

---

## :file_folder: YARA en nuestro lab

Los archivos utilizados para el hunting se encuentran en:

```text 
/opt/cybersoc-hunting/evidence
```

La regla utilizada en el laboratorio busca diferentes patrones relacionados con PurpleWolf.

Entre ellos:

```text 
$campaign
$c2
$marker
$agent
```

La condición de la regla requiere que coincidan:

```text 
3 de los 4 patrones
Para que exista un match
```

---

## :arrows_counterclockwise: ¿Cómo se ejecuta?

**YARA no se ejecuta cuando FIM detecta un cambio**.

La ejecución se realiza periódicamente mediante un **timer de systemd**.

El resultado se registra en:

```text
/var/log/cybersoc-yara.log
```

Posteriormente, Wazuh puede monitorizar ese log.

---

## :link: YARA + Wazuh

El flujo dentro del laboratorio es:

```text 
Archivos
   │
   ▼
YARA
   │
   ▼
/var/log/cybersoc-yara.log
   │
   ▼
Wazuh Agent
   │
   ▼
Wazuh Manager
   │
   ▼
Regla Wazuh 100600
   │
   ▼
Alerta
```

La regla **`100600`** es una regla personalizada de Wazuh utilizada para procesar los eventos relacionados con YARA.

Es diferente de la regla YARA que realiza la búsqueda de patrones.

---

## :brain: Lo esencial

Para este laboratorio debemos recordar:

* YARA busca **patrones dentro de archivos**.
* La búsqueda se realiza sobre `/opt/cybersoc-hunting/evidence`.
* La regla utiliza cuatro patrones.
* Se requiere coincidencia de **3 de 4**.
* YARA se ejecuta mediante un **timer de systemd**.
* El resultado se registra en `/var/log/cybersoc-yara.log`.
* Wazuh procesa posteriormente ese log.
* `100600` es la **regla personalizada de Wazuh**, no el identificador de la regla YARA.
* YARA y FIM son mecanismos **independientes**.

El flujo que debemos recordar es:

```text
Archivos
   ↓
YARA
   ↓
YARA Match
   ↓
cybersoc-yara.log
   ↓
Wazuh
   ↓
Regla 100600
   ↓
Alerta
```