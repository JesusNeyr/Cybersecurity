# :shield: 02 — Wazuh

Arquitectura **single-node** en VM1, donde se ejecutan mediante Docker los tres componentes centrales:

* Wazuh Manager
* Wazuh Indexer
* Wazuh Dashboard

En VM2 se instala el **Wazuh Agent**, que posteriormente será utilizado para recopilar la telemetría generada por Suricata y otros componentes del laboratorio.

---

# :dart: Objetivo

Al finalizar esta etapa tendremos:

![](../img/flujo_completo.png)

El resultado esperado es que el Agent de VM2 aparezca conectado y activo en Wazuh.

---

# :computer: Arquitectura de Wazuh

## VM1 — CyberSOC

```text
IP: 192.168.56.10
```

Componentes:

```text
Wazuh Manager
Wazuh Indexer
Wazuh Dashboard
```

El contenedor se ejecuta mediante Docker en:

```text
/opt/wazuh-docker/single-node
```
---

## VM2 — CyberRange

```text
IP: 192.168.56.20
```

Componente:

```text
Wazuh Agent
```

El agente sera nombrado como:

```text
cyberrange-suricata
```

---

# :arrows_counterclockwise: Flujo de Wazuh

La comunicación básica queda de la siguiente manera:

```text
    VM2 - CyberRange ─────── eventos ────────> VM1 - CyberSOC
```

---

# :one: Wazuh en VM1 — CyberSOC

Todos los pasos de esta sección se realizan en:

```text
VM1 — CyberSOC
192.168.56.10
```

---
## :whale: 1.1 Instalación de Docker

Wazuh se desplegará mediante **Docker** y **Docker Compose**. Por este motivo, Docker debe estar instalado y funcionando tanto en la **VM 1 (CyberSOC)** como en la **VM 2 (Cyberrange)**.

### :computer: 1.1 Actualizar paquetes e instalar dependencias

Ejecutar en **VM 1 y VM 2**:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

Estos paquetes permiten descargar y validar los componentes necesarios para agregar el repositorio oficial de Docker.

### :key: 1.1.2 Crear el directorio para la clave GPG

Crear el directorio donde se almacenará la clave utilizada para verificar los paquetes del repositorio de Docker:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

### :key: 1.1.3 Descargar la clave GPG de Docker

Descargamos la clave oficial del repositorio:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Establecemos permisos de lectura para que APT pueda utilizar la clave:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### :package: 1.1.4 Agregar el repositorio oficial de Docker

Creamos la configuración del repo oficial de Docker:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

El comando obtiene automáticamente el nombre de la versión de Ubuntu instalada y la arquitectura del sistema, evitando tener que escribir esos valores manualmente.

### :arrows_counterclockwise: 1.1.5 Actualizar los repositorios

Una vez agregado el repositorio de Docker:

```bash
sudo apt update
```

### :whale: 1.1.6 Instalar Docker Engine y Docker Compose

Instalar Docker Engine junto con los componentes necesarios para utilizar Docker Compose:

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Los componentes principales son:

* `docker-ce`: Docker Engine.
* `docker-ce-cli`: interfaz de línea de comandos de Docker.
* `containerd.io`: runtime utilizado por Docker.
* `docker-buildx-plugin`: herramienta para realizar builds de imágenes.
* `docker-compose-plugin`: permite utilizar `docker compose` para definir y administrar aplicaciones compuestas por varios contenedores.

### :white_check_mark: 1.1.7 Verificar el servicio Docker

Comprobar que el servicio se encuentre iniciado:

```bash
sudo systemctl status docker --no-pager
```

Debe aparecer:

```text
active (running)
```

Si Docker está instalado pero no se encuentra iniciado, puede iniciarse mediante:

```bash
sudo systemctl start docker
```

Para que Docker se inicie automáticamente con el sistema:

```bash
sudo systemctl enable docker
```

---

# :gear: 2. Preparar el sistema para Wazuh Indexer en VM1 CyberSoc

Wazuh Indexer requiere aumentar el parámetro del kernel:

```bash
sudo sysctl -w vm.max_map_count=262144
```

La documentación oficial de Wazuh establece `262144` como valor requerido para el despliegue Docker.

Esto aplica el valor inmediatamente.

Para que la configuración permanezca después de reiniciar la máquina, se debe crear una configuración persistente.

Ejecutar:

```bash
sudo tee /etc/sysctl.d/99-wazuh.conf >/dev/null <<EOF
vm.max_map_count=262144
EOF
```

El archivo creado es:

```text
/etc/sysctl.d/99-wazuh.conf
```

Su contenido debe ser:

```text
vm.max_map_count=262144
```

---

## :mag: 2.1 Verificar el valor

Ejecutar:

```bash
sysctl vm.max_map_count
```

Resultado esperado:

```text
vm.max_map_count = 262144
```

También se puede comprobar directamente el archivo:

```bash
cat /etc/sysctl.d/99-wazuh.conf
```

Debe mostrar:

```text
vm.max_map_count=262144
```

> :white_check_mark: No continuar con el despliegue de Wazuh si este valor no es `262144`.

---

# :package: 3. Obtener Wazuh Docker

Usaremos para el lab el repo oficial de `wazuh-docker` y la versión:

```text
v4.14.7
```

---

## :wrench: 3.1 Instalar Git

Si Git todavía no está instalado en VM1 ni en VM2:

```bash
sudo apt update
sudo apt install -y git
```

Verificar:

```bash
git --version
```

---

## :arrow_down: 3.2 Descargar Wazuh Docker

El repositorio se ubicará dentro de `/opt`.

Ejecutar:

```bash
cd /opt
```

Clonar la versión utilizada:

```bash
sudo git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.7
```

El resultado será:

```text
/opt/wazuh-docker
```

---

## :file_folder: 3.3 Entrar al despliegue single-node

El laboratorio utiliza la modalidad single-node.

Ejecutar:

```bash
cd /opt/wazuh-docker/single-node
```

La ruta debe quedar:

```text
/opt/wazuh-docker/single-node
```

---

# :lock: 4. Generar los certificados

Los componentes de Wazuh utilizan comunicación segura entre ellos.

El despliegue Docker single-node proporciona el archivo:

```text
generate-indexer-certs.yml
```

que se utiliza para generar los certificados necesarios.

Desde:

```text
/opt/wazuh-docker/single-node
```

ejecutar:

```bash
sudo docker compose -f generate-indexer-certs.yml run --rm generator
```

El contenedor utilizado para generar los certificados se ejecuta temporalmente y se elimina después de finalizar.

---

## :mag: 4.1 Verificar los certificados

Ejecutar:

```bash
sudo ls config/wazuh_indexer_ssl_certs
```

La existencia de archivos en este directorio indica que la generación terminó correctamente.

---

# :rocket: 5. Iniciar Wazuh

Mantenerse en:

```text
/opt/wazuh-docker/single-node
```

Ejecutar:

```bash
sudo docker compose up -d
```

El parámetro `-d` inicia los contenedores en segundo plano.

El stack incluye:

```text
Wazuh Manager
Wazuh Indexer
Wazuh Dashboard
```

---

# :mag: 5.1 Verificar los contenedores

Ejecutar:

```bash
sudo docker compose ps
```

Se deben visualizar los servicios correspondientes al contenedor.

Se espera que los contenedores se encuentren ejecutándose.

Una comprobación típica debe mostrar los servicios:

```text
wazuh.manager
wazuh.indexer
wazuh.dashboard
```

---

## :warning: 5.2 Si algún componente no inicia

Si algún contenedor aparece detenido o no saludable, consultar los logs desde el mismo directorio:

```bash
sudo docker compose logs --tail=100
```

Esto permite revisar los mensajes recientes de los componentes.

No asumir que el despliegue falló únicamente porque el Dashboard tarda en estar disponible.

---

# :desktop_computer: 6. Verificar Wazuh Dashboard

Desde VM1 se puede realizar una prueba local:

```bash
curl -k -I https://localhost
```
---

## :globe_with_meridians: 6.1 Acceder desde el navegador

Desde un navegador que tenga conectividad con VM1:

```text
https://192.168.56.10
```

El Dashboard debe mostrar la pantalla de inicio de sesión.

Acceso: 
URL:        https://192.168.56.10
Usuario:    admin
Contraseña: SecretPassword

---

# :computer: 7. Wazuh Agent en VM2 - CyberRange

Una vez que el stack central funciona correctamente, se configura el Agent.

Los siguientes pasos se realizan en:

```text
VM2 — CyberRange
```

El Agent permitirá que VM2 envíe telemetría al Manager ubicado en:

```text
192.168.56.10
```

---

# :key: 7.1 Agregar el repositorio de Wazuh

Actualizar los paquetes:

```bash
sudo apt update
```

Instalar los paquetes necesarios:

```bash
sudo apt install -y gnupg apt-transport-https curl
```

Importar la clave del repositorio de Wazuh:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
```

Ajustar los permisos:

```bash
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
```

Agregar el repositorio:

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list
```

Actualizar la información de paquetes:

```bash
sudo apt update
```

---

# :package: 8. Instalar el Wazuh Agent

El Agent se instalará con los siguientes parámetros:

| Parámetro           | Valor                 |
| ------------------- | --------------------- |
| Wazuh Manager       | `192.168.56.10`       |
| Registration Server | `192.168.56.10`       |
| Nombre del Agent    | `cyberrange-suricata` |

La dirección del Manager es importante porque determina hacia qué servidor debe conectarse el Agent.

La instalación se realiza mediante la definicions de variables:

```bash
sudo env \
  WAZUH_MANAGER="192.168.56.10" \
  WAZUH_REGISTRATION_SERVER="192.168.56.10" \
  WAZUH_AGENT_NAME="cyberrange-suricata" \
  apt-get install -y wazuh-agent
```

---

# :arrows_counterclockwise: 9. Iniciar el Agent

Después de instalar el paquete, recargar la configuración de servicios:

```bash
sudo systemctl daemon-reload
```

Habilitar el servicio para que se inicie automáticamente y arrancarlo:

```bash
sudo systemctl enable --now wazuh-agent
```

---

# :mag: 9.1 Verificar el servicio

Ejecutar:

```bash
sudo systemctl status wazuh-agent --no-pager
```

El estado esperado es:

```text
Active: active (running)
```

Si el servicio está detenido, revisar los mensajes del servicio antes de continuar.

---

# :satellite: 9.2 Verificar la conexión con el Manager

El log principal del Agent se encuentra en:

```text
/var/ossec/logs/ossec.log
```

Buscar mensajes relacionados con la conexión:

```bash
sudo grep -E "Connected to|Unable to connect" \
  /var/ossec/logs/ossec.log | tail
```

Un resultado esperado es:

```text
Connected to the server
```

---

# :computer: 10. Verificar el Agent desde VM1

La conexión también debe comprobarse desde el lado del Manager.

Volver a:

```text
VM1 — CyberSOC
```

Entrar al directorio del despliegue:

```bash
cd /opt/wazuh-docker/single-node
```

Ejecutar:

```bash
sudo docker compose exec wazuh.manager \
  /var/ossec/bin/agent_control -lc
```

Este comando consulta los agentes registrados desde el propio Wazuh Manager.

---

## :white_check_mark: Resultado esperado

Debe aparecer el agente:

```text
cyberrange-suricata    Active
```
confirma que el Manager reconoce al Agent y que existe comunicación entre ambos.

---

# :mag_right: 11. Comprobación final de Wazuh

Antes de continuar con Suricata, comprobar:

### VM1 — CyberSOC

```text
[ ] IP configurada: 192.168.56.10
[ ] Docker está activo
[ ] vm.max_map_count = 262144
[ ] Repositorio wazuh-docker v4.14.7 disponible
[ ] /opt/wazuh-docker/single-node existe
[ ] Certificados generados
[ ] Wazuh Manager está ejecutándose
[ ] Wazuh Indexer está ejecutándose
[ ] Wazuh Dashboard está ejecutándose
[ ] Dashboard accesible mediante HTTPS
```

### VM2 — CyberRange

```text
[ ] IP configurada: 192.168.56.20
[ ] Repositorio de Wazuh configurado
[ ] Wazuh Agent instalado
[ ] Agent habilitado
[ ] wazuh-agent está active (running)
[ ] Agent puede comunicarse con 192.168.56.10
[ ] Agent aparece como cyberrange-suricata
[ ] Agent aparece como Active en el Manager
```

---

# :white_check_mark: Resultado de esta etapa

Al finalizar esta etapa, la infraestructura Wazuh debe quedar:


En este punto **Wazuh ya está preparado como plataforma central**, pero todavía no recibe los eventos de Suricata.

La integración de:

```text
    Suricata -> eve.json -> wazuh Agent-> wazuh Manager
```
---

# :arrow_forward: Siguiente paso

Continuar con:

➡️ [03-suricata.md](./03-suricata.md)

En esa etapa se preparará Suricata en VM2, junto con Docker/DVWA/Attacker, la interfaz `br-cybersoc`, la salida `eve.json` y la regla personalizada:

```text
SID: 1000001
CYBERSOC - Acceso HTTP a DVWA
```
