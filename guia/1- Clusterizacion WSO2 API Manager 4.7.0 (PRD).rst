Manual de Implementación: Clúster Activo-Activo WSO2 API Manager 4.7.0
Rocky Linux 9 + JDK 21 + MariaDB (Granja 10.132.0.56) + Balanceador F5
================================================================================

:Versión del producto: WSO2 API Manager 4.7.0 (All-in-One)
:Entorno: Producción (PRD)
:Sistema Operativo: Rocky Linux 9.x (compatible con RHEL 9.6 probado oficialmente)
:Java Runtime: Temurin OpenJDK 21 (probado oficialmente para 4.7.0)
:Base de Datos: MariaDB en granja ``10.132.0.56`` (esquemas ``WSO2AM_DB`` y ``WSO2SHARED_DB``)
:Balanceador: F5 (gestionado por equipo de redes; configuración F5 fuera de alcance)
:Nodos:
   - ``lrkprdappdckposapi01`` (``10.133.4.187``)
   - ``lrkprdappdckposapi02`` (``10.133.4.188``)
:Sincronización de artefactos: Sistema de archivos compartido NFS
:Estado: Guía de referencia basada en documentación oficial WSO2

.. contents:: Tabla de Contenidos
   :depth: 3
   :local:

.. warning::

   **Aviso de certificación**: Este manual se basa en la documentación oficial de
   WSO2 API Manager 4.7.0 y en las prácticas recomendadas de despliegue
   activo-activo. No constituye una "certificación" formal emitida por WSO2.
   Para obtener una validación certificada, se recomienda contactar al soporte
   oficial de WSO2 o a un socio certificado y someter la configuración a revisión.

   **Nota sobre MariaDB**: La matriz oficial de compatibilidad de WSO2 APIM 4.7.0
   no lista MariaDB explícitamente entre las bases de datos probadas [citation:21]. MariaDB es
   altamente compatible con MySQL y funciona con el driver MariaDB Connector/J,
   pero **queda fuera del alcance de las pruebas oficiales**. Se recomienda
   validar exhaustivamente en un entorno de pruebas antes de producción.

================================================================================
0. Topología y Preparación Previa
================================================================================

0.1 Topología Objetivo
----------------------

.. list-table:: Nodos y Componentes
   :header-rows: 1
   :widths: 30 30 40

   * - Rol
     - Hostname
     - IP / Identificador
   * - Nodo APIM 1
     - ``lrkprdappdckposapi01``
     - ``10.133.4.187``
   * - Nodo APIM 2
     - ``lrkprdappdckposapi02``
     - ``10.133.4.188``
   * - Granja MariaDB
     - ``mariadb-prd-cluster``
     - ``10.132.0.56``
   * - VIP F5
     - ``api.tu-dominio.com``
     - Gestionada por equipo de redes

0.2 Elementos a Preparar de Antemano
-------------------------------------

- Archivo de certificado SSL del dominio (``.crt``) y clave privada (``.key``).
- Al menos un recurso **NFS compartido** para montar en
  ``repository/deployment/server`` de ambos nodos.
- Usuario de base de datos con permisos para crear bases de datos y tablas, o
  DBA que proporcione las bases ya creadas.
- Acceso root/sudo a ambos nodos Rocky Linux 9.
- Conectividad de red verificada entre ambos nodos y hacia ``10.132.0.56:3306``.

**Consultas obligatorias al equipo de DBA (MariaDB 10.132.0.56)**:

.. list-table:: Puntos a Confirmar con el Equipo de Base de Datos
   :header-rows: 1
   :widths: 30 70

   * - Tema
     - Acción requerida
   * - Política SSL
     - Confirmar si la cadena JDBC debe usar ``useSSL=true`` (con
       ``trustServerCertificate=true`` o certificado importado al truststore)
       o ``useSSL=false``. **No asumir ``useSSL=false`` sin validación**.
   * - Codificación de caracteres
     - Confirmar si los esquemas ``WSO2AM_DB`` y ``WSO2SHARED_DB`` usan
       ``latin1_bin`` (recomendado para WSO2) o ``utf8mb4``. La codificación
       debe ser **consistente** entre ambas bases de datos y el parámetro
       ``characterEncoding`` en la URL JDBC si aplica.
   * - Versión exacta de MariaDB
     - Registrar la versión del servidor (``SELECT VERSION();``) para
       referencia y troubleshooting.
   * - Driver JDBC recomendado
     - Confirmar la versión de MariaDB Connector/J que el equipo de DBA
       considera estable para producción.

================================================================================
1. Ajustes del Sistema Operativo (EJECUTAR ANTES DEL SERVICIO)
================================================================================

.. important::

   **Ejecutar esta fase completa antes de crear o habilitar el servicio
   ``wso2apim.service``.** Los límites de ``nofile`` y ``nproc`` deben estar
   vigentes para el usuario ``wso2`` antes de que systemd inicie el proceso
   Java. Si se configuran después, el proceso heredará los valores
   predeterminados del sistema y las restricciones no aplicarán.

**Realizar en ambos nodos:**

1.1 Límites de Descriptores de Archivos
----------------------------------------

.. code-block:: bash

   sudo tee /etc/security/limits.d/99-wso2.conf > /dev/null << 'EOF'
   wso2 soft nofile 65535
   wso2 hard nofile 65535
   wso2 soft nproc 32768
   wso2 hard nproc 32768
   EOF

Verificar que el usuario ``wso2`` puede aplicar los límites (requiere sesión
nueva o ``su - wso2 -s /bin/bash -c "ulimit -n"``).

1.2 Parámetros del Kernel
--------------------------

.. code-block:: bash

   sudo tee /etc/sysctl.d/99-wso2.conf > /dev/null << 'EOF'
   vm.max_map_count = 262144
   net.core.somaxconn = 4096
   EOF
   sudo sysctl --system

Verificar:

.. code-block:: bash

   sysctl vm.max_map_count net.core.somaxconn

1.3 Sincronización Horaria
---------------------------

.. code-block:: bash

   sudo systemctl enable --now chronyd
   chronyc tracking
   # Verificar que el offset sea bajo (< 10 ms idealmente)

================================================================================
2. Descarga e Instalación de JDK 21
================================================================================

**Realizar en ambos nodos.**

Según la matriz de compatibilidad oficial, WSO2 APIM 4.7.0 fue probado con
**Temurin OpenJDK 21**.

.. code-block:: bash

   # Rocky Linux 9: instalar Temurin 21
   sudo dnf install -y temurin-21-jdk
   # Verificar versión
   java -version
   # La salida debe mostrar 21.x

Configurar ``JAVA_HOME`` creando ``/etc/profile.d/java.sh``:

.. code-block:: bash

   export JAVA_HOME=/usr/lib/jvm/temurin-21-jdk
   export PATH=$JAVA_HOME/bin:$PATH

Verificar:

.. code-block:: bash

   source /etc/profile.d/java.sh
   echo $JAVA_HOME

================================================================================
3. Descarga e Instalación de WSO2 API Manager 4.7.0
================================================================================

**Realizar en ambos nodos.**

Descargar ``wso2am-4.7.0.zip`` desde la página oficial de productos de WSO2.

.. code-block:: bash

   # Crear usuario del sistema
   sudo groupadd -r wso2
   sudo useradd -r -g wso2 -d /opt/wso2 -s /sbin/nologin wso2

   # Descomprimir
   sudo mkdir -p /opt/wso2
   sudo unzip wso2am-4.7.0.zip -d /opt/wso2/
   sudo chown -R wso2:wso2 /opt/wso2/wso2am-4.7.0

.. note::

   **No iniciar el servicio todavía.** Primero completar toda la configuración
   de las fases siguientes.

================================================================================
4. Creación de Bases de Datos MariaDB y Driver JDBC
================================================================================

4.1 Creación de Esquemas y Usuario
-----------------------------------

**Ejecutar en el servidor de base de datos** ``10.132.0.56`` (o solicitar al DBA).
Ajustar la codificación según lo confirmado con el equipo de DBA (ver sección 0.2).

.. code-block:: sql

   -- Verificar codificación con DBA antes de ejecutar
   CREATE DATABASE WSO2AM_DB CHARACTER SET latin1 COLLATE latin1_bin;
   CREATE DATABASE WSO2SHARED_DB CHARACTER SET latin1 COLLATE latin1_bin;

   CREATE USER 'wso2_prd_user'@'10.133.4.187' IDENTIFIED BY 'TU_PASSWORD_SEGURO_BD';
   CREATE USER 'wso2_prd_user'@'10.133.4.188' IDENTIFIED BY 'TU_PASSWORD_SEGURO_BD';

   GRANT ALL PRIVILEGES ON WSO2AM_DB.* TO 'wso2_prd_user'@'10.133.4.187';
   GRANT ALL PRIVILEGES ON WSO2SHARED_DB.* TO 'wso2_prd_user'@'10.133.4.187';
   GRANT ALL PRIVILEGES ON WSO2AM_DB.* TO 'wso2_prd_user'@'10.133.4.188';
   GRANT ALL PRIVILEGES ON WSO2SHARED_DB.* TO 'wso2_prd_user'@'10.133.4.188';

   FLUSH PRIVILEGES;

4.2 Inicialización de Esquemas
-------------------------------

**Ejecutar en el Nodo 1** usando los scripts SQL incluidos con WSO2:

.. code-block:: bash

   # Base de datos compartida
   mysql -h 10.132.0.56 -u wso2_prd_user -p WSO2SHARED_DB \
     < /opt/wso2/wso2am-4.7.0/dbscripts/mysql.sql

   # Base de datos APIM
   mysql -h 10.132.0.56 -u wso2_prd_user -p WSO2AM_DB \
     < /opt/wso2/wso2am-4.7.0/dbscripts/apimgt/mysql.sql

4.3 Colocación del Driver JDBC
-------------------------------

Descargar ``mariadb-java-client-3.x.x.jar`` y colocarlo en **ambos nodos**:

.. code-block:: bash

   cp mariadb-java-client-3.x.x.jar \
     /opt/wso2/wso2am-4.7.0/repository/components/lib/
   chown wso2:wso2 \
     /opt/wso2/wso2am-4.7.0/repository/components/lib/mariadb-java-client-3.x.x.jar

================================================================================
5. Configuración de Keystore y Truststore
================================================================================

**Punto clave de un clúster activo-activo: ambos nodos deben usar exactamente
el mismo keystore y truststore** [citation:3][citation:9].

5.1 Generación en el Nodo 1
----------------------------

.. code-block:: bash

   cd /opt/wso2/wso2am-4.7.0/repository/resources/security/

   # Respaldar los originales
   cp wso2carbon.jks wso2carbon.jks.bak
   cp client-truststore.jks client-truststore.jks.bak

   # Convertir el certificado de dominio a formato PKCS12
   openssl pkcs12 -export \
     -in /ruta/api.tu-dominio.com.crt \
     -inkey /ruta/api.tu-dominio.com.key \
     -name wso2carbon \
     -out wso2carbon.p12 \
     -passout pass:wso2carbon

   # Importar a JKS
   keytool -importkeystore \
     -srckeystore wso2carbon.p12 -srcstoretype PKCS12 -srcstorepass wso2carbon \
     -destkeystore wso2carbon.jks -deststoretype JKS -deststorepass wso2carbon

   # Importar el certificado al truststore
   keytool -import -trustcacerts -alias wso2carbon-prd \
     -file /ruta/api.tu-dominio.com.crt \
     -keystore client-truststore.jks -storepass wso2carbon -noprompt

   chown wso2:wso2 wso2carbon.jks wso2carbon.p12 client-truststore.jks

5.2 Replicación hacia el Nodo 2
--------------------------------

.. code-block:: bash

   # Ejecutar en el Nodo 1
   rsync -avz /opt/wso2/wso2am-4.7.0/repository/resources/security/ \
     root@10.133.4.188:/opt/wso2/wso2am-4.7.0/repository/resources/security/

   # En el Nodo 2 corregir propietario
   ssh root@10.133.4.188 \
     "chown -R wso2:wso2 /opt/wso2/wso2am-4.7.0/repository/resources/security/"

================================================================================
6. Montaje del Sistema de Archivos Compartido NFS
================================================================================

**Este es el mecanismo principal recomendado oficialmente para la sincronización
de artefactos en despliegues activo-activo All-in-One** [citation:3][citation:9].

**Realizar en ambos nodos:**

.. code-block:: bash

   sudo dnf install -y nfs-utils
   sudo setsebool -P virt_use_nfs 1   # Si SELinux está habilitado

   # Agregar a /etc/fstab (reemplazar <IP_NFS> y <RUTA_EXPORT>)
   echo "<IP_NFS>:<RUTA_EXPORT> \
   /opt/wso2/wso2am-4.7.0/repository/deployment/server \
   nfs4 defaults,_netdev,noatime 0 0" | sudo tee -a /etc/fstab

   sudo systemctl daemon-reload
   sudo mount -a

**Si el directorio NFS está vacío**, copiar el contenido inicial desde el Nodo 1:

.. code-block:: bash

   sudo cp -rp /opt/wso2/wso2am-4.7.0/repository/deployment/server/* \
     /punto/de/montaje/nfs/
   sudo chown -R wso2:wso2 /opt/wso2/wso2am-4.7.0/repository/deployment/server

================================================================================
7. Configuración Completa de deployment.toml
================================================================================

**Editar en ambos nodos** el archivo
``/opt/wso2/wso2am-4.7.0/repository/conf/deployment.toml``.

Ajustar ``node_ip`` según el nodo (``10.133.4.187`` en Nodo 1,
``10.133.4.188`` en Nodo 2) y asegurar que la clave en ``[encryption]`` sea
**idéntica en ambos servidores** [citation:9].

.. code-block:: toml

   [server]
   hostname = "api.tu-dominio.com"      # Dominio VIP de F5
   node_ip = "10.133.4.187"             # Nodo 2 cambiar a 10.133.4.188

   [transport.https.properties]
   proxyPort = 443                       # Evita que la URL generada incluya :9443

   [super_admin]
   username = "admin"
   password = "TU_PASSWORD_ADMIN_PRD"
   create_admin_account = true

   # Clave de cifrado: ambos nodos deben ser idénticos, 64 caracteres hex
   [encryption]
   key = "GENERAR_UNA_CLAVE_HEX_DE_64_CARACTERES_Y_USARLA_EN_AMBOS_NODOS"

   # Base de datos APIM — Ajustar URL según política SSL del DBA
   [database.apim_db]
   type = "mariadb"
   url = "jdbc:mariadb://10.132.0.56:3306/WSO2AM_DB?useSSL=false&allowPublicKeyRetrieval=true"
   username = "wso2_prd_user"
   password = "TU_PASSWORD_SEGURO_BD"

   [database.apim_db.pool_options]
   maxActive = 80
   maxWait = 60000
   testOnBorrow = true
   validationQuery = "SELECT 1"

   # Base de datos compartida
   [database.shared_db]
   type = "mariadb"
   url = "jdbc:mariadb://10.132.0.56:3306/WSO2SHARED_DB?useSSL=false&allowPublicKeyRetrieval=true"
   username = "wso2_prd_user"
   password = "TU_PASSWORD_SEGURO_BD"

   [database.shared_db.pool_options]
   maxActive = 80
   maxWait = 60000
   testOnBorrow = true
   validationQuery = "SELECT 1"

   # Keystore y Truststore
   [keystore.tls]
   file_name = "wso2carbon.jks"
   type = "JKS"
   password = "wso2carbon"
   alias = "wso2carbon"
   key_password = "wso2carbon"

   [truststore]
   file_name = "client-truststore.jks"
   type = "JKS"
   password = "wso2carbon"

   # --- Entorno Gateway (Publisher → Gateways del clúster) ---
   [[apim.gateway.environment]]
   name = "Production and Sandbox"
   type = "hybrid"
   display_in_api_console = true
   description = "Gateway Cluster Produccion 4.7.0"
   show_as_token_endpoint_url = true
   service_url = "https://localhost:${mgt.transport.https.port}/services/"
   username = "${admin.username}"
   password = "${admin.password}"
   ws_endpoint = "ws://api.tu-dominio.com:9099"
   wss_endpoint = "wss://api.tu-dominio.com:8099"
   http_endpoint = "http://api.tu-dominio.com:80"
   https_endpoint = "https://api.tu-dominio.com:443"

   # --- Throttling Distribuido (Traffic Manager en ambos nodos) ---
   [apim.throttling]
   enable_unlimited_tier = true
   enable_header_based_throttling = false
   enable_jwt_claim_based_throttling = false
   enable_query_param_based_throttling = false

   [[apim.throttling.url_group]]
   traffic_manager_urls = ["tcp://10.133.4.187:9611", "tcp://10.133.4.188:9611"]
   traffic_manager_auth_urls = ["ssl://10.133.4.187:9711", "ssl://10.133.4.188:9711"]
   type = "loadbalance"

.. note::

   **Sobre clustering Hazelcast**: La documentación oficial de despliegue
   activo-activo All-in-One de WSO2 se basa principalmente en **base de datos
   compartida + sistema de archivos compartido (NFS)**. El clúster de Hazelcast
   (``[clustering]``) **no es un requisito obligatorio** para este escenario.
   La configuración de ``[apim.throttling]`` con ``url_group`` sí es necesaria
   para que el Gateway consulte al Traffic Manager local de cada nodo [citation:14][citation:22].

================================================================================
8. Configuración del Servicio systemd (ÚLTIMO PASO ANTES DE ARRANCAR)
================================================================================

.. important::

   **Crear el servicio systemd solamente después de haber completado las
   secciones 1 (límites del SO), 6 (NFS) y 7 (deployment.toml).** De esta
   manera los límites de ``nofile`` y ``nproc`` estarán vigentes cuando
   systemd lance el proceso Java.

**Realizar en ambos nodos:**

.. code-block:: bash

   sudo tee /etc/systemd/system/wso2apim.service > /dev/null << 'EOF'
   [Unit]
   Description=WSO2 API Manager 4.7.0
   After=network-online.target remote-fs.target
   Wants=network-online.target

   [Service]
   Type=forking
   User=wso2
   Group=wso2
   Environment="JAVA_HOME=/usr/lib/jvm/temurin-21-jdk"
   Environment="JVM_MEM_OPTS=-Xms4g -Xmx4g"
   ExecStart=/opt/wso2/wso2am-4.7.0/bin/api-manager.sh start
   ExecStop=/opt/wso2/wso2am-4.7.0/bin/api-manager.sh stop
   PIDFile=/opt/wso2/wso2am-4.7.0/wso2carbon.pid
   LimitNOFILE=65535
   Restart=on-failure
   RestartSec=15

   [Install]
   WantedBy=multi-user.target
   EOF

   sudo systemctl daemon-reload
   sudo systemctl enable wso2apim

**Dimensionamiento JVM**: ``-Xms4g -Xmx4g`` es un punto de partida razonable
para un nodo All-in-One con carga moderada. Ajustar según la memoria RAM
disponible y el perfil de carga real. Mantener ``Xms=Xmx`` para evitar pausas
por redimensionamiento del heap.

================================================================================
9. Firewall (Reglas Simétricas)
================================================================================

**Las reglas deben ser simétricas**: cada nodo debe permitir el tráfico
proveniente del otro nodo y de la subred del F5.

9.1 Puertos requeridos entre nodos (10.133.4.187 ↔ 10.133.4.188)
-----------------------------------------------------------------

.. list-table:: Puertos de Comunicación Interna del Clúster
   :header-rows: 1
   :widths: 20 30 50

   * - Puerto
     - Protocolo
     - Propósito
   * - 4000
     - TCP
     - Hazelcast / comunicación de clúster
   * - 9611
     - TCP
     - Traffic Manager (binary)
   * - 9711
     - TCP
     - Traffic Manager (SSL)
   * - 5672
     - TCP
     - Broker / mensajería interna

9.2 Puertos desde la subred del F5
-----------------------------------

.. list-table:: Puertos de Acceso desde F5
   :header-rows: 1
   :widths: 20 30 50

   * - Puerto
     - Protocolo
     - Propósito
   * - 9443
     - TCP
     - Portales (Publisher, DevPortal, Admin)
   * - 8243
     - TCP
     - Gateway HTTPS (Pass-through)

9.3 Ejemplo de aplicación (Nodo 1)
------------------------------------

.. code-block:: bash

   # Permitir tráfico desde el Nodo 2
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="4000" protocol="tcp" accept'
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="9611-9711" protocol="tcp" accept'
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="5672" protocol="tcp" accept'

   # Permitir tráfico desde subred del F5
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<SUBRED_F5>" port port="9443" protocol="tcp" accept'
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<SUBRED_F5>" port port="8243" protocol="tcp" accept'

   sudo firewall-cmd --reload

**Repetir en el Nodo 2 invirtiendo la IP de origen** (``10.133.4.187`` en
lugar de ``10.133.4.188`` para las reglas entre nodos).

================================================================================
10. Inicio y Verificación
================================================================================

10.1 Inicio Secuencial
-----------------------

**Iniciar primero el Nodo 1:**

.. code-block:: bash

   sudo systemctl start wso2apim
   sudo tail -f /opt/wso2/wso2am-4.7.0/repository/logs/wso2carbon.log

Esperar a ver el mensaje ``WSO2 Carbon started in X sec`` y confirmar que no
hay errores de conexión a la base de datos ni excepciones de enlace de puertos.

**Luego iniciar el Nodo 2:**

.. code-block:: bash

   sudo systemctl start wso2apim
   sudo tail -f /opt/wso2/wso2am-4.7.0/repository/logs/wso2carbon.log

10.2 Matriz de Pruebas de Aceptación
-------------------------------------

.. list-table:: Acta de Validación Técnica
   :header-rows: 1
   :widths: 10 25 40 25

   * - ID
     - Caso de Prueba
     - Procedimiento
     - Resultado Esperado
   * - **TC-01**
     - Acceso a Portales vía F5
     - Ingresar a ``https://api.tu-dominio.com/publisher`` y
       ``https://api.tu-dominio.com/devportal``.
     - Login exitoso sin ``:9443`` en la URL.
   * - **TC-02**
     - Publicación y Sincronización de API
     - Crear y publicar una API de prueba en Publisher; verificar
       sincronización entre nodos vía NFS y persistencia en MariaDB.
     - API visible en DevPortal y disponible en ambos nodos.
   * - **TC-03**
     - Balanceo Activo-Activo del Gateway
     - Generar token y ejecutar 20 peticiones ``curl -k`` al endpoint del
       Gateway; inspeccionar logs de acceso en ambos nodos.
     - Ambas instancias atienden peticiones con HTTP ``200``.
   * - **TC-04**
     - Failover ante Caída del Nodo 1
     - Detener ``wso2apim`` en el Nodo 1 mientras se envían peticiones
       continuas al Gateway.
     - F5 redirige el 100% del tráfico al Nodo 2 sin pérdida.
   * - **TC-05**
     - Reincorporación del Nodo 1
     - Iniciar ``wso2apim`` en el Nodo 1 y verificar ``wso2carbon.log``.
     - Nodo 1 se reincorpora y F5 vuelve a distribuir tráfico.

================================================================================
11. Plan de Rollback (Contingencia)
================================================================================

**Si se detecta una incidencia crítica durante la ventana de implementación,
ejecutar en este orden:**

1. **Detener servicios en ambos nodos:**

   .. code-block:: bash

      sudo systemctl stop wso2apim

2. **Restaurar respaldos de configuración y keystores** (generados antes de
   la ventana, según Fase 0 del manual original):

   .. code-block:: bash

      # En cada nodo, restaurar desde el tar de respaldo
      sudo tar -xzvf /root/backups_wso2/backup_wso2apim470_prd_$(hostname)_*.tar.gz \
        -C /

3. **Revertir cambios en la base de datos** (solo si es necesario y
   coordinado con el DBA):

   - Si los esquemas fueron creados en esta ventana y no contienen datos:
     ``DROP DATABASE WSO2AM_DB; DROP DATABASE WSO2SHARED_DB;``
   - Si contienen datos: restaurar desde el respaldo lógico del DBA.

4. **Desmontar NFS** (si se montó en esta ventana):

   .. code-block:: bash

      sudo umount /opt/wso2/wso2am-4.7.0/repository/deployment/server
      # Eliminar la línea correspondiente de /etc/fstab

5. **Coordinar con el equipo de F5** la deshabilitación o reversión del pool
   de producción si el servicio no queda operativo.

6. **Reiniciar el servicio anterior validado** (si existe una versión previa)
   y confirmar disponibilidad de portales y Gateway.

================================================================================
12. Resumen de Riesgos y Recomendaciones
================================================================================

.. list-table:: Matriz de Compatibilidad y Riesgos
   :header-rows: 1
   :widths: 30 20 50

   * - Elemento
     - Estado
     - Observación
   * - JDK 21 (Temurin)
     - Compatible
     - Probado oficialmente para 4.7.0.
   * - Rocky Linux 9
     - Compatible
     - Básicamente compatible con RHEL 9.6 probado oficialmente.
   * - MariaDB
     - **Riesgo**
     - **No está en la lista de pruebas oficiales** [citation:21]. Validar SSL, codificación
       y driver con el DBA antes de producción.
   * - NFS compartido
     - Recomendado
     - Método de sincronización recomendado oficialmente; depende de la
       disponibilidad del NFS.
   * - Clustering Hazelcast
     - Opcional
     - No habilitado por defecto en esta guía; el estándar activo-activo
       oficial se basa en BD + NFS compartidos.
   * - Configuración de sistema (limits/sysctl)
     - **Obligatorio antes del servicio**
     - Si se omite, el proceso Java no heredará los límites correctos.
   * - Reglas de firewall simétricas
     - **Obligatorio**
     - Verificar que cada nodo permita el tráfico del otro y de la subred F5.

.. note::

   **Vía para certificación oficial**: Tener una suscripción válida de WSO2 y
   enviar la configuración del entorno (especialmente la elección de MariaDB,
   la política SSL de la cadena JDBC, y la arquitectura NFS) al soporte de
   WSO2 para revisión; o encargar a un socio certificado de WSO2 una auditoría
   de implementación. Los pasos anteriores pueden servir como base para dicha
   auditoría.
