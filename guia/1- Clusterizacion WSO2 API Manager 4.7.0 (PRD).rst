
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
   no lista MariaDB explícitamente entre las bases de datos probadas. MariaDB es
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

================================================================================
1. Descarga e Instalación de JDK 21
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
2. Descarga e Instalación de WSO2 API Manager 4.7.0
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
3. Creación de Bases de Datos MariaDB y Driver JDBC
================================================================================

3.1 Creación de Esquemas y Usuario
-----------------------------------

**Ejecutar en el servidor de base de datos** ``10.132.0.56`` (o solicitar al DBA):

.. code-block:: sql

   CREATE DATABASE WSO2AM_DB CHARACTER SET latin1 COLLATE latin1_bin;
   CREATE DATABASE WSO2SHARED_DB CHARACTER SET latin1 COLLATE latin1_bin;

   CREATE USER 'wso2_prd_user'@'10.133.4.187' IDENTIFIED BY 'TU_PASSWORD_SEGURO_BD';
   CREATE USER 'wso2_prd_user'@'10.133.4.188' IDENTIFIED BY 'TU_PASSWORD_SEGURO_BD';

   GRANT ALL PRIVILEGES ON WSO2AM_DB.* TO 'wso2_prd_user'@'10.133.4.187';
   GRANT ALL PRIVILEGES ON WSO2SHARED_DB.* TO 'wso2_prd_user'@'10.133.4.187';
   GRANT ALL PRIVILEGES ON WSO2AM_DB.* TO 'wso2_prd_user'@'10.133.4.188';
   GRANT ALL PRIVILEGES ON WSO2SHARED_DB.* TO 'wso2_prd_user'@'10.133.4.188';

   FLUSH PRIVILEGES;

3.2 Inicialización de Esquemas
-------------------------------

**Ejecutar en el Nodo 1** usando los scripts SQL incluidos con WSO2:

.. code-block:: bash

   # Base de datos compartida
   mysql -h 10.132.0.56 -u wso2_prd_user -p WSO2SHARED_DB \
     < /opt/wso2/wso2am-4.7.0/dbscripts/mysql.sql

   # Base de datos APIM
   mysql -h 10.132.0.56 -u wso2_prd_user -p WSO2AM_DB \
     < /opt/wso2/wso2am-4.7.0/dbscripts/apimgt/mysql.sql

3.3 Colocación del Driver JDBC
-------------------------------

Descargar ``mariadb-java-client-3.x.x.jar`` y colocarlo en **ambos nodos**:

.. code-block:: bash

   cp mariadb-java-client-3.x.x.jar \
     /opt/wso2/wso2am-4.7.0/repository/components/lib/
   chown wso2:wso2 \
     /opt/wso2/wso2am-4.7.0/repository/components/lib/mariadb-java-client-3.x.x.jar

================================================================================
4. Configuración de Keystore y Truststore
================================================================================

**Punto clave de un clúster activo-activo: ambos nodos deben usar exactamente
el mismo keystore y truststore.**

4.1 Generación en el Nodo 1
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

4.2 Replicación hacia el Nodo 2
--------------------------------

.. code-block:: bash

   # Ejecutar en el Nodo 1
   rsync -avz /opt/wso2/wso2am-4.7.0/repository/resources/security/ \
     root@10.133.4.188:/opt/wso2/wso2am-4.7.0/repository/resources/security/

   # En el Nodo 2 corregir propietario
   ssh root@10.133.4.188 \
     "chown -R wso2:wso2 /opt/wso2/wso2am-4.7.0/repository/resources/security/"

================================================================================
5. Montaje del Sistema de Archivos Compartido NFS
================================================================================

**Este es el mecanismo principal recomendado oficialmente para la sincronización
de artefactos en despliegues activo-activo All-in-One.**

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
6. Configuración de deployment.toml
================================================================================

**Editar en ambos nodos** el archivo
``/opt/wso2/wso2am-4.7.0/repository/conf/deployment.toml``.

Ajustar ``node_ip`` según el nodo (``10.133.4.187`` en Nodo 1,
``10.133.4.188`` en Nodo 2) y asegurar que la clave en ``[encryption]`` sea
**idéntica en ambos servidores**.

.. code-block:: toml

   [server]
   hostname = "api.tu-dominio.com"      # Dominio VIP de F5
   node_ip = "10.133.4.187"             # Nodo 2 cambiar a 10.133.4.188

   [transport.https.properties]
   proxyPort = 443                       # Evita que la URL generada incluya :9443

   # Clave de cifrado: ambos nodos deben ser idénticos, 64 caracteres hex
   [encryption]
   key = "GENERAR_UNA_CLAVE_HEX_DE_64_CARACTERES_Y_USARLA_EN_AMBOS_NODOS"

   # Base de datos APIM
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

.. note::

   **Sobre clustering Hazelcast**: La documentación oficial de despliegue
   activo-activo All-in-One de WSO2 se basa principalmente en **base de datos
   compartida + sistema de archivos compartido (NFS)**. El clúster de Hazelcast
   (``[clustering]``) **no es un requisito obligatorio** para este escenario.
   Si su equipo tiene requisitos específicos de rendimiento de caché, consulte
   por separado la documentación de clustering de 4.7.0 para agregarlo, pero no
   lo incluya a ciegas.

================================================================================
7. Configuración del Servicio systemd
================================================================================

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

================================================================================
8. Ajustes del Sistema Operativo y Firewall
================================================================================

8.1 Ajustes del Kernel (ambos nodos)
-------------------------------------

.. code-block:: bash

   sudo tee /etc/sysctl.d/99-wso2.conf > /dev/null << 'EOF'
   vm.max_map_count = 262144
   net.core.somaxconn = 4096
   EOF
   sudo sysctl --system

8.2 Límites de Descriptores de Archivos
----------------------------------------

.. code-block:: bash

   sudo tee /etc/security/limits.d/99-wso2.conf > /dev/null << 'EOF'
   wso2 soft nofile 65535
   wso2 hard nofile 65535
   wso2 soft nproc 32768
   wso2 hard nproc 32768
   EOF

8.3 Firewall (firewalld)
-------------------------

Abrir los puertos necesarios para la comunicación del clúster entre los dos
nodos (**4000**, **9611**, **9711**, **5672**), así como los puertos **9443**
y **8243** para el acceso desde F5. La configuración detallada del F5 queda
fuera del alcance de este manual.

.. code-block:: bash

   # Ejemplo en el Nodo 1
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="4000" protocol="tcp" accept'
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="9611-9711" protocol="tcp" accept'
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.133.4.188" port port="5672" protocol="tcp" accept'
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<SUBRED_F5>" port port="9443" protocol="tcp" accept'
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<SUBRED_F5>" port port="8243" protocol="tcp" accept'
   sudo firewall-cmd --reload

================================================================================
9. Inicio y Verificación
================================================================================

9.1 Inicio Secuencial
----------------------

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

9.2 Matriz de Pruebas de Aceptación
------------------------------------

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
10. Resumen de Riesgos y Recomendaciones
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
     - **No está en la lista de pruebas oficiales.** Validar exhaustivamente
       sentencias SQL y driver antes de producción.
   * - NFS compartido
     - Recomendado
     - Método de sincronización recomendado oficialmente; depende de la
       disponibilidad del NFS.
   * - Clustering Hazelcast
     - Opcional
     - No habilitado por defecto en esta guía; el estándar activo-activo
       oficial se basa en BD + NFS compartidos.

.. note::

   **Vía para certificación oficial**: Tener una suscripción válida de WSO2 y
   enviar la configuración del entorno (especialmente la elección de MariaDB y
   la arquitectura NFS) al soporte de WSO2 para revisión; o encargar a un socio
   certificado de WSO2 una auditoría de implementación. Los pasos anteriores
   pueden servir como base para dicha auditoría.
