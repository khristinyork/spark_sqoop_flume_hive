# Para que convivan en la misma red el docker de cloudera y el docker-compose de Confluent:

* El Resource Manager de cloudera tiene el puerto 8088
y en el docker-compose de Confluence el puerto de kql-server tiene el mismo puerto el 8088
 por eso cambiamos en el docker_compouse.yml el puerto de kdql-server a 8988

* Además el docker de cloudera debe estar en el misma red que el docker-compose de  Confluent 

--network=cp-all-in-one_default:


#### sudo docker run --hostname=quickstart.cloudera  --network=cp-all-in-one_default --privileged=true -t -i -v /home/docker-clodera:/src --publish-all=true -p 8088:58088 -p 50090:50090 -p 50070:50070 -p 50075:50075 -p 8042:8042 -p 60030:60030 -p 25000:25000 -p 25010:25010 -p 8983:8983 -p 11000:11000 repository/cloudera_spark2:1 /usr/bin/docker-quickstart





#### docker-compose .yml
  version: '2'
  
    services:
      zookeeper:
        image: confluentinc/cp-zookeeper:5.3.1
        hostname: zookeeper
        container_name: zookeeper
        ports:
          - "2181:2181"
        environment:
          ZOOKEEPER_CLIENT_PORT: 2181
          ZOOKEEPER_TICK_TIME: 2000
          
      broker:
        image: confluentinc/cp-enterprise-kafka:5.3.1
        hostname: broker
        container_name: broker
        depends_on:
          - zookeeper
        ports:
          - "29092:29092"
          - "9092:9092"
        environment:
          KAFKA_BROKER_ID: 1
          KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
          KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
          KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
          KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
          KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
          KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
          CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: broker:29092
          CONFLUENT_METRICS_REPORTER_ZOOKEEPER_CONNECT: zookeeper:2181
          CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
          CONFLUENT_METRICS_ENABLE: 'true'
          CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'
      schema-registry:
        image: confluentinc/cp-schema-registry:5.3.1
        hostname: schema-registry
        container_name: schema-registry
        depends_on:
          - zookeeper
          - broker
        ports:
          - "8081:8081"
        environment:
          SCHEMA_REGISTRY_HOST_NAME: schema-registry
          SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: 'zookeeper:2181'

      connect:
        image: cnfldemos/kafka-connect-datagen:0.1.3-5.3.1
        hostname: connect
        container_name: connect
        depends_on:
          - zookeeper
          - broker
      - schema-registry
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: 'broker:29092'
      CONNECT_REST_ADVERTISED_HOST_NAME: connect
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: compose-connect-group
      CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_FLUSH_INTERVAL_MS: 10000
      CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_INTERNAL_KEY_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_INTERNAL_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      # CLASSPATH required due to CC-2422
      CLASSPATH: /usr/share/java/monitoring-interceptors/monitoring-interceptors-5.3.1.jar
      CONNECT_PRODUCER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor"
      CONNECT_CONSUMER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor"
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
      CONNECT_LOG4J_LOGGERS: org.apache.zookeeper=ERROR,org.I0Itec.zkclient=ERROR,org.reflections=ERROR

      control-center:
        image: confluentinc/cp-enterprise-control-center:5.3.1
        hostname: control-center
        container_name: control-center
        depends_on:
          - zookeeper
          - broker
          - schema-registry
          - connect
          - ksql-server
        ports:
          - "9021:9021"
        environment:
          CONTROL_CENTER_BOOTSTRAP_SERVERS: 'broker:29092'
          CONTROL_CENTER_ZOOKEEPER_CONNECT: 'zookeeper:2181'
          CONTROL_CENTER_CONNECT_CLUSTER: 'connect:8083'
          CONTROL_CENTER_KSQL_URL: "http://ksql-server:8988"
          CONTROL_CENTER_KSQL_ADVERTISED_URL: "http://localhost:8988"
          CONTROL_CENTER_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
          CONTROL_CENTER_REPLICATION_FACTOR: 1
          CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS: 1
          CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS: 1
          CONFLUENT_METRICS_TOPIC_REPLICATION: 1
          PORT: 9021

      ksql-server:
        image: confluentinc/cp-ksql-server:5.3.1
        hostname: ksql-server
        container_name: ksql-server
        depends_on:
          - broker
          - connect
        ports:
          - "8988:8988"
    environment:
      KSQL_CONFIG_DIR: "/etc/ksql"
      KSQL_LOG4J_OPTS: "-Dlog4j.configuration=file:/etc/ksql/log4j-rolling.properties"
      KSQL_BOOTSTRAP_SERVERS: "broker:29092"
      KSQL_HOST_NAME: ksql-server
      KSQL_LISTENERS: "http://0.0.0.0:8988"
      KSQL_CACHE_MAX_BYTES_BUFFERING: 0
      KSQL_KSQL_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
      KSQL_PRODUCER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor"
      KSQL_CONSUMER_INTERCEPTOR_CLASSES: "io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor"

      ksql-cli:
        image: confluentinc/cp-ksql-cli:5.3.1
        container_name: ksql-cli
        depends_on:
          - broker
          - connect
          - ksql-server
        entrypoint: /bin/sh
        tty: true

      ksql-datagen:
        # Downrev ksql-examples to 5.1.2 due to DEVX-798 (work around issues in 5.2.0)
        image: confluentinc/ksql-examples:5.3.1
        hostname: ksql-datagen
        container_name: ksql-datagen
        depends_on:
          - ksql-server
          - broker
          - schema-registry
          - connect
        command: "bash -c 'echo Waiting for Kafka to be ready... && \
                           cub kafka-ready -b broker:29092 1 40 && \
                           echo Waiting for Confluent Schema Registry to be ready... && \
                           cub sr-ready schema-registry 8081 40 && \
                           echo Waiting a few seconds for topic creation to finish... && \
                           sleep 11 && \
                           tail -f /dev/null'"
        environment:
          KSQL_CONFIG_DIR: "/etc/ksql"
          KSQL_LOG4J_OPTS: "-Dlog4j.configuration=file:/etc/ksql/log4j-rolling.properties"
          STREAMS_BOOTSTRAP_SERVERS: broker:29092
          STREAMS_SCHEMA_REGISTRY_HOST: schema-registry
          STREAMS_SCHEMA_REGISTRY_PORT: 8081

      rest-proxy:
        image: confluentinc/cp-kafka-rest:5.3.1
        depends_on:
          - zookeeper
          - broker
          - schema-registry
        ports:
          - 8082:8082
        hostname: rest-proxy
        container_name: rest-proxy
        environment:
          KAFKA_REST_HOST_NAME: rest-proxy
          KAFKA_REST_BOOTSTRAP_SERVERS: 'broker:29092'
          KAFKA_REST_LISTENERS: "http://0.0.0.0:8082"
          KAFKA_REST_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'



/usr/java/jdk1.8.0_131

export JAVA_HOME=/usr/java/jdk1.8.0_131


### network cp-all-in-one_default

* Para arrancar el docker compose de confluent :
### cd /home/cris/Confluent/examples/cp-all-in-one

### sudo docker-compose up  --build;sudo docker-compose ps;

* Run this command to verify that the services are up and running.

### sudo docker-compose ps;

* Parar todos los servicios:

### sudo docker container stop $(sudo docker container ls -a -q -f "label=io.confluent.docker");

* Navigate to the Control Center web interface at
 ### http://localhost:9021/.

* Meterte nos dentro de un servicio del docker-compose de confluence

### docker-compose exec ksql-cli ksql http://ksql-server:8088


### sudo docker-compose exec -i -t broker /bin/bash

cris@cris-VirtualBox:~/Confluent/examples/cp-all-in-one$ sudo docker-compose up -d --build;sudo docker-compose ps;

* Guardar la traza de log
### sudo docker-compose up >> console.log 2>&1

    Starting zookeeper       ... done
    Starting schema-registry ... done
    Starting rest-proxy      ... done
    Starting connect         ... done
    Starting ksql-server     ... done
    Starting ksql-datagen    ... done
    Starting control-center  ... done
    Starting ksql-cli        ... done

         Name                    Command                       State                                Ports                      
    ---------------------------------------------------------------------------------------------------------------------------
    broker            /etc/confluent/docker/run        Up                      0.0.0.0:29092->29092/tcp, 0.0.0.0:9092->9092/tcp
    connect           /etc/confluent/docker/run        Up (health: starting)   0.0.0.0:8083->8083/tcp, 9092/tcp                
    control-center    /etc/confluent/docker/run        Up                      0.0.0.0:9021->9021/tcp                          
    ksql-cli          /bin/sh                          Up                                                                      
    ksql-datagen      bash -c echo Waiting for K ...   Up                                                                      
    ksql-server       /etc/confluent/docker/run        Up                      0.0.0.0:8088->8088/tcp                          
    rest-proxy        /etc/confluent/docker/run        Up                      0.0.0.0:8082->8082/tcp                          
    schema-registry   /etc/confluent/docker/run        Up                      0.0.0.0:8081->8081/tcp                          
    zookeeper         /etc/confluent/docker/run        Up                      0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tc

* Docker network
### sudo docker network ls


    NETWORK ID          NAME                    DRIVER              SCOPE
    e39c19084fec        app-tier                bridge              local
    0174de20fb6a        bridge                  bridge              local
    a92ce5cb9325        cp-all-in-one_default   bridge              local
    74587912aa8a        host                    host                local
    077e19f151a4        none                    null                local
    
    
    
### ifconfig

    br-a92ce5cb9325 Link encap:Ethernet  direcciónHW 02:42:3a:33:41:c0  
              Direc. inet:172.19.0.1  Difus.:172.19.255.255  Másc:255.255.0.0
              Dirección inet6: fe80::42:3aff:fe33:41c0/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:177899 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:165721 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:14993835 (14.9 MB)  TX bytes:14046025 (14.0 MB)

    br-e39c19084fec Link encap:Ethernet  direcciónHW 02:42:2d:d6:82:74  
              Direc. inet:172.18.0.1  Difus.:172.18.255.255  Másc:255.255.0.0
              ACTIVO DIFUSIÓN MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:0 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:0 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:0 (0.0 B)  TX bytes:0 (0.0 B)

    docker0   Link encap:Ethernet  direcciónHW 02:42:23:61:72:50  
              Direc. inet:172.17.0.1  Difus.:172.17.255.255  Másc:255.255.0.0
              ACTIVO DIFUSIÓN MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:0 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:0 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:0 (0.0 B)  TX bytes:0 (0.0 B)

    enp0s3    Link encap:Ethernet  direcciónHW 08:00:27:40:0c:db  
              Direc. inet:10.0.2.15  Difus.:10.0.2.255  Másc:255.255.255.0
              Dirección inet6: fe80::769:2ec5:7536:5a3/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:825318 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:175158 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:1000 
              Bytes RX:1044652552 (1.0 GB)  TX bytes:14647977 (14.6 MB)

    lo        Link encap:Bucle local  
              Direc. inet:127.0.0.1  Másc:255.0.0.0
              Dirección inet6: ::1/128 Alcance:Anfitrión
              ACTIVO BUCLE FUNCIONANDO  MTU:65536  Métrica:1
              Paquetes RX:412970 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:412970 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:1000 
              Bytes RX:36509439 (36.5 MB)  TX bytes:36509439 (36.5 MB)

    veth1509176 Link encap:Ethernet  direcciónHW 56:c4:87:ca:4f:5c  
              Dirección inet6: fe80::54c4:87ff:feca:4f5c/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:57 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:151 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:5246 (5.2 KB)  TX bytes:106323 (106.3 KB)

    veth000c37b Link encap:Ethernet  direcciónHW fe:f3:ba:5a:af:f7  
              Dirección inet6: fe80::fcf3:baff:fe5a:aff7/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:28217 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:40863 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:2869807 (2.8 MB)  TX bytes:3036619 (3.0 MB)

    veth06b60e8 Link encap:Ethernet  direcciónHW ea:f5:0c:04:49:94  
              Dirección inet6: fe80::e8f5:cff:fe04:4994/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:9078 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:12888 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:2022728 (2.0 MB)  TX bytes:1298064 (1.2 MB)

    veth18ed0b8 Link encap:Ethernet  direcciónHW 5a:bc:d1:b6:9a:08  
              Dirección inet6: fe80::58bc:d1ff:feb6:9a08/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:973968 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:884254 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:175822465 (175.8 MB)  TX bytes:174069463 (174.0 MB)

    veth33703b8 Link encap:Ethernet  direcciónHW da:b9:2a:8d:c0:12  
              Dirección inet6: fe80::d8b9:2aff:fe8d:c012/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:67 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:112 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:4957 (4.9 KB)  TX bytes:11188 (11.1 KB)

    veth4b64af3 Link encap:Ethernet  direcciónHW 1a:9b:c4:bc:75:f0  
              Dirección inet6: fe80::189b:c4ff:febc:75f0/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:26790 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:39834 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:2551738 (2.5 MB)  TX bytes:2989094 (2.9 MB)

    veth688670c Link encap:Ethernet  direcciónHW aa:24:ad:7b:27:56  
              Dirección inet6: fe80::a824:adff:fe7b:2756/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:351694 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:273006 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:52904371 (52.9 MB)  TX bytes:28572244 (28.5 MB)

    vethbbe4a74 Link encap:Ethernet  direcciónHW 7a:07:28:9a:93:fb  
              Dirección inet6: fe80::7807:28ff:fe9a:93fb/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:0 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:66 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:0 (0.0 B)  TX bytes:7461 (7.4 KB)

    vethcc51369 Link encap:Ethernet  direcciónHW fa:d2:8f:c2:bb:86  
              Dirección inet6: fe80::f8d2:8fff:fec2:bb86/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:477989 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:617570 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:115714479 (115.7 MB)  TX bytes:141819750 (141.8 MB)

    vethfd7198a Link encap:Ethernet  direcciónHW 1a:98:57:b7:be:2b  
              Dirección inet6: fe80::1898:57ff:feb7:be2b/64 Alcance:Enlace
              ACTIVO DIFUSIÓN FUNCIONANDO MULTICAST  MTU:1500  Métrica:1
              Paquetes RX:16 errores:0 perdidos:0 overruns:0 frame:0
              Paquetes TX:65 errores:0 perdidos:0 overruns:0 carrier:0
              colisiones:0 long.colaTX:0 
              Bytes RX:1170 (1.1 KB)  TX bytes:8263 (8.2 KB)




