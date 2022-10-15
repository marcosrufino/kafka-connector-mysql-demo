# Start Zookeeper
```
docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 zookeeper:3.5.5
```
# Start Kafka Broker
```
docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:1.0
```
# Start MySQL (don't forget to replace the mysql.cnf path with the one pointing to the one in your filesystem)
```config
[mysqld]
skip-host-cache
skip-name-resolve
user=mysql
symbolic-links=0

server-id = 223344
log_bin = mysql-bin
expire_logs_days = 1
binlog_format = row
```


```
docker run -it --rm --name mysql -p 3306:3306 -v ${PWD}/mysql.cnf:/etc/mysql/conf.d/mysql.cnf \
-e MYSQL_ROOT_PASSWORD=password -e MYSQL_USER=globomantics -e MYSQL_PASSWORD=password -e MYSQL_DATABASE=globomantics  \
mysql:5.7
```
# Start MySQL terminal
```
docker run -it --rm --name mysqlterm --link mysql --rm mysql:5.7 \
sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
```

```
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT  ON *.* TO 'globomantics' IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES ON globomantics.* TO 'globomantics'@'%';

USE globomantics;

CREATE  TABLE `articles` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `author` VARCHAR(255) NOT NULL ,
  `title` VARCHAR(255) NOT NULL,
  `diagram` VARCHAR(255) ,
  `body` VARCHAR(255) ,
  `footnote` VARCHAR(255) )
ENGINE = InnoDB;

DESCRIBE `articles`;

INSERT INTO articles ( author, title, diagram, body, footnote ) VALUES
( "Tom", "Digital Transformation", "https://globomantics.com/digital-transformation.png", "A new era of computers..." , "1 - Internet Of Things" ),
( "Maria", "Being Data Driven", "https://globomantics.com/data-driven.png", "The decision making process ..." , "1 - large ses of data" );




```

# Start Kafka Connect
```
docker run -it --rm --name connect -p 8083:8083 \
-e GROUP_ID=1 \
-e CONFIG_STORAGE_TOPIC=kafka_connect_configs \
-e OFFSET_STORAGE_TOPIC=kafka_connect_offsets \
-e STATUS_STORAGE_TOPIC=kafka_connect_statuses \
--link zookeeper:zookeeper \
--link kafka:kafka \
--link mysql:mysql \
debezium/connect:1.0
```
```
curl -H "Accept:application/json" localhost:8083/
```

```
docker exec -it kafka bash

./kafka-topics.sh --list --bootstrap-server 172.17.0.3:9092
```
 - Return
```
__consumer_offsets
kafka_connect_configs
kafka_connect_offsets
kafka_connect_statuses
```

```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '
{
  "name": "articles-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "globomantics",
    "database.password": "password",
    "database.server.id": "223344",
    "database.server.name": "globomantics",
    "database.whitelist": "globomantics",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes"
    }
}'
```

```
./kafka-topics.sh --list --bootstrap-server 172.17.0.3:9092
```


```
./kafka-console-consumer --bootstrap-server 172.17.0.3:9092 --topic articles --from-beginning

```


```
INSERT INTO articles ( author, title, diagram, body, footnote ) VALUES
( "Paul", "Kafka Connect", "https://globomantics.com/kafka-connect.png", "It will never get easier..." , "1 - High Throughput" );

UPDATE articles SET  footnote = "1 - large sets of data" WHERE author = "Maria";

DELETE FROM articles WHERE author='Tom';
```
