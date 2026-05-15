# Guía de Instalación: Clúster WSO2 API Manager 4.6.0.22 en Rocky Linux 9

A continuación tienes la guía probada en entornos reales, optimizada para tu PoC. Asumimos que el paquete ya está descomprimido en `/opt/wso2am` en **ambos** nodos.

**Arquitectura del laboratorio:**

| Nodo    | IP              | Hostname            | Rol                          |
|---------|-----------------|---------------------|------------------------------|
| nodo01  | 192.168.123.42  | nodo01.lab.local    | APIM all-in-one + MariaDB    |
| nodo02  | 192.168.123.43  | nodo02.lab.local    | APIM all-in-one              |

---

## FASE 1 — Preparación del Sistema (ejecutar en AMBOS nodos)

### 1.1 Actualizar el sistema y deshabilitar SELinux (recomendado para lab)

```bash
sudo dnf update -y
sudo dnf install -y wget unzip nano net-tools tar policycoreutils-python-utils

# Para laboratorio: SELinux en modo permisivo (en producción configura políticas)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

### 1.2 Configurar `/etc/hosts` (en ambos nodos, idéntico)

```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.123.42   nodo01 nodo01.lab.local
192.168.123.43   nodo02 nodo02.lab.local
EOF
```

Verifica la resolución cruzada:

```bash
ping -c 2 nodo01
ping -c 2 nodo02
```

### 1.3 Instalar Java 11 (Temurin — recomendado por WSO2)

```bash
sudo tee /etc/yum.repos.d/adoptium.repo <<'EOF'
[Adoptium]
name=Adoptium
baseurl=https://packages.adoptium.net/artifactory/rpm/rocky/$releasever/$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.adoptium.net/artifactory/api/gpg/key/public
EOF

sudo dnf install -y temurin-11-jdk
```

### 1.4 Configurar `JAVA_HOME` a nivel global

```bash
sudo tee /etc/profile.d/java.sh <<'EOF'
export JAVA_HOME=/usr/lib/jvm/temurin-11-jdk
export PATH=$JAVA_HOME/bin:$PATH
EOF

sudo chmod +x /etc/profile.d/java.sh
source /etc/profile.d/java.sh
java -version    # Verifica
```

### 1.5 Crear el usuario `wso2user` y asignar permisos

```bash
sudo useradd -m -s /bin/bash wso2user
sudo passwd wso2user      # Asigna una contraseña

# Propietario del directorio de instalación
sudo chown -R wso2user:wso2user /opt/wso2am
```

### 1.6 Ajustar límites del sistema (importante para Carbon)

```bash
sudo tee -a /etc/security/limits.conf <<'EOF'
wso2user   soft   nofile   65536
wso2user   hard   nofile   65536
wso2user   soft   nproc    20000
wso2user   hard   nproc    20000
EOF

sudo tee -a /etc/sysctl.conf <<'EOF'
fs.file-max = 2097152
net.ipv4.tcp_fin_timeout = 30
EOF

sudo sysctl -p
```

### 1.7 Configurar firewalld (puertos esenciales de WSO2 APIM)

```bash
# Consolas web (Publisher, Dev Portal, Admin, Carbon)
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=9763/tcp

# Gateway
sudo firewall-cmd --permanent --add-port=8243/tcp
sudo firewall-cmd --permanent --add-port=8280/tcp

# Throttling / eventos (Thrift, JMS, Binary)
sudo firewall-cmd --permanent --add-port=9611/tcp
sudo firewall-cmd --permanent --add-port=9711/tcp
sudo firewall-cmd --permanent --add-port=5672/tcp
sudo firewall-cmd --permanent --add-port=9099/tcp

# Hazelcast (clustering WKA)
sudo firewall-cmd --permanent --add-port=4000/tcp

sudo firewall-cmd --reload
```

En **nodo01** además abre el puerto de MariaDB:

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

---

## FASE 2 — Base de Datos MariaDB (solo en nodo01)

### 2.1 Instalar MariaDB 10.6

```bash
sudo dnf install -y mariadb-server mariadb
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

Durante `mysql_secure_installation`: asigna password root, elimina usuarios anónimos, permite acceso remoto **NO**, elimina BD de test, recarga privilegios.

### 2.2 Permitir acceso remoto

```bash
sudo tee /etc/my.cnf.d/wso2-bind.cnf <<'EOF'
[mysqld]
bind-address = 0.0.0.0
max_connections = 500
EOF

sudo systemctl restart mariadb
```

### 2.3 Crear bases de datos y usuario WSO2

```bash
sudo mysql -u root -p
```

Dentro de la consola MariaDB:

```sql
CREATE DATABASE apim_db   CHARACTER SET latin1 COLLATE latin1_swedish_ci;
CREATE DATABASE shared_db CHARACTER SET latin1 COLLATE latin1_swedish_ci;

CREATE USER 'wso2carbon'@'localhost'      IDENTIFIED BY 'Wso2Carbon#2026';
CREATE USER 'wso2carbon'@'192.168.123.42' IDENTIFIED BY 'Wso2Carbon#2026';
CREATE USER 'wso2carbon'@'192.168.123.43' IDENTIFIED BY 'Wso2Carbon#2026';

GRANT ALL PRIVILEGES ON apim_db.*   TO 'wso2carbon'@'localhost';
GRANT ALL PRIVILEGES ON apim_db.*   TO 'wso2carbon'@'192.168.123.42';
GRANT ALL PRIVILEGES ON apim_db.*   TO 'wso2carbon'@'192.168.123.43';
GRANT ALL PRIVILEGES ON shared_db.* TO 'wso2carbon'@'localhost';
GRANT ALL PRIVILEGES ON shared_db.* TO 'wso2carbon'@'192.168.123.42';
GRANT ALL PRIVILEGES ON shared_db.* TO 'wso2carbon'@'192.168.123.43';

FLUSH PRIVILEGES;
EXIT;
```

### 2.4 Ejecutar los scripts de inicialización de WSO2

```bash
# apim_db
mysql -u wso2carbon -p'Wso2Carbon#2026' apim_db   < /opt/wso2am/dbscripts/apimgt/mysql.sql

# shared_db (registro + user store)
mysql -u wso2carbon -p'Wso2Carbon#2026' shared_db < /opt/wso2am/dbscripts/mysql.sql
```

Verifica que se crearon las tablas:

```bash
mysql -u wso2carbon -p'Wso2Carbon#2026' -e "USE apim_db;   SHOW TABLES;" | wc -l
mysql -u wso2carbon -p'Wso2Carbon#2026' -e "USE shared_db; SHOW TABLES;" | wc -l
```

### 2.5 Driver JDBC de MariaDB (en AMBOS nodos)

```bash
cd /opt/wso2am/repository/components/lib
sudo -u wso2user wget https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/2.7.9/mariadb-java-client-2.7.9.jar
sudo chown wso2user:wso2user mariadb-java-client-2.7.9.jar
```

> **Nota:** La versión 2.7.x es la que mejor compatibilidad ha mostrado con WSO2 4.x en producción. Evita la rama 3.x salvo que tengas requisitos específicos.

---

## FASE 3 — Configuración de Certificados (en AMBOS nodos)

Trabajaremos con un keystore por nodo (firmando con CN = hostname propio) y un truststore común con ambos certificados. Esto es lo que ha funcionado bien en entornos reales.

### 3.1 En nodo01 — generar keystore

```bash
sudo su - wso2user
cd /opt/wso2am/repository/resources/security

# Respalda los keystore de fábrica
mv wso2carbon.jks wso2carbon.jks.original
mv client-truststore.jks client-truststore.jks.original

# Keystore del nodo01
keytool -genkey -alias wso2carbon -keyalg RSA -keysize 2048 \
  -keystore wso2carbon.jks \
  -dname "CN=nodo01.lab.local,OU=APIM,O=Lab,L=Madrid,ST=Madrid,C=ES" \
  -storepass wso2carbon -keypass wso2carbon -validity 3650 \
  -ext SAN=DNS:nodo01,DNS:nodo01.lab.local,DNS:localhost,IP:192.168.123.42,IP:127.0.0.1

# Exportar el certificado público
keytool -export -alias wso2carbon -keystore wso2carbon.jks \
  -file /tmp/nodo01-pub.crt -storepass wso2carbon
```

### 3.2 En nodo02 — generar keystore (mismo procedimiento, distinto CN/SAN)

```bash
sudo su - wso2user
cd /opt/wso2am/repository/resources/security

mv wso2carbon.jks wso2carbon.jks.original
mv client-truststore.jks client-truststore.jks.original

keytool -genkey -alias wso2carbon -keyalg RSA -keysize 2048 \
  -keystore wso2carbon.jks \
  -dname "CN=nodo02.lab.local,OU=APIM,O=Lab,L=Madrid,ST=Madrid,C=ES" \
  -storepass wso2carbon -keypass wso2carbon -validity 3650 \
  -ext SAN=DNS:nodo02,DNS:nodo02.lab.local,DNS:localhost,IP:192.168.123.43,IP:127.0.0.1

keytool -export -alias wso2carbon -keystore wso2carbon.jks \
  -file /tmp/nodo02-pub.crt -storepass wso2carbon
```

### 3.3 Intercambiar los certificados públicos entre nodos

Desde nodo01 (como `wso2user`):

```bash
scp /tmp/nodo01-pub.crt wso2user@nodo02:/tmp/
scp wso2user@nodo02:/tmp/nodo02-pub.crt /tmp/
```

### 3.4 Construir el `client-truststore.jks` (en AMBOS nodos)

Crea el truststore importando primero las CAs por defecto de Java y luego ambos certificados WSO2:

```bash
cd /opt/wso2am/repository/resources/security

# Crear truststore base copiando el cacerts de Java (CAs raíz) — opcional pero recomendado
cp $JAVA_HOME/lib/security/cacerts client-truststore.jks
keytool -storepasswd -keystore client-truststore.jks \
  -storepass changeit -new wso2carbon

# Importar certificado de nodo01
keytool -import -trustcacerts -alias wso2carbon-nodo01 \
  -file /tmp/nodo01-pub.crt \
  -keystore client-truststore.jks -storepass wso2carbon -noprompt

# Importar certificado de nodo02
keytool -import -trustcacerts -alias wso2carbon-nodo02 \
  -file /tmp/nodo02-pub.crt \
  -keystore client-truststore.jks -storepass wso2carbon -noprompt

# Verificar
keytool -list -keystore client-truststore.jks -storepass wso2carbon | grep -i wso2carbon
```

> **Convención de paths que usaremos en deployment.toml:**
> - Keystore: `${carbon.home}/repository/resources/security/wso2carbon.jks`
> - Truststore: `${carbon.home}/repository/resources/security/client-truststore.jks`

---

## FASE 4 — `deployment.toml` (uno por nodo)

Ruta: `/opt/wso2am/repository/conf/deployment.toml`

### 4.1 `deployment.toml` para **NODO01**

```toml
# ============================================================
# WSO2 API Manager 4.6.0.22 — NODO 01 (192.168.123.42)
# ============================================================

[server]
hostname = "nodo01.lab.local"
node_ip = "192.168.123.42"
server_role = "default"
offset = 0

[super_admin]
username = "admin"
password = "admin"
create_admin_account = true

[user_store]
type = "database_unique_id"

# --------- Bases de datos centralizadas ---------
[database.apim_db]
type     = "mariadb"
url      = "jdbc:mariadb://nodo01.lab.local:3306/apim_db?useSSL=false"
username = "wso2carbon"
password = "Wso2Carbon#2026"
driver   = "org.mariadb.jdbc.Driver"

[database.apim_db.pool_options]
maxActive       = 150
maxWait         = 60000
validationQuery = "SELECT 1"

[database.shared_db]
type     = "mariadb"
url      = "jdbc:mariadb://nodo01.lab.local:3306/shared_db?useSSL=false"
username = "wso2carbon"
password = "Wso2Carbon#2026"
driver   = "org.mariadb.jdbc.Driver"

[database.shared_db.pool_options]
maxActive       = 150
maxWait         = 60000
validationQuery = "SELECT 1"

# --------- Keystore / Truststore ---------
[keystore.tls]
file_name    = "wso2carbon.jks"
type         = "JKS"
password     = "wso2carbon"
alias        = "wso2carbon"
key_password = "wso2carbon"

[truststore]
file_name = "client-truststore.jks"
type      = "JKS"
password  = "wso2carbon"

# --------- Clúster (Hazelcast WKA) ---------
[clustering]
membership_scheme = "wka"
domain            = "wso2.apim.domain"
local_member_host = "192.168.123.42"
local_member_port = "4000"
members           = ["192.168.123.42:4000", "192.168.123.43:4000"]

# --------- Inbuilt Artifact Synchronization (DB, sin NFS) ---------
[apim.sync_runtime_artifacts.gateway]
gateway_labels = ["Default"]

[apim.sync_runtime_artifacts.gateway.event_listener_configuration]
type = "db"

# --------- Gateway Environment ---------
[[apim.gateway.environment]]
name                       = "Default"
type                       = "hybrid"
display_in_api_console     = true
description                = "Cluster Gateway PoC"
show_as_token_endpoint_url = true
service_url                = "https://localhost:9443/services/"
username                   = "${admin.username}"
password                   = "${admin.password}"
ws_endpoint                = "ws://nodo01.lab.local:9099"
wss_endpoint               = "wss://nodo01.lab.local:8099"
http_endpoint              = "http://nodo01.lab.local:8280"
https_endpoint             = "https://nodo01.lab.local:8243"

# --------- Event Hub (sincronización de eventos entre nodos) ---------
[apim.throttling]
enable_data_publishing       = true
enable_policy_deploy_cluster = true

[apim.event_hub]
enable                = true
service_url           = "https://nodo01.lab.local:${mgt.transport.https.port}/services/"
event_listening_endpoints = ["tcp://nodo01.lab.local:5672", "tcp://nodo02.lab.local:5672"]
```

### 4.2 `deployment.toml` para **NODO02**

Es idéntico salvo `hostname`, `node_ip`, `local_member_host` y los endpoints del gateway. Las diferencias clave están **marcadas**:

```toml
# ============================================================
# WSO2 API Manager 4.6.0.22 — NODO 02 (192.168.123.43)
# ============================================================

[server]
hostname    = "nodo02.lab.local"   # <-- DIFERENTE
node_ip     = "192.168.123.43"     # <-- DIFERENTE
server_role = "default"
offset      = 0

[super_admin]
username = "admin"
password = "admin"
create_admin_account = true

[user_store]
type = "database_unique_id"

# --------- Bases de datos (apuntan SIEMPRE a nodo01) ---------
[database.apim_db]
type     = "mariadb"
url      = "jdbc:mariadb://nodo01.lab.local:3306/apim_db?useSSL=false"
username = "wso2carbon"
password = "Wso2Carbon#2026"
driver   = "org.mariadb.jdbc.Driver"

[database.apim_db.pool_options]
maxActive       = 150
maxWait         = 60000
validationQuery = "SELECT 1"

[database.shared_db]
type     = "mariadb"
url      = "jdbc:mariadb://nodo01.lab.local:3306/shared_db?useSSL=false"
username = "wso2carbon"
password = "Wso2Carbon#2026"
driver   = "org.mariadb.jdbc.Driver"

[database.shared_db.pool_options]
maxActive       = 150
maxWait         = 60000
validationQuery = "SELECT 1"

# --------- Keystore / Truststore ---------
[keystore.tls]
file_name    = "wso2carbon.jks"
type         = "JKS"
password     = "wso2carbon"
alias        = "wso2carbon"
key_password = "wso2carbon"

[truststore]
file_name = "client-truststore.jks"
type      = "JKS"
password  = "wso2carbon"

# --------- Clúster (Hazelcast WKA) ---------
[clustering]
membership_scheme = "wka"
domain            = "wso2.apim.domain"
local_member_host = "192.168.123.43"             # <-- DIFERENTE
local_member_port = "4000"
members           = ["192.168.123.42:4000", "192.168.123.43:4000"]

# --------- Inbuilt Artifact Synchronization ---------
[apim.sync_runtime_artifacts.gateway]
gateway_labels = ["Default"]

[apim.sync_runtime_artifacts.gateway.event_listener_configuration]
type = "db"

# --------- Gateway Environment ---------
[[apim.gateway.environment]]
name                       = "Default"
type                       = "hybrid"
display_in_api_console     = true
description                = "Cluster Gateway PoC"
show_as_token_endpoint_url = true
service_url                = "https://localhost:9443/services/"
username                   = "${admin.username}"
password                   = "${admin.password}"
ws_endpoint                = "ws://nodo02.lab.local:9099"          # <-- DIFERENTE
wss_endpoint               = "wss://nodo02.lab.local:8099"         # <-- DIFERENTE
http_endpoint              = "http://nodo02.lab.local:8280"        # <-- DIFERENTE
https_endpoint             = "https://nodo02.lab.local:8243"       # <-- DIFERENTE

# --------- Event Hub ---------
[apim.throttling]
enable_data_publishing       = true
enable_policy_deploy_cluster = true

[apim.event_hub]
enable                    = true
service_url               = "https://nodo01.lab.local:${mgt.transport.https.port}/services/"
event_listening_endpoints = ["tcp://nodo01.lab.local:5672", "tcp://nodo02.lab.local:5672"]
```

> **Diferencias clave a recordar entre los nodos:**
> 1. `[server] hostname` y `node_ip`
> 2. `[clustering] local_member_host` (la IP propia)
> 3. Los endpoints del `[[apim.gateway.environment]]` apuntan al hostname propio
> 4. `offset = 0` en ambos: como están en VMs distintas, no hay colisión de puertos
> 5. La lista `members` es **idéntica** en los dos nodos

---

## FASE 5 — Notas Críticas sobre el Bloque `[clustering]`

El bloque `[clustering]` ya está incluido en los TOML. Puntos clave que conviene revisar manualmente:

```toml
[clustering]
membership_scheme = "wka"                                # Well-Known Address: lista estática
domain            = "wso2.apim.domain"                   # Debe coincidir en todos los nodos
local_member_host = "<IP de este nodo>"                  # CAMBIA por nodo
local_member_port = "4000"                               # Puerto Hazelcast (TCP)
members           = ["192.168.123.42:4000",
                    "192.168.123.43:4000"]               # Lista IDÉNTICA en todos los nodos
```

- `domain` **tiene que ser exactamente igual** en ambos nodos — si difiere, Hazelcast crea dos clústeres separados de un miembro cada uno.
- `members` se mantiene completa en todos los nodos (incluso incluye al propio).
- El puerto 4000/TCP **tiene que estar abierto** en firewalld entre nodos (ya lo abrimos en la Fase 1.7).
- Si más adelante agregas un tercer nodo, basta con extender `members` en los tres TOMLs y reiniciar.

---

## FASE 6 — Inicio y Verificación

### 6.1 Permisos finales antes del arranque (ambos nodos)

```bash
sudo chown -R wso2user:wso2user /opt/wso2am
```

### 6.2 Arrancar **primero nodo01**, luego nodo02

En **nodo01**:

```bash
sudo su - wso2user
cd /opt/wso2am/bin
./api-manager.sh start

# Seguir los logs en tiempo real
tail -f /opt/wso2am/repository/logs/wso2carbon.log
```

Espera a ver `WSO2 Carbon started in XXX sec`. Luego arranca **nodo02** del mismo modo.

### 6.3 Confirmar la formación del clúster

En el log de **cualquiera** de los dos nodos busca estas líneas (señales positivas):

```bash
grep -E "HazelcastClusteringAgent|Member.*joined|Cluster initialization completed|HazelcastInstance" \
   /opt/wso2am/repository/logs/wso2carbon.log
```

Mensajes esperados:

```
INFO  - HazelcastClusteringAgent Initializing cluster ...
INFO  - HazelcastClusteringAgent Using WKA based membership management scheme
INFO  - HazelcastClusteringAgent Local member: [<UUID>] - Host:192.168.123.42, Remote Host:null, Port: 4000, ...
INFO  - HazelcastClusteringAgent Cluster initialization completed
INFO  - HazelcastClusteringAgent Member joined [<UUID>]: /192.168.123.43:4000
```

Si ves `Members [2]` en la línea de Hazelcast, el clúster está formado correctamente.

### 6.4 Comprobaciones adicionales

**Puertos en escucha** (en ambos nodos):

```bash
ss -tlnp | grep -E "9443|8243|8280|4000"
```

**Conectividad Hazelcast desde nodo01 a nodo02:**

```bash
nc -zv 192.168.123.43 4000
```

**Acceso a consolas (desde tu equipo, con `nodo01`/`nodo02` resueltos en tu hosts local):**

| Consola      | URL                                            |
|--------------|------------------------------------------------|
| Publisher    | https://nodo01.lab.local:9443/publisher        |
| Dev Portal   | https://nodo01.lab.local:9443/devportal        |
| Admin Portal | https://nodo01.lab.local:9443/admin            |
| Carbon       | https://nodo01.lab.local:9443/carbon           |

(Sustituye `nodo01` por `nodo02` para acceder al otro nodo.)

**Validación del Inbuilt Artifact Synchronization:**

1. Crea una API simple en `https://nodo01:9443/publisher` y haz **Deploy → Default Gateway**.
2. Espera ~30s y consulta la misma API en el gateway de nodo02:
   ```bash
   curl -k https://nodo02.lab.local:8243/<context>/<version>/<resource>
   ```
3. Si responde correctamente, la sincronización vía DB está funcionando — **no se requiere NFS**.

---

## Apuntes finales que evitan problemas en producción

- **No uses `offset` distinto en cada nodo**: están en VMs separadas, no hace falta. Cambiar offsets rompe los endpoints generados.
- **El driver MariaDB** debe estar en `repository/components/lib` **antes** del primer arranque, no en `dropins`.
- **Si el clúster no se forma**: revisa primero `firewalld` (puerto 4000), después el campo `domain` y la coincidencia exacta de `members` en ambos TOMLs.
- **Caracteres especiales en passwords de BD**: si usas `@`, `#`, `$`, escápalos en el TOML o cámbialos por símbolos seguros (`-`, `_`). El `#` en `Wso2Carbon#2026` funciona porque está entre comillas.
- **Backups antes de modificar**: siempre `cp deployment.toml deployment.toml.bak` antes de cualquier cambio.

Cuando termines la PoC y vayas a producción, los puntos a revisar son: certificados firmados por CA interna (no autofirmados), load balancer frontal (HAProxy/Nginx) con sticky sessions para las consolas, separación lógica BD ↔ app, y backups regulares de `apim_db` y `shared_db`.
