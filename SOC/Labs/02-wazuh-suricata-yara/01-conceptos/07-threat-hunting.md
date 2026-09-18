# :mag: Threat Hunting

**Threat Hunting** es la búsqueda de evidencias que permiten determinar qué está ocurriendo dentro de un sistema, más allá de una alerta individual.

Hacemos Threat Hunting sobre **VM2 — CyberRange**, utilizando las evidencias generadas por **Threat Intelligence, FIM y YARA**.

---

## :dart: ¿Qué buscamos en nuestro laboratorio?

hipotesis:

> El host que generó una alerta de Threat Intelligence podría contener archivos relacionados con los indicadores PurpleWolf.

Por eso, después de obtener una alerta TI, investigamos el host buscando evidencias adicionales.

---

## :mag_right: ¿Qué aporta cada mecanismo?

Cada componente busca algo diferente:

```text
Threat Intelligence
        ↓
¿La actividad involucra un IOC conocido?

FIM
        ↓
¿Hubo cambios en los archivos?

YARA
        ↓
¿Los archivos contienen patrones definidos?
```

En nuestro laboratorio:

* **Threat Intelligence:** aporta contexto sobre la IP observada.
* **FIM:** detecta cambios en `/opt/cybersoc-hunting/evidence`.
* **YARA:** busca los patrones definidos en la regla `CYBERSOC_PurpleWolf_Artifact`.

---

## :link: ¿Cómo se relacionan?

No forman una cadena donde uno necesariamente active al siguiente.

```text
                    VM2
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
      Suricata      FIM        YARA
          │          │          │
          ▼          │          ▼
    evento de red    │     YARA Match
          │          │          │
          ▼          ▼          │
      Threat Intelligence       │
          │                     │
          └──────────┬──────────┘
                     ▼
                  Wazuh
                     │
                     ▼
               Investigación
```

**FIM registra cambios. YARA se ejecuta de forma independiente mediante un timer de `systemd`. FIM no dispara YARA.**

---

## :file_folder: Evidencia 

Los archivos utilizados para el hunting se encuentran en:

```text
/opt/cybersoc-hunting/evidence
```

El laboratorio utiliza archivos de prueba inofensivos que contienen indicadores simulados de **PurpleWolf**.

La regla YARA busca, entre otros patrones:

```text
PurpleWolf
172.30.0.20
CYBERSOC-LAB
PurpleWolf-C2
```

y requiere que coincidan **3 de ellos**.

---

## :shield: Resultado esperado

El objetivo no es simplemente obtener una alerta, sino **correlacionar diferentes evidencias**:

```text
IP observada
    ↓
Threat Intelligence
    ↓
Alerta priorizada
    ↓
Investigación del host
    ├── FIM → cambios en archivos
    └── YARA → patrones encontrados
```

Esto permite pasar de:

```text
"Existe una alerta"
```

a:

```text
"Tenemos diferentes evidencias que debemos analizar
para determinar qué ocurrió."
```

Una coincidencia de IOC o YARA **no demuestra por sí sola que exista un compromiso**; debe analizarse junto con el resto de la evidencia.

---

## :brain: Lo esencial

Recordar:

* Threat Hunting = **búsqueda e investigación activa de evidencias**.
* Se realiza principalmente sobre **VM2**.
* **Threat Intelligence** aporta contexto sobre IOCs.
* **FIM** detecta cambios en archivos.
* **YARA** busca patrones dentro de archivos.
* FIM y YARA funcionan **de forma independiente**.
* El objetivo es **correlacionar evidencias**, no asumir automáticamente que una coincidencia equivale a un incidente.

El concepto central es:

```text
Detección
   ↓
Contexto
   ↓
Búsqueda de evidencias
   ↓
Correlación
   ↓
Investigación
```
