# :warning: Threat Intelligence

**Threat Intelligence (TI)** es información que aporta contexto sobre indicadores relacionados con posibles amenazas.

En nuestro lab utilizamos una **CDB List de Wazuh** para clasificar una IP observada en un evento de Suricata.

---

## :dart: ¿Qué hacemos en nuestro lab?

En la primera etapa, Suricata detecta tráfico proveniente de:

```text 
172.30.0.20
```

Wazuh recibe ese evento, pero inicialmente solamente conoce datos como:

```text 
src_ip  = 172.30.0.20
dest_ip = 172.30.0.10
```

La Threat Intelligence agrega contexto sobre esa IP.

```text 
Evento Suricata
      │
      ▼
src_ip = 172.30.0.20
      │
      ▼
Threat Intelligence
      │
      ▼
CDB List
      │
      ▼
IOC Match
```

Objetivo transformar un evento de red genérico en una alerta que tenga **contexto adicional para su investigación**.

---

## :file_folder: CDB List

Wazuh utiliza una **CDB List** para almacenar los indicadores que queremos consultar.

En nuestro laboratorio la lista es:

```text 
/var/ossec/etc/lists/threat-intel-ip
```

Contiene pares:

```text 
IP:etiqueta
```

Por ejemplo:

```text 
172.30.0.20:PurpleWolf-C2-high
198.51.100.25:DemoC2-high
203.0.113.44:DemoPhishing-high
```

La IP `172.30.0.20` corresponde al **Attacker de nuestro laboratorio**. Las demás son indicadores de ejemplo y no se utilizan para generar tráfico.

---

## :mag: ¿Cómo se realiza la coincidencia?

Wazuh toma la IP de origen del evento:

```text 
src_ip = 172.30.0.20
```

y la consulta contra la CDB:

```text 
172.30.0.20
       │
       ▼
threat-intel-ip
       │
       ▼
PurpleWolf-C2-high
       │
       ▼
IOC Match
```

La CDB proporciona el **indicador y su etiqueta**.

No determina por sí misma el nivel de la alerta.

---

## :shield: Regla personalizada `100500`

Cuando existe una coincidencia, Wazuh utiliza nuestra regla personalizada:

```text 
100500
```

Esta regla exige dos condiciones:

```text 
Evento Suricata
      │
      ▼
Regla Wazuh 86601
      │
      ▼
¿src_ip está en la CDB?
      │
     SÍ
      │
      ▼
Regla 100500
      │
      ▼
Level 12
```

La regla `100500` **no pertenece a Suricata**.

Es una regla personalizada de **Wazuh** creada específicamente para esta fase del laboratorio.

---

## :link: Relación con Suricata

El flujo completo de esta etapa es:

```text 
Attacker
172.30.0.20
     │
     ▼
DVWA
172.30.0.10
     │
     ▼
Suricata
     │
     │ SID 1000001
     ▼
eve.json
     │
     ▼
Wazuh Agent
     │
     ▼
Wazuh Manager
     │
     ▼
Regla base 86601
     │
     ▼
CDB Threat Intelligence
     │
     │ IOC Match
     ▼
Regla 100500
     │
     ▼
Level 12
```

Los identificadores representan funciones diferentes:

| ID        | Componente | Función                                    |
| --------- | ---------- | ------------------------------------------ |
| `1000001` | Suricata   | Detecta el tráfico HTTP                    |
| `86601`   | Wazuh      | Regla base para el evento de Suricata      |
| `100500`  | Wazuh      | Regla personalizada de Threat Intelligence |

---

## :warning: Importante

La coincidencia con:

```text 
PurpleWolf-C2-high
```

**no demuestra por sí sola que exista un compromiso real**.

En este laboratorio los indicadores son **simulados** y sirven para practicar el proceso de enriquecimiento y priorización de alertas.

La idea es:

```text 
Detección
   ↓
IOC observado
   ↓
Consulta TI
   ↓
Contexto
   ↓
Alerta priorizada
   ↓
Investigación
```

---

## :brain: Lo esencial

Para este laboratorio debemos recordar:

* **Threat Intelligence** aporta contexto sobre indicadores.
* Nuestro IOC principal es `172.30.0.20`.
* Los IOC se almacenan en una **CDB List**.
* La CDB utiliza el formato `clave:valor`.
* `1000001` → regla de **Suricata**.
* `86601` → regla base de **Wazuh**.
* `100500` → regla personalizada de **Threat Intelligence en Wazuh**.
* La coincidencia con la CDB hace que el evento sea procesado por la regla `100500`.
* `100500` genera una alerta de **Level 12**.
* La CDB aporta los indicadores; **la regla 100500 decide qué hacer con la coincidencia**.