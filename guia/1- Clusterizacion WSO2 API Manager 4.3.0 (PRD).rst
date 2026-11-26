=============================================================================
Plan de Trabajo y Procedimiento: Clusterizacion WSO2 API Manager 4.3.0 (PRD)
=============================================================================

.. contents:: Contenido
   :depth: 2
   :local:

Parte 1: Plan de Trabajo — Clusterizacion WSO2 APIM 4.3.0 (PRD)
==================================================================

Objetivo y Alcance
------------------

Conformar un cluster activo-activo de 2 nodos para **WSO2 API Manager 4.3.0** en el entorno de Produccion:

* **Nodo 1:** ``lrkprdappdckposapi01`` (``10.133.4.187``)
* **Nodo 2:** ``lrkprdappdckposapi02`` (``10.133.4.188``)

Prerrequisitos e Insumos Necesarios
-----------------------------------

Antes de iniciar la ventana de ejecucion, confirma disponibilidad de:

* **Base de datos relacional centralizada** (PostgreSQL, MariaDB, MySQL, Oracle o MSSQL) accesible desde ambas IPs (``10.133.4.187`` y ``10.133.4.188``) y el archivo ``.jar`` del driver JDBC compatible.
* **DNS / FQDN del servicio balanceado** (ejemplo: ``api.tu-dominio.com`` o ``apim-prd.tu-dominio.local``) apuntando a la VIP del balanceador o proxy reverso.
* **Certificado TLS** (``.crt``/``.pem`` y clave privada ``.key``, o ``.p12``/``.jks``) emitido para el FQDN balanceado.
* **Mecanismo de sincronizacion de artefactos** definido (almacenamiento compartido NFS/NAS montado en ambos nodos para ``repository/deployment/server`` o integracion Git/CI-CD segun estandar interno).
* **Reglas de firewall internas** habilitadas entre ``10.133.4.187`` y ``10.133.4.188`` (puertos ``9443``, ``8243``, ``8280``, puertos de mensajeria/throttling ``5672``/``9711``/``9611`` y puerto de cluster Hazelcast ``4000`` TCP si aplica).

Fases de Ejecucion
-------------------

.. list-table:: Cronograma de Fases en Produccion
   :header-rows: 1
   :widths: 15 35 30 20

   * - Fase
     - Actividad Principal
     - Responsable / Entregable
     - Ventana
   * - **Fase 1**
     - Respaldo integral y verificacion de red, DNS y firewall
     - Backup de ``<APIM_HOME>`` y BDs actuales
     - Pre-ventana
   * - **Fase 2**
     - Creacion e inicializacion de bases de datos compartidas (``WSO2AM_DB``, ``WSO2SHARED_DB``)
     - Esquemas SQL cargados + JDBC en ambos nodos
     - Ventana (Paso 1)
   * - **Fase 3**
     - Configuracion de certificados TLS y keystores
     - Keystore/Truststore actualizados en Nodo 1 y Nodo 2
     - Ventana (Paso 2)
   * - **Fase 4**
     - Configuracion de ``deployment.toml`` y sincronizacion de artefactos
     - Archivos ``deployment.toml`` homologados en ambos nodos
     - Ventana (Paso 3 y Paso 4)
   * - **Fase 5**
     - Configuracion de balanceador / Nginx reverse proxy
     - Upstreams, TLS y health checks activos
     - Ventana (Paso 5)
   * - **Fase 6**
     - Arranque secuencial, pruebas funcionales y prueba de failover
     - Validacion de logs, portales, Gateway y caida controlada
     - Ventana (Paso 6)

Parte 2: Procedimiento Paso a Paso (Ejecucion en PRD)
=======================================================

Paso 0: Respaldo y Detencion Controlada
--------------------------------------

Ejecuta en **ambos nodos** (``10.133.4.187`` y ``10.133.4.188``):

1. Deten el servicio actual de WSO2 API Manager:

   .. code-block:: bash

      systemctl stop wso2apim
      # O si se gestiona con el script nativo:
      # <APIM_HOME>/bin/api-manager.sh stop

2. Crea un respaldo completo del directorio de configuracion y seguridad antes de modificar archivos:

   .. code-block:: bash

      tar -czvf /root/backup_wso2apim430_prd_$(hostname)_$(date +%Y%m%d).tar.gz \
        <APIM_HOME>/repository/conf \
        <APIM_HOME>/repository/resources/security \
        <APIM_HOME>/repository/deployment/server

3. Verifica conectividad entre nodos y resolucion de nombres en ``/etc/hosts`` o DNS en **ambos servidores**:

   .. code-block:: text

      10.133.4.187   lrkprdappdckposapi01
      10.133.4.188   lrkprdappdckposapi02
      <IP_VIP_O_PROXY>   api.tu-dominio.com

Paso 1: Preparar Bases de Datos Compartidas y Driver JDBC
--------------------------------------------------------

En un cluster de WSO2 APIM 4.3.0, **no** se usa la base de datos embebida H2. Ambos nodos deben apuntar a las mismas bases de datos externas.

1. En el servidor de base de datos de Produccion, crea dos bases de datos vacias con codificacion UTF-8:

   * ``WSO2AM_DB`` (datos de APIs, suscripciones, aplicaciones y tokens).
   * ``WSO2SHARED_DB`` (registro de configuracion, gobernanza y gestion de usuarios).

2. Ejecuta los scripts SQL incluidos en ``<APIM_HOME>/dbscripts/`` segun el motor de base de datos:

   * Contra ``WSO2SHARED_DB``: ejecuta el script principal del motor (por ejemplo, ``postgresql.sql`` o ``mysql.sql``).
   * Contra ``WSO2AM_DB``: ejecuta el script de APIM (por ejemplo, ``apimgt/postgresql.sql`` o ``apimgt/mysql.sql``).

3. Copia el driver JDBC ``.jar`` en **ambos nodos** (``lrkprdappdckposapi01`` y ``lrkprdappdckposapi02``):

   .. code-block:: bash

      cp /ruta/al/driver-jdbc.jar <APIM_HOME>/repository/components/lib/
      chown wso2user:wso2group <APIM_HOME>/repository/components/lib/driver-jdbc.jar

Paso 2: Configurar Keystores y Certificados TLS
----------------------------------------------

Ambos nodos deben compartir exactamente los mismos almacenes de claves (``wso2carbon.jks`` o ``.p12`` y ``client-truststore.jks``) para evitar fallos en validacion interna de tokens JWT y conexiones SSL entre componentes.

1. Si se utiliza un certificado propio para el FQDN del balanceador (``api.tu-dominio.com``), importalo en el keystore del **Nodo 1** (``lrkprdappdckposapi01``) y agrega su certificado publico (y la cadena CA) al ``client-truststore.jks``.
2. Copia los archivos resultantes de ``<APIM_HOME>/repository/resources/security/`` desde el **Nodo 1** hacia el **Nodo 2** (``lrkprdappdckposapi02``):

   .. code-block:: bash

      rsync -avz <APIM_HOME>/repository/resources/security/ \
        root@10.133.4.188:<APIM_HOME>/repository/resources/security/

Paso 3: Configurar Sincronizacion de Artefactos (Gateway / Synapse)
------------------------------------------------------------------

Cuando se publica o modifica una API en un nodo, el otro nodo debe recibir los artefactos de Synapse generados (``repository/deployment/server/synapse-configs`` y ``executionplans``).

* **Opcion A (Almacenamiento compartido NFS):** Monta un volumen NFS compartido en ``<APIM_HOME>/repository/deployment/server`` en ambos nodos (``10.133.4.187`` y ``10.133.4.188``) con permisos de lectura y escritura para el usuario de ejecucion de WSO2.
* **Opcion B (Git / Artifact Synchronization):** Configura ``[apim.sync_runtime_artifacts.gateway]`` en ``deployment.toml`` apuntando a un repositorio Git interno dedicado al entorno PRD.

Paso 4: Configurar ``deployment.toml`` en Ambos Nodos
----------------------------------------------------

Edita ``<APIM_HOME>/repository/conf/deployment.toml`` en **ambos nodos**. La configuracion es homologa salvo por ``node_ip`` y ``local_member_host``, que corresponden a la IP propia de cada servidor.

.. code-block:: toml

   [server]
   hostname = "api.tu-dominio.com"
   node_ip = "10.133.4.187" # Cambiar a "10.133.4.188" en lrkprdappdckposapi02
   mode = "single"

   [super_admin]
   username = "admin"
   password = "TU_PASSWORD_SEGURO"
   create_admin_account = true

   # --- Base de datos compartida (Registry & User Management) ---
   [database.shared_db]
   type = "postgresql" # Cambiar a mysql / oracle / mssql segun corresponda
   url = "jdbc:postgresql://<IP_SERVIDOR_BD>:5432/WSO2SHARED_DB"
   username = "wso2_prd_user"
   password = "TU_PASSWORD_BD"

   # --- Base de datos de API Manager ---
   [database.apim_db]
   type = "postgresql"
   url = "jdbc:postgresql://<IP_SERVIDOR_BD>:5432/WSO2AM_DB"
   username = "wso2_prd_user"
   password = "TU_PASSWORD_BD"

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

   # --- Clustering (Coordinacion entre Nodo 1 y Nodo 2) ---
   [clustering]
   membership_scheme = "wka"
   domain = "wso2.apim.prd.domain"
   local_member_host = "10.133.4.187" # Cambiar a "10.133.4.188" en lrkprdappdckposapi02
   local_member_port = "4000"

   [[clustering.members]]
   host = "10.133.4.187"
   port = 4000

   [[clustering.members]]
   host = "10.133.4.188"
   port = 4000

   # --- Gateway Environment (URLs publicas balanceadas) ---
   [[apim.gateway.environment]]
   name = "Production and Sandbox"
   type = "hybrid"
   display_in_api_console = true
   description = "Gateway Cluster Produccion"
   show_as_token_endpoint_url = true
   service_url = "https://localhost:${mgt.transport.https.port}/services/"
   username = "${admin.username}"
   password = "${admin.password}"
   ws_endpoint = "ws://api.tu-dominio.com:9099"
   wss_endpoint = "wss://api.tu-dominio.com:8099"
   http_endpoint = "http://api.tu-dominio.com:80"
   https_endpoint = "https://api.tu-dominio.com:443"

   # --- Trafico y Throttling compartido entre nodos ---
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
   Si el balanceador o Nginx reverse proxy recibe en el puerto ``443`` y reenvia hacia los puertos internos de WSO2 (``9443`` para portales/administracion y ``8243`` para trafico de APIs), configura los parametros ``proxyPort`` correspondientes en ``deployment.toml`` para evitar que las redirecciones OAuth2 incluyan ``:9443`` en la URL publica.

Paso 5: Configurar Balanceador / Reverse Proxy
---------------------------------------------

Configura los bloques de balanceo frente a ``10.133.4.187`` y ``10.133.4.188``:

1. **Upstream para Portales (Publisher, DevPortal, Carbon, OAuth2 — puerto ``9443``):**

   * Habilita afinidad de sesion (``ip_hash`` o stickiness) para sesiones administrativas y de portales.
   * Enruta contextos como ``/publisher``, ``/devportal``, ``/carbon``, ``/oauth2``, ``/oidc`` y ``/api/am``.

2. **Upstream para Trafico de APIs (Gateway — puerto ``8243``):**

   * Balanceo *round-robin* o *least_conn* hacia ``10.133.4.187:8243`` y ``10.133.4.188:8243``.
   * Configura **health checks** contra el endpoint de salud o una API de prueba y verifica la continuidad del servicio ante caida de un miembro.

Paso 6: Arranque Secuencial y Validacion del Cluster
---------------------------------------------------

1. **Arranca primero el Nodo 1 (``lrkprdappdckposapi01``):**

   .. code-block:: bash

      systemctl start wso2apim
      tail -f <APIM_HOME>/repository/logs/wso2carbon.log

   Espera hasta confirmar el arranque limpio sin errores de conexion a base de datos ni de red.

2. **Arranca despues el Nodo 2 (``lrkprdappdckposapi02``):**

   .. code-block:: bash

      systemctl start wso2apim
      tail -f <APIM_HOME>/repository/logs/wso2carbon.log

   Verifica en ``wso2carbon.log`` que el nodo se une al cluster con ``10.133.4.187:4000``.

3. **Lista de verificacion de aceptacion:**

   * **Sincronizacion de APIs:** Publica una API de prueba en Publisher y valida su visibilidad en DevPortal a traves del FQDN balanceado.
   * **Distribucion de carga:** Ejecuta invocaciones repetidas contra una API en el Gateway y revisa los logs de acceso de ambos nodos.
   * **Prueba de failover:** Deten temporalmente el servicio en ``lrkprdappdckposapi01``, comprueba que las peticiones al Gateway siguen respondiendo mediante ``lrkprdappdckposapi02``, y vuelve a iniciar ``lrkprdappdckposapi01``.
