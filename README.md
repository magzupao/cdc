# cdc
Detecta los eventos de mysql y los propaga via kafka para que otro systema pueda consumirlo.

vamos a crear una red donde se detecte los cambios producidos en wordpress y los publique en kafka. 

Se van a crear dos archivos, porque dos y no solo uno, es que la primera red crea debzium con una mysql definida con una BBDD y el segundo solo se desplegara wordpress.

## 1 creamos la red debezium

debezium.yaml
```
#
# docker network create --gateway 172.18.0.1 --subnet 172.18.0.0/16 redecommerce
#
version: '3'

services:
  zookeeper:
    image: debezium/zookeeper:1.6
    hostname: zookeeper
    container_name: zoo
    ports:
      - '2181:2181'
      - '2888:2888'
      - '3888:3888'

  kafka:
    image: debezium/kafka:1.6
    container_name: kafka
    hostname: kafka
    environment: 
      ZOOKEEPER_CONNECT: zookeeper
    ports:
      - '9092:9092'
    depends_on:
      - zookeeper

  mysql:
    image: debezium/example-mysql:1.6
    container_name: mysql
    hostname: mysql
    environment:
      MYSQL_ROOT_PASSWORD: debezium
      MYSQL_USER: mysqluser
      MYSQL_PASSWORD: mysqlpw
    ports:
      - '3306:3306'

  kafka_connect:
    image: debezium/connect:1.6
    container_name: connect
    hostname: connect
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses
    ports:
      - '8083:8083'
    depends_on:
      - zookeeper
      - kafka
      - mysql

#
# Creating mysql connector
#

# curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.include.list": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'

# Getting connectors
# curl -H "Accept:application/json" localhost:8083/connectors/

# Getting inventory-connector
# curl -i -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector

```

## 2 creamos la BBDD que necesita wordpress

```
docker container exec -u root -it mysql bash
mysql -uroot -pdebezium

CREATE USER 'wordpress'@'%' IDENTIFIED BY 'wordpress';
GRANT ALL PRIVILEGES ON *.* TO wordpress@'%' IDENTIFIED BY 'wordpress';

CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON wordpress.* TO 'wordpress'@'%';
flush privileges;
```

## 3 creamos la instancia wordpress
```
version: '3'

services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    hostname: wordpress
    volumes:
      - wordpress_data:/var/www/html
    ports:
      - "8000:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: mysql:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress

volumes:
  wordpress_data: {}

```

## 4 Acciones a ejecutar
Ejecutamos lo siguiente:  
docker-compose -f debezium.yaml up

Al finalizar ingresamos al contenedor mysql y creamos la BBDD wordpress.  

docker container exec -u root -it mysql bash  

Despues ejecutamos:  
docker-compose -f debezium-wordpress.yaml up

## 5 establecemos la conexion entre mysql y debezium

Ejecutamos lo siguiente:
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "wordpress-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.include.list": "wordpress", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.wordpress" } }'

```

## 6 Kafka
Inicializamos un topic para ver los mensajes generados al detectar un cambio en la BBDD.

Creamos un topic que detecte cualquier cambio en la tabla wp_users
```
docker container exec -u root -it kafka bash

./kafka-topics.sh --list --zookeeper zookeeper:2181

./kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic dbserver1.wordpress.wp_users --from-beginning
```

## 7 Otra forma  
  
Ejecutamos el fichero yaml usando la red por default debezium-sin-red.yaml:
```
docker-compose -f debezium-sin-red.yaml up
```
Ejecutamos el punto 2.  
  
Ejecutamos en el contenedor 'connect', tener en cuenta de ejecutar el comando en una sola linea, reemplazamos las IPs de:  
localhost:8083 = 172.20.0.6:8083  
mysql:3306 = 172.20.0.4:3306  
kafka:9092 = 172.20.0.5:9092  
  
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 172.20.0.6:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "172.20.0.4", "database.port": "3306", "database.user": "root", "database.password": "debezium", "database.server.name": "dbserver1", "database.include.list": "inventory", "database.history.kafka.bootstrap.servers": "172.20.0.5:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'

```

La respuesta de ejecutar el comando anterior nos da la siguiente respuesta:

```
HTTP/1.1 201 Created
Date: Sat, 07 May 2022 19:23:48 GMT
Location: http://172.20.0.6:8083/connectors/inventory-connector
Content-Type: application/json
Content-Length: 471
Server: Jetty(9.4.38.v20210224)

{"name":"inventory-connector","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","tasks.max":"1","database.hostname":"172.20.0.4","database.port":"3306","database.user":"root","database.password":"debezium","database.server.name":"dbserver1","database.include.list":"inventory","database.history.kafka.bootstrap.servers":"172.20.0.5:9092","database.history.kafka.topic":"dbhistory.inventory","name":"inventory-connector"},"tasks":[],"type":"source"}
```  
  
Ingresamos al contenedor 'kafka' y vemos los topics creados:
```
./kafka-topics.sh --list --zookeeper zookeeper:2181
```

Obtenemos esta respuesta, se ha creado un topic general 'dbserver1' y por cada tabla.
```
__consumer_offsets
dbhistory.inventory
dbserver1
dbserver1.inventory.addresses
dbserver1.inventory.customers
dbserver1.inventory.geom
dbserver1.inventory.orders
dbserver1.inventory.products
dbserver1.inventory.products_on_hand
my_connect_configs
my_connect_offsets
my_connect_statuses
```

Invocamos un topic para visualizar las trazas:
```
./kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic dbserver1 --from-beginning
```

Obtenemos la respuesta:
```
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"} ...
```



