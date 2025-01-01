# Debezium to monitor a MySQL database Using Kafka

https://debezium.io/documentation/reference/tutorial.html

Introduction to Debezium
Debezium is a distributed platform that turns your existing databases into event streams, so applications can see and respond immediately to each row-level change in the databases.

Debezium is built on top of Apache Kafka and provides Kafka Connect compatible connectors that monitor specific database management systems. Debezium records the history of data changes in Kafka logs, from where your application consumes them. This makes it possible for your application to easily consume all of the events correctly and completely. Even if your application stops unexpectedly, it will not miss anything: when the application restarts, it will resume consuming the events where it left off.

Debezium includes multiple connectors. In this tutorial, you will use the MySQL connector(https://debezium.io/documentation/reference/connectors/mysql.html).

## Starting the services

You could go with following as well: https://github.com/debezium/debezium-examples/blob/master/tutorial/docker-compose-mysql.yaml


Using Debezium requires three separate services: ZooKeeper, Kafka, and the Debezium connector service. 

In this tutorial, you will set up a single instance of each service using Docker and the Debezium Docker images.

To start the services needed for this tutorial, you must:

- Start Zookeeper

- Start Kafka

- Start a MySQL database

- Start a MySQL command line client

- Start Kafka Connect

## Use of docker

This tutorial uses Docker and the Debezium Docker images to run the ZooKeeper, Kafka, Debezium, and MySQL services. Running each service in a separate container simplifies the setup so that you can see Debezium in action.

### Starting Zookeeper

ZooKeeper is the first service you must start.

`docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:1.0`

Verify that ZooKeeper started and is listening on port 2181:

```sh
Starting up in standalone mode
ZooKeeper JMX enabled by default
Using config: /zookeeper/conf/zoo.cfg
2017-09-21 07:15:55,417 - INFO  [main:QuorumPeerConfig@134] - Reading configuration from: /zookeeper/conf/zoo.cfg
2017-09-21 07:15:55,419 - INFO  [main:DatadirCleanupManager@78] - autopurge.snapRetainCount set to 3
2017-09-21 07:15:55,419 - INFO  [main:DatadirCleanupManager@79] - autopurge.purgeInterval set to 1
...
port 0.0.0.0/0.0.0.0:2181 
```

### Starting Kafka

After starting ZooKeeper, you can start Kafka in a new container.

`docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:1.0`

Verify that Kafka started.

```sh
...
2017-09-21 07:16:59,085 - INFO  [main-EventThread:ZkClient@713] - zookeeper state changed (SyncConnected)
2017-09-21 07:16:59,218 - INFO  [main:Logging$class@70] - Cluster ID = LPtcBFxzRvOzDSXhc6AamA
...
2017-09-21 07:16:59,649 - INFO  [main:Logging$class@70] - [Kafka Server 1], started  
```

### Starting a MySQL database

At this point, you have started ZooKeeper and Kafka, but you still need a database server from which Debezium can capture changes. In this procedure, you will start a MySQL server with an example database (inventory database).

This command runs a new container using version 1.0 of the debezium/example-mysql image, which is based on the mysql:5.7 image. It also defines and populates a sample inventory database:

`docker run -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.0`

Verify that the MySQL server starts.

The MySQL server starts and stops a few times as the configuration is modified. You should see output similar to the following:

```sh
...
017-09-21T07:18:50.824629Z 0 [Note] mysqld: ready for connections.
Version: '5.7.19-log'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
```

### Starting a MySQL command line client

https://github.com/debezium/debezium-examples/blob/master/tutorial/docker-compose-mysql.yaml


After starting MySQL, you start a MySQL command line client so that you access the sample inventory database.

`docker run -it --rm --name mysqlterm --link mysql --rm mysql:5.7 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'`

docker run --name postgresterm --link postgres --rm postgres:11 -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 postgres

Verify that the MySQL command line client started.

```sh
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.17-log MySQL Community Server (GPL)

mysql>
```

At the mysql> command prompt, switch to the inventory database:

`mysql> use inventory;`

List the tables in the database:

```sh
mysql> show tables;
+---------------------+
| Tables_in_inventory |
+---------------------+
| addresses           |
| customers           |
| geom                |
| orders              |
| products            |
| products_on_hand    |
+---------------------+
6 rows in set (0.00 sec)
```

Use the MySQL command line client to explore the database and view the pre-loaded data in the database.

```sh
mysql> SELECT * FROM customers;
+------+------------+-----------+-----------------------+
| id   | first_name | last_name | email                 |
+------+------------+-----------+-----------------------+
| 1001 | Sally      | Thomas    | sally.thomas@acme.com |
| 1002 | George     | Bailey    | gbailey@foobar.com    |
| 1003 | Edward     | Walker    | ed@walker.com         |
| 1004 | Anne       | Kretchmar | annek@noanswer.org    |
+------+------------+-----------+-----------------------+
4 rows in set (0.00 sec)
```

### Starting Kafka Connect

After starting MySQL and connecting to the inventory database with the MySQL command line client, you start the Kafka Connect service. This service exposes a REST API to manage the Debezium MySQL connector.

`docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:1.0`

Verify that Kafka Connect started and is ready to accept connections.:

```sh
...
2020-02-06 15:48:33,939 INFO   ||  Kafka version: 2.4.0   [org.apache.kafka.common.utils.AppInfoParser]
...
2020-02-06 15:48:34,485 INFO   ||  [Worker clientId=connect-1, groupId=1] Starting connectors and tasks using config offset -1   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2020-02-06 15:48:34,485 INFO   ||  [Worker clientId=connect-1, groupId=1] Finished starting connectors and tasks   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
```

Use the Kafka Connect REST API to check the status of the Kafka Connect service

check the status of the Kafka Connect service:
`curl -H "Accept:application/json" localhost:8083/`

Check the list of connectors registered with Kafka Connect:
`curl -H "Accept:application/json" localhost:8083/connectors/`

gives output [] as no connectors are currently registered with kafka

### Deploying the MySQL connector

1.  **Registering a connector to monitor the inventory database**

By registering the Debezium MySQL connector, the connector will start monitoring the MySQL database server’s binlog. The binlog records all of the database’s transactions (such as changes to individual rows and changes to the schemas). When a row in the database changes, Debezium generates a change event.

register connector with kafka:

`curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'`

verify:

`curl -H "Accept:application/json" localhost:8083/connectors/`

output: ["inventory-connector"]`

Review the connector’s tasks:

`curl -i -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector`

```json
HTTP/1.1 200 OK
Date: Thu, 06 Feb 2020 22:12:03 GMT
Content-Type: application/json
Content-Length: 531
Server: Jetty(9.4.20.v20190813)

{
  "name": "inventory-connector",
  ...
  "tasks": [
    {
      "connector": "inventory-connector",  
      "task": 0
    }
  ]
}
```

The connector is running a single task (task 0) to do its work. The connector only supports a single task, because MySQL records all of its activities in one sequential binlog. This means the connector only needs one reader to get a consistent, ordered view of all of the events.

2. **Watching the connector start up**

When you register a connector, it generates a large amount of log output in the Kafka Connect container. By reviewing this output, you can better understand the process that the connector goes through from the time it is created until it begins reading the MySQL server’s binlog.

check https://debezium.io/documentation/reference/tutorial.html#watching-connector-start-up for more.


### Viewing change events

`docker run -it --rm --name watcher --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:1.0 watch-topic -a -k dbserver1.inventory.customers`

#### Viewing a create event
By viewing the `dbserver1.inventory.customers` topic, you can see how the MySQL connector captured create events in the inventory database. In this case, the create events capture new customers being added to the database.

Open a new terminal, and use it to start the watch-topic utility to watch the dbserver1.inventory.customers topic from the beginning of the topic.

`docker run -it --rm --name watcher --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:1.0 watch-topic -a -k dbserver1.inventory.customers`

-a
Watches all events since the topic was created.

-k
Specifies that the output should include the event’s key. In this case, this contains the row’s primary key.


output:
```sh
Using ZOOKEEPER_CONNECT=172.17.0.2:2181
Using KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.0.7:9092
Using KAFKA_BROKER=172.17.0.3:9092
Contents of topic dbserver1.inventory.customers:
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}
...
```

Here are the details of the key of the last event (formatted for readability):
```json
{
  "schema": {
    "type": "struct",
    "name": "dbserver1.inventory.customers.Key"
    "optional": false,
    "fields": [
      {
        "field": "id",
        "type": "int32",
        "optional": false
      }
    ]
  },
  "payload": {
    "before": null,
    "after": {
      "id": 1004,
      "first_name": "Anne",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "source": {
      "version": "1.1.0.Beta2",
      "name": "dbserver1",
      "server_id": 0,
      "ts_sec": 0,
      "gtid": null,
      "file": "mysql-bin.000003",
      "pos": 154,
      "row": 0,
      "snapshot": true,
      "thread": null,
      "db": "inventory",
      "table": "customers"
    },
    "op": "c",
    "ts_ms": 1486500577691
  }
}
```

The event has two parts: a `schema` and a `payload`. The schema contains a Kafka Connect schema describing what is in the payload. In this case, the payload is a struct named dbserver1.inventory.customers.Key that is not optional and has one required field (id of type int32).

#### Updating the database and viewing the update event

```sql
mysql> UPDATE customers SET first_name='Anne Marie' WHERE id=1004;
Query OK, 1 row affected (0.05 sec)
Rows matched: 1  Changed: 1  Warnings: 0


mysql> SELECT * FROM customers;
+------+------------+-----------+-----------------------+
| id   | first_name | last_name | email                 |
+------+------------+-----------+-----------------------+
| 1001 | Sally      | Thomas    | sally.thomas@acme.com |
| 1002 | George     | Bailey    | gbailey@foobar.com    |
| 1003 | Edward     | Walker    | ed@walker.com         |
| 1004 | Anne Marie | Kretchmar | annek@noanswer.org    |
+------+------------+-----------+-----------------------+
4 rows in set (0.00 sec)
```

Switch to the terminal running watch-topic to see a new fifth event.

key of new event:

```json
{
    "schema": {
      "type": "struct",
      "name": "dbserver1.inventory.customers.Key"
      "optional": false,
      "fields": [
        {
          "field": "id",
          "type": "int32",
          "optional": false
        }
      ]
    },
    "payload": {
      "id": 1004
    }
  }
```

Here is that new event’s value:

```json
{
  "schema": {...},
  "payload": {
    "before": {  
      "id": 1004,
      "first_name": "Anne",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "after": {  
      "id": 1004,
      "first_name": "Anne Marie",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "source": {  
      "name": "1.1.0.Beta2",
      "name": "dbserver1",
      "server_id": 223344,
      "ts_sec": 1486501486,
      "gtid": null,
      "file": "mysql-bin.000003",
      "pos": 364,
      "row": 0,
      "snapshot": null,
      "thread": 3,
      "db": "inventory",
      "table": "customers"
    },
    "op": "u",  
    "ts_ms": 1486501486308  
  }
}
```

#### Deleting a record in the database and viewing the delete event

```sql
mysql> DELETE FROM addresses WHERE customer_id=1004;

mysql> DELETE FROM customers WHERE id=1004;
Query OK, 1 row affected (0.00 sec)
```

Switch to the terminal running watch-topic to see two new events:

key for the first new event:

```json
{
  "schema": {
    "type": "struct",
    "name": "dbserver1.inventory.customers.Key"
    "optional": false,
    "fields": [
      {
        "field": "id",
        "type": "int32",
        "optional": false
      }
    ]
  },
  "payload": {
    "id": 1004
  }
}
```

Here is the value of the first new event:

```json
{
  "schema": {...},
  "payload": {
    "before": {  
      "id": 1004,
      "first_name": "Anne Marie",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "after": null,  
    "source": {  
      "name": "1.1.0.Beta2",
      "name": "dbserver1",
      "server_id": 223344,
      "ts_sec": 1486501558,
      "gtid": null,
      "file": "mysql-bin.000003",
      "pos": 725,
      "row": 0,
      "snapshot": null,
      "thread": 3,
      "db": "inventory",
      "table": "customers"
    },
    "op": "d",  
    "ts_ms": 1486501558315  
  }
}
```

### CLeanup

`docker stop mysqlterm watcher connect mysql kafka zookeeper`

