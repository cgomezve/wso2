===========================================================================================
Plan de Trabajo y Guia de Implementacion: Cluster WSO2 API Manager 4.7.0 en Rocky Linux 9
===========================================================================================

:Entorno: Produccion (PRD)
:Sistema Operativo: Rocky Linux 9.x
:Producto: WSO2 API Manager 4.7.0 (Despliegue All-in-One Activo-Activo)
:Java Runtime: JDK 21 (o JDK 25 segun estandar aprobado)
:Nodos:
   - ``lrkprdappdckposapi01`` (``10.133.4.187``)
   - ``lrkprdappdckposapi02`` (``10.133.4.188``)

.. contents:: Tabla de Contenidos
   :depth: 3
   :local:

=================================================================
Parte 1: Plan de Trabajo Detallado (Ventana de Implementacion)
=================================================================

1.1 Objetivo y Arquitectura de Referencia
-----------------------------------------

Implementar una arquitectura de alta disponibilidad (HA) activo-activo para **WSO2 API Manager 4.7.0** sobre **Rocky Linux 9**, utilizando **JDK 21**, bases de datos relacionales compartidas (``WSO2AM_DB`` y ``WSO2SHARED_DB``), clave de cifrado interna compartida (``[encryption]``), sincronizacion de artefactos y reverse proxy Nginx con terminacion TLS.

.. code-block:: text

                        [ Clientes / Consumidores API ]
                                       |
                            HTTPS (443) / HTTP (80)
                                       v
                     +-----------------------------------+
                     |              F5                   |
                     |       (api.tu-dominio.com)        |
                     +-----------------------------------+
                            /                     \
             Portales (:9443) / Gateway (:8243)    \
                          /                         \
                         v                           v
      +---------------------------------+   +---------------------------------+
      |      lrkprdappdckposapi01       |   |      lrkprdappdckposapi02       |
      |          10.133.4.187           |   |          10.133.4.188           |
      |  WSO2 APIM 4.7.0 (All-in-One)   |   |  WSO2 APIM 4.7.0 (All-in-One)   |
      +---------------------------------+   +---------------------------------+
               |               |                     |               |
               |               +----------+----------+               |
               |                          |                          |
               v                          v                          v
      +-----------------+      +---------------------+      +-----------------+
      |   NFS / Git     |      |    Base de Datos    |      |   NFS / Git     |
      | (Artefactos)    |      | WSO2AM_DB           |      | (Artefactos)    |
      |                 |      | WSO2SHARED_DB       |      |                 |
      +-----------------+      +---------------------+      +-----------------+

1.2 Matriz de Requisitos Previos (Pre-Check)
--------------------------------------------

.. list-table:: Verificacion de Prerrequisitos antes de Ventana (APIM 4.7.0)
   :header-rows: 1
   :widths: 20 50 30

   * - Componente
     - Requisito Tecnico
     - Criterio de Aceptacion
   * - **Sistema Operativo**
     - Rocky Linux 9.x actualizado via ``dnf``, ``ulimit -n`` >= 65535, ``vm.max_map_count`` >= 262144 y sincronizacion NTP (``chronyd``) activa en ambos nodos.
     - Diferencia horaria < 10 ms entre ``10.133.4.187`` y ``10.133.4.188``.
   * - **Java Runtime**
     - **JDK 21** (o JDK 25) instalado via ``dnf`` (``java-21-openjdk-devel``) con ``JAVA_HOME`` configurado en ambos nodos.
     - ``java -version`` reporta version 21.x en ambos servidores.
   * - **Paquete WSO2 APIM 4.7.0**
     - Distribucion ``wso2am-4.7.0.zip`` descomprimida en ``/opt/wso2/wso2am-4.7.0`` (o actualizacion controlada) con usuario ``wso2``.
     - Estructura ``bin/``, ``repository/`` y ``dbscripts/`` presente en ambos nodos.
   * - **Firewall (firewalld)**
     - Puertos habilitados entre nodos: ``4000/tcp`` (Hazelcast WKA), ``9611/tcp``, ``9711/tcp`` (Traffic Manager), ``5672/tcp`` (Broker), ``9443/tcp`` y ``8243/tcp``.
     - Pruebas ``nc -zv`` exitosas entre ambos nodos.
   * - **Base de Datos**
     - Instancia PostgreSQL / MariaDB / MySQL / Oracle / MSSQL en PRD con esquemas ``WSO2AM_DB`` y ``WSO2SHARED_DB`` creados en UTF-8.
     - Conexion JDBC validada desde ambos nodos.
   * - **Almacenamiento**
     - Recurso NFS compartido montado en ``/opt/wso2/wso2am-4.7.0/repository/deployment/server`` (o sincronizacion Git) con permisos para usuario ``wso2``.
     - Archivo creado en Nodo 1 visible inmediatamente en Nodo 2.
   * - **Certificados y DNS**
     - Registro DNS ``api.tu-dominio.com`` apuntando al Nginx Reverse Proxy y certificado TLS vigente (``.crt`` y ``.key``).
     - Resolucion DNS correcta y certificado valido sin alertas de cadena.

1.3 Cronograma y Plan de Ejecucion
----------------------------------

.. list-table:: Plan de Trabajo Paso a Paso (WSO2 APIM 4.7.0)
   :header-rows: 1
   :widths: 10 25 40 15 10

   * - Fase
     - Actividad
     - Entregable / Accion Clave
     - Duracion Est.
     - Rollback
   * - **0**
     - Preparacion y Respaldo
     - Detencion de instancias previas, backup, instalacion de JDK 21 y despliegue de APIM 4.7.0 en Rocky Linux 9.
     - 25 min
     - N/A
   * - **1**
     - Bases de Datos Compartidas
     - Creacion de ``WSO2AM_DB`` y ``WSO2SHARED_DB``, ejecucion de scripts SQL de 4.7.0 y despliegue de driver JDBC.
     - 25 min
     - Drop BDs
   * - **2**
     - Seguridad TLS y Keystores
     - Configuracion de ``wso2carbon.jks`` y ``client-truststore.jks`` homologados entre Nodo 1 y Nodo 2.
     - 20 min
     - Restaurar backup
   * - **3**
     - Sincronizacion de Artefactos
     - Montaje NFS en ``repository/deployment/server`` con ``_netdev`` en ``/etc/fstab`` y permisos ``wso2``.
     - 15 min
     - Desmontar NFS
   * - **4**
     - Configuracion ``deployment.toml``
     - Parametrizacion de BDs, clave ``[encryption]``, clustering WKA, Traffic Manager y ``proxyPort = 443`` en ambos nodos.
     - 25 min
     - Restaurar ``deployment.toml``
   * - **5**
     - Servicio ``systemd`` y Nginx
     - Creacion de unidad ``wso2apim.service`` con ``JAVA_HOME`` JDK 21 y reverse proxy Nginx con SSL y health checks.
     - 20 min
     - Revertir conf Nginx
   * - **6**
     - Arranque y Caso de Exito
     - Inicio secuencial Nodo 1 -> Nodo 2, verificacion de logs, prueba de APIs y prueba de failover controlado.
     - 30 min
     - Restaurar snapshot/tar

1.4 Plan de Rollback (Contingencia)
-----------------------------------

En caso de incidencia critica durante la ventana:

1. Detener ``wso2apim.service`` en ``lrkprdappdckposapi01`` y ``lrkprdappdckposapi02``.
2. Restaurar los respaldos comprimidos generados en la **Fase 0** (``/root/backups_wso2/backup_wso2apim470_prd_*.tar.gz``).
3. Revertir la configuracion de Nginx al estado previo y recargar (``nginx -t && systemctl reload nginx``).
4. Iniciar el servicio anterior validado y comprobar disponibilidad de portales y Gateway.

=================================================================
Parte 2: Procedimiento Tecnico Detallado en Rocky Linux 9
=================================================================

.. note::
   En esta guia se asume como ruta de instalacion ``/opt/wso2/wso2am-4.7.0`` y usuario de sistema ``wso2``. Ajusta las rutas si tu estandar corporativo define otro directorio.

Fase 0: Preparacion del Sistema Operativo (Rocky Linux 9), JDK 21 y Respaldo
----------------------------------------------------------------------------

Ejecuta los siguientes pasos en **ambos nodos** (``10.133.4.187`` y ``10.133.4.188``):

1. **Detener procesos existentes y respaldar configuracion/instalacion previa:**

   .. code-block:: bash

      systemctl stop wso2apim 2>/dev/null || true
      mkdir -p /root/backups_wso2
      if [ -d /opt/wso2/wso2am-4.7.0 ]; then
        tar -czvf /root/backups_wso2/backup_wso2apim470_prd_$(hostname)_$(date +%Y%m%d_%H%M).tar.gz \
          /opt/wso2/wso2am-4.7.0/repository/conf \
          /opt/wso2/wso2am-4.7.0/repository/resources/security \
          /opt/wso2/wso2am-4.7.0/repository/deployment/server
      fi

2. **Instalar JDK 21 y utilidades base en Rocky Linux 9 via ``dnf``:**

   .. code-block:: bash

      dnf install -y java-21-openjdk java-21-openjdk-devel unzip rsync nc
      java -version

3. **Crear usuario dedicado del sistema y descomprimir WSO2 API Manager 4.7.0:**

   .. code-block:: bash

      groupadd -r wso2 2>/dev/null || true
      useradd -r -g wso2 -d /opt/wso2 -s /sbin/nologin wso2 2>/dev/null || true
      mkdir -p /opt/wso2
      unzip -q /tmp/wso2am-4.7.0.zip -d /opt/wso2/
      chown -R wso2:wso2 /opt/wso2/wso2am-4.7.0

4. **Ajustar limites de descriptores de archivos y parametros del kernel:**

   .. code-block:: bash

      cat << 'EOF' > /etc/security/limits.d/99-wso2.conf
      wso2 soft nofile 65535
      wso2 hard nofile 65535
      wso2 soft nproc 32768
      wso2 hard nproc 32768
      EOF

      cat << 'EOF' > /etc/sysctl.d/99-wso2.conf
      vm.max_map_count = 262144
      net.core.somaxconn = 4096
      EOF
      sysctl --system

5. **Configurar ``/etc/hosts`` y reglas de ``firewalld`` en Rocky Linux 9:**

   .. code-block:: bash

      cat << 'EOF' >> /etc/hosts
      10.133.4.187   lrkprdappdckposapi01
      10.133.4.188   lrkprdappdckposapi02
      <IP_NGINX_PROXY>   api.tu-dominio.com
      EOF

      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.187" port port="4000" protocol="tcp" accept'
      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="4000" protocol="tcp" accept'
      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.187" port port="9611-9711" protocol="tcp" accept'
      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="9611-9711" protocol="tcp" accept'
      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.187" port port="5672" protocol="tcp" accept'
      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="5672" protocol="tcp" accept'
      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<IP_NGINX_PROXY>" port port="9443" protocol="tcp" accept'
      firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<IP_NGINX_PROXY>" port port="8243" protocol="tcp" accept'
      firewall-cmd --reload

6. **Verificar sincronizacion horaria con Chrony:**

   .. code-block:: bash

      systemctl enable --now chronyd
      chronyc tracking

Fase 1: Preparar Bases de Datos Compartidas y Driver JDBC
--------------------------------------------------------

1. **Crear bases de datos y usuario en el motor RDBMS (ejemplo PostgreSQL):**

   .. code-block:: sql

      CREATE USER wso2_prd_user WITH PASSWORD 'TU_PASSWORD_SEGURO_BD';
      CREATE DATABASE "WSO2SHARED_DB" OWNER wso2_prd_user ENCODING 'UTF8';
      CREATE DATABASE "WSO2AM_DB" OWNER wso2_prd_user ENCODING 'UTF8';
      GRANT ALL PRIVILEGES ON DATABASE "WSO2SHARED_DB" TO wso2_prd_user;
      GRANT ALL PRIVILEGES ON DATABASE "WSO2AM_DB" TO wso2_prd_user;

2. **Inicializar los esquemas SQL de WSO2 APIM 4.7.0 desde el Nodo 1:**

   .. code-block:: bash

      export PGPASSWORD='TU_PASSWORD_SEGURO_BD'
      psql -h <IP_SERVIDOR_BD> -U wso2_prd_user -d WSO2SHARED_DB \
        -f /opt/wso2/wso2am-4.7.0/dbscripts/postgresql.sql

      psql -h <IP_SERVIDOR_BD> -U wso2_prd_user -d WSO2AM_DB \
        -f /opt/wso2/wso2am-4.7.0/dbscripts/apimgt/postgresql.sql

3. **Copiar el driver JDBC en ambos nodos:**

   .. code-block:: bash

      cp /tmp/postgresql-42.7.3.jar /opt/wso2/wso2am-4.7.0/repository/components/lib/
      chown wso2:wso2 /opt/wso2/wso2am-4.7.0/repository/components/lib/postgresql-42.7.3.jar

Fase 2: Homologar Keystores y Truststores TLS
---------------------------------------------

Ambos nodos deben utilizar exactamente los mismos almacenes de llaves para firmar y validar tokens JWT internos y conexiones TLS.

1. **En el Nodo 1 (``lrkprdappdckposapi01``), generar o importar el certificado del dominio:**

   .. code-block:: bash

      cd /opt/wso2/wso2am-4.7.0/repository/resources/security

      cp wso2carbon.jks wso2carbon.jks.orig
      cp client-truststore.jks client-truststore.jks.orig

      openssl pkcs12 -export \
        -in /etc/pki/tls/certs/api.tu-dominio.com.crt \
        -inkey /etc/pki/tls/private/api.tu-dominio.com.key \
        -name wso2carbon \
        -out wso2carbon.p12 \
        -passout pass:wso2carbon

      rm -f wso2carbon.jks
      keytool -importkeystore \
        -srckeystore wso2carbon.p12 -srcstoretype PKCS12 -srcstorepass wso2carbon \
        -destkeystore wso2carbon.jks -deststoretype JKS -deststorepass wso2carbon

      keytool -import -trustcacerts -alias wso2carbon-prd \
        -file /etc/pki/tls/certs/api.tu-dominio.com.crt \
        -keystore client-truststore.jks -storepass wso2carbon -noprompt

      chown wso2:wso2 wso2carbon.jks wso2carbon.p12 client-truststore.jks

2. **Replicar exactamente los almacenes hacia el Nodo 2 (``lrkprdappdckposapi02``):**

   .. code-block:: bash

      rsync -avz /opt/wso2/wso2am-4.7.0/repository/resources/security/ \
        root@10.133.4.188:/opt/wso2/wso2am-4.7.0/repository/resources/security/

      ssh root@10.133.4.188 "chown -R wso2:wso2 /opt/wso2/wso2am-4.7.0/repository/resources/security/"

Fase 3: Configurar Almacenamiento Compartido para Artefactos (NFS)
------------------------------------------------------------------

Para que los artefactos desplegados en un nodo esten disponibles en el otro sin desfasajes:

1. **Instalar utilidades NFS y habilitar booleano SELinux en Rocky Linux 9 (ambos nodos):**

   .. code-block:: bash

      dnf install -y nfs-utils
      setsebool -P virt_use_nfs 1

2. **Respaldar contenido inicial de ``deployment/server`` y montar recurso NFS en ambos nodos:**

   .. code-block:: bash

      cp -rp /opt/wso2/wso2am-4.7.0/repository/deployment/server /tmp/server_seed_470

      echo "<IP_SERVIDOR_NFS>:/exports/wso2apim470_prd_server  /opt/wso2/wso2am-4.7.0/repository/deployment/server  nfs4  defaults,_netdev,noatime  0 0" >> /etc/fstab
      systemctl daemon-reload
      mount -a

      cp -rn /tmp/server_seed_470/* /opt/wso2/wso2am-4.7.0/repository/deployment/server/
      chown -R wso2:wso2 /opt/wso2/wso2am-4.7.0/repository/deployment/server

Fase 4: Configurar ``deployment.toml`` en Ambos Nodos
----------------------------------------------------

Edita ``/opt/wso2/wso2am-4.7.0/repository/conf/deployment.toml`` en ambos nodos. Ajusta ``node_ip`` y ``local_member_host`` segun corresponda (``10.133.4.187`` en Nodo 1 y ``10.133.4.188`` en Nodo 2) y asegura que la clave en ``[encryption]`` sea identica en ambos servidores.

.. code-block:: toml

   [server]
   hostname = "api.tu-dominio.com"
   node_ip = "10.133.4.187" # Cambiar a "10.133.4.188" en lrkprdappdckposapi02
   mode = "single"

   [super_admin]
   username = "admin"
   password = "TU_PASSWORD_ADMIN_PRD"
   create_admin_account = true

   # --- Clave interna de cifrado compartida entre nodos ---
   [encryption]
   key = "REEMPLAZAR_POR_CLAVE_SEGURA_COMPARTIDA_IDENTICA_EN_AMBOS_NODOS"

   # --- Base de Datos Compartida (Registry y Usuarios) ---
   [database.shared_db]
   type = "postgresql"
   url = "jdbc:postgresql://<IP_SERVIDOR_BD>:5432/WSO2SHARED_DB"
   username = "wso2_prd_user"
   password = "TU_PASSWORD_SEGURO_BD"

   [database.shared_db.pool_options]
   maxActive = 80
   maxWait = 60000
   testOnBorrow = true
   validationQuery = "SELECT 1"

   # --- Base de Datos API Manager ---
   [database.apim_db]
   type = "postgresql"
   url = "jdbc:postgresql://<IP_SERVIDOR_BD>:5432/WSO2AM_DB"
   username = "wso2_prd_user"
   password = "TU_PASSWORD_SEGURO_BD"

   [database.apim_db.pool_options]
   maxActive = 80
   maxWait = 60000
   testOnBorrow = true
   validationQuery = "SELECT 1"

   # --- Keystore y Truststore ---
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

   # --- Puertos Proxy para Reverse Proxy Nginx (Terminacion TLS en 443) ---
   [transport.https.properties]
   proxyPort = 443

   [transport.passthru_https.listener.parameters]
   WSDLEPRPrefix = "https://api.tu-dominio.com"
   BindAddress = "0.0.0.0"

   # --- Clustering WKA Hazelcast ---
   [clustering]
   membership_scheme = "wka"
   domain = "wso2.apim470.prd.domain"
   local_member_host = "10.133.4.187" # Cambiar a "10.133.4.188" en lrkprdappdckposapi02
   local_member_port = "4000"

   [[clustering.members]]
   host = "10.133.4.187"
   port = 4000

   [[clustering.members]]
   host = "10.133.4.188"
   port = 4000

   # --- Entorno Gateway Publico ---
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

   # --- Throttling Compartido ---
   [apim.throttling]
   enable_unlimited_tier = true
   enable_header_based_throttling = false
   enable_jwt_claim_based_throttling = false
   enable_query_param_based_throttling = false

   [[apim.throttling.url_group]]
   traffic_manager_urls = ["tcp://10.133.4.187:9611", "tcp://10.133.4.188:9611"]
   traffic_manager_auth_urls = ["ssl://10.133.4.187:9711", "ssl://10.133.4.188:9711"]
   type = "loadbalance"

Fase 5: Configurar Servicio ``systemd`` y Reverse Proxy Nginx
------------------------------------------------------------

1. **Crear unidad ``systemd`` con JDK 21 en ambos nodos Rocky Linux 9:**

   .. code-block:: bash

      cat << 'EOF' > /etc/systemd/system/wso2apim.service
      [Unit]
      Description=WSO2 API Manager 4.7.0
      After=network-online.target remote-fs.target
      Wants=network-online.target

      [Service]
      Type=forking
      User=wso2
      Group=wso2
      Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk"
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

      systemctl daemon-reload
      systemctl enable wso2apim

2. **Configurar Nginx Reverse Proxy (en el servidor balanceador/proxy):**

   .. code-block:: nginx

      upstream wso2_portals_9443 {
          ip_hash;
          server 10.133.4.187:9443 max_fails=3 fail_timeout=30s;
          server 10.133.4.188:9443 max_fails=3 fail_timeout=30s;
      }

      upstream wso2_gateway_8243 {
          least_conn;
          server 10.133.4.187:8243 max_fails=3 fail_timeout=15s;
          server 10.133.4.188:8243 max_fails=3 fail_timeout=15s;
      }

      server {
          listen 443 ssl http2;
          server_name api.tu-dominio.com;

          ssl_certificate     /etc/pki/tls/certs/api.tu-dominio.com.crt;
          ssl_certificate_key /etc/pki/tls/private/api.tu-dominio.com.key;
          ssl_protocols       TLSv1.2 TLSv1.3;

          proxy_set_header Host              $host;
          proxy_set_header X-Real-IP         $remote_addr;
          proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto https;

          # Portales de Administracion, Publisher, DevPortal y OAuth2
          location ~ ^/(publisher|devportal|admin|carbon|oauth2|oidc|api/am|services/Version) {
              proxy_pass https://wso2_portals_9443;
              proxy_ssl_verify off;
              proxy_read_timeout 180s;
          }

          # Trafico de APIs en Runtime (Gateway)
          location / {
              proxy_pass https://wso2_gateway_8243;
              proxy_ssl_verify off;
              proxy_read_timeout 120s;
          }
      }

   Validar y recargar Nginx:

   .. code-block:: bash

      setsebool -P httpd_can_network_connect 1
      nginx -t && systemctl reload nginx

=================================================================
Parte 3: Caso de Exito — Arranque y Validacion de Aceptacion
=================================================================

Para dar por exitoso el despliegue de **WSO2 API Manager 4.7.0** en Produccion, ejecuta y registra la evidencia de cada prueba:

3.1 Arranque Secuencial y Verificacion de Logs
----------------------------------------------

1. **Iniciar Nodo 1 (``lrkprdappdckposapi01``):**

   .. code-block:: bash

      systemctl start wso2apim
      tail -f /opt/wso2/wso2am-4.7.0/repository/logs/wso2carbon.log

   * **Evidencia esperada:** Mensaje ``WSO2 Carbon started in X sec`` sobre JDK 21 sin errores ``SQLException`` ni ``BindException``.

2. **Iniciar Nodo 2 (``lrkprdappdckposapi02``):**

   .. code-block:: bash

      systemctl start wso2apim
      tail -f /opt/wso2/wso2am-4.7.0/repository/logs/wso2carbon.log

   * **Evidencia esperada:** En ambos logs aparece el registro de membresia del cluster con ambos nodos activos:

     .. code-block:: text

        Members [2] {
            Member [10.133.4.187]:4000
            Member [10.133.4.188]:4000
        }

3.2 Matriz de Pruebas Funcionales y Failover (Caso de Exito)
------------------------------------------------------------

.. list-table:: Acta de Validacion Tecnica (Caso de Exito PRD — APIM 4.7.0)
   :header-rows: 1
   :widths: 10 25 40 25

   * - ID
     - Caso de Prueba
     - Procedimiento de Ejecucion
     - Resultado Esperado (OK)
   * - **TC-01**
     - Acceso a Portales via Balanceador
     - Ingresar a ``https://api.tu-dominio.com/publisher`` y ``https://api.tu-dominio.com/devportal`` con usuario administrativo.
     - Login OAuth2 exitoso sin redireccion al puerto ``:9443`` en la URL.
   * - **TC-02**
     - Publicacion y Sincronizacion de API
     - Crear y publicar una API REST de prueba (``HealthCheckAPI v1.0.0``) en Publisher; verificar sincronizacion entre nodos.
     - La API queda publicada y visible en DevPortal y disponible en ambos nodos del cluster.
   * - **TC-03**
     - Balanceo Activo-Activo del Gateway
     - Generar un token de acceso y ejecutar 20 peticiones ``curl -k https://api.tu-dominio.com/healthcheck/1.0.0``; inspeccionar logs de acceso en Nodo 1 y Nodo 2.
     - Ambas instancias atienden peticiones con codigo HTTP ``200``.
   * - **TC-04**
     - Continuidad ante Caida de Nodo 1 (Failover)
     - Ejecutar ``systemctl stop wso2apim`` en ``lrkprdappdckposapi01`` mientras se envian peticiones continuas al endpoint del Gateway.
     - Nginx redirige el 100% del trafico a ``lrkprdappdckposapi02`` sin perdida de peticiones ni errores de autenticacion de token.
   * - **TC-05**
     - Reincorporacion del Nodo 1
     - Ejecutar ``systemctl start wso2apim`` en ``lrkprdappdckposapi01`` y verificar ``wso2carbon.log``.
     - El Nodo 1 se reincorpora al cluster WKA (``Members [2]``) y vuelve a recibir trafico balanceado desde Nginx.
