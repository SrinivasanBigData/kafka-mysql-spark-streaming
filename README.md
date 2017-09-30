# kafka-mysql-spark-streaming
This repository will go a bit deeper into Kafka. On one side, I have MySQL data. On the other side, I have Spark. How can I stream my MySQL database into Spark with Scala ? Well, Kafka seems to be a pretty good solution.

## Our environment

To be able to use Kafka to stream MySQL data to Spark, we will need a couple of services. Basically, we will need a kafka and a zookeeper services. Of course, we will also need a Spark, and a MySQL service. The thing is that will also need a Maven service. I will explain a bit later why.

In the end, our docker-compose file looks like this:

    version: '2'
    services:

      zookeeper:
        image: wurstmeister/zookeeper
        container_name: zookeeper
        ports:
          - "2181:2181"

      kafka:
        image: wurstmeister/kafka
        container_name: kafka
        ports:
          - "9092:9092"
        environment:
          KAFKA_ADVERTISED_HOST_NAME: 172.21.0.1
          KAFKA_CREATE_TOPICS: "connect-test:1:1"
          KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
        links:
          - zookeeper
          - mysql
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - content:/data

      mysql:
        image: mysql
        ports:
          - 3306:3306
        container_name: mysql
        environment:
          MYSQL_ROOT_PASSWORD: rootPassword
          MYSQL_DATABASE: product

      spark:
        image: gettyimages/spark
        container_name: spark
        links:
          - kafka
          - zookeeper
        volumes:
          - content:/data

      maven:
        image: maven
        container_name: maven
        command: bash
        stdin_open: true
        tty: true
        volumes:
          - content:/data

    volumes:
     content:
       driver: local
       
Ok, what is going on there deserves some explanation.

Our Kafka service is connected to my mysql and zookeeper services. It needs zookeeper to work. Besides, Kafka will need to access the MySQL database. When the kafka service starts, it will automatically create the connect-test topic.

The mysql service is defined with its root password and default database (that will also automatically be created at startup). So far, so good…

The spark service is linked to zookeeper. It needs the coordinator to fetch the stream. The rest of the configuration is quite standard.

Finally, we have a maven service, also standard configuration.

## Setup IP of your Kafka container

Before starting your kafka connect, you need to set the Kafka container’s IP adress into the KAFKA_ADVERTISED_HOSTNAME environment variable. You shall first connect to your Kafka container :

        $ docker exec -it kafka /bin/sh
        
Then, fetch the container’s IP adress, and update the KAFKA_ADVERTISED_HOSTNAME environment variable :

        / # echo $KAFKA_ADVERTISED_HOST_NAME
        127.0.0.1
        / # /sbin/ip route|awk '/default/ { print $3 }'
        172.21.0.1
        / # export KAFKA_ADVERTISED_HOST_NAME=172.21.0.1
        / # echo $KAFKA_ADVERTISED_HOST_NAME
        172.21.0.1

## Build Kafka MySQL connectors

At first, we will connect to our mysql container in order to build the database schema. We will create one table, and insert 2 data. It goes like this :

    $ docker exec -it mysql /bin/bash
    root@9b7e6489fb57:/# mysql -h localhost -u root -prootPassword
    
After connecting to the MySQL client, let’s set up our schema :

    mysql > use product;
    mysql > CREATE TABLE IF NOT EXISTS accounts (id int(5) NOT NULL AUTO_INCREMENT, location varchar(250) DEFAULT NULL, PRIMARY KEY(id));
    
We are done with MySQL. We now have to build the files that will allow Kafka to communicate with MySQL. Actually, Kafka needs 2 things :

- A MySQL driver to connect and query some MySQL
- A connector to allow Kafka to use MySQL as a source through kafka-connect

For the driver, things are pretty simple. You will just need to download the file. Connect to your maven container, and launch the following commands :

    $ docker exec -it maven /bin/bash
    root@e726499abc3a:/# cd /data/
    root@e726499abc3a:/data# wget http://central.maven.org/maven2/mysql/mysql-connector-java/5.1.6/mysql-connector-java-5.1.6.jar
    
It is important that the file you download is located in the /data folder. It is important because it is the folder that will be shared through all our containers (as you can see in the docker-compose file).

If we were to use Confluent platform, we could just fetch the connector, everything would be plugged, and the job would be done. In this tutorial, we choose to play it old school. We deliberately wish not to use a specific platform to make this all work. What this means is that we will need to rebuild the MySQL connector for our Kafka container. We will then fetch the source of the connector, and build it. Connect to your maven container, and launch the following commands :

    $ docker exec -it maven /bin/bash
    root@e726499abc3a:/# cd /data/
    root@e726499abc3a:/# git clone https://github.com/confluentinc/kafka-connect-jdbc
    root@e726499abc3a:/# cd kafka-connect-jdbc/
    root@e726499abc3a:/# git checkout tags/v3.2.1
    root@e726499abc3a:/# mvn package
    
An important thing to notice is that we would always work from a tagged version of the repository. This is the guaranty that we clone an “official” version of the code.

The build should take a couple of minutes. There will be much downloads and tests. After this, a new folder will be created in _/data/kafka-connect-jdbc/_, called target. The built connector for MySQL is in this folder. You just need to do this :

    $ docker exec -it maven /bin/bash
    root@e726499abc3a:/# mv /data/kafka-connect-jdbc/target/kafka-connect-jdbc-3.2.1.jar /data
    
## Set everything ready

It is time now to connect to our kafka container, and plug the connectors in kafka libraries :

    $ docker exec -it kafka /bin/sh
    / # cp /data/*.jar /opt/kafka/libs
    / # cd /data
    
We are now going to create the properties files that we will need to launch the connect. First, create in /data, a file called _standalone.properties_ :

    bootstrap.servers=localhost:9092

    key.converter=org.apache.kafka.connect.json.JsonConverter
    value.converter=org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable=true
    value.converter.schemas.enable=true

    internal.key.converter=org.apache.kafka.connect.json.JsonConverter
    internal.value.converter=org.apache.kafka.connect.json.JsonConverter
    internal.key.converter.schemas.enable=false
    internal.value.converter.schemas.enable=false

    offset.storage.file.filename=/tmp/connect.offsets
    offset.flush.interval.ms=10000
    
We will also create in /data,  a file called _mysql.properties_ :

    name=mysql-source
    connector.class=io.confluent.connect.jdbc.JdbcSourceConnector
    tasks.max=10
    connection.url=jdbc:mysql://mysql:3306/product
    connection.password = rootPassword
    connection.user = root
    table.whitelist=accounts

    mode=incrementing
    incrementing.column.name=id

    topic.prefix=db
    
You will notice that we wrote in this file the credentials that will allow us to connect to the MySQL table. the connection url is mysql, which is the name of the mysql container.

The tricky thing here is the topic.prefix parameter. When the source (mysql connector) launches, kafka will automatically create a new topic. This topic will have the name of the table (accounts in this tutorial), and a prefix which is the topic.prefix value (db here).

We can now launch our kafka connect :

    / # cd /opt/kafka
    / # bin/connect-standalone.sh /data/standalone.properties /data/mysql.properties
    
You will see a bunch of information printed in the console. You can list the topics registered. In another console, connect to your kafka container, and type this:

    $ docker exec -it kafka /bin/sh
    / # kafka-topics.sh --list --zookeeper zookeeper:2181
    connect-test
    dbaccounts
    
As you can see, a new topic was created!

## Stream this all !

You can now connect to your spark container, and type the following commands:

    $ docker exec -it spark /bin/sh
    root@e7452c65bdc4:/usr/spark-2.1.0# cd bin
    root@e7452c65bdc4:/usr/spark-2.1.0# ./spark-shell --packages org.apache.spark:spark-streaming-kafka-0-8_2.11:2.1.0
    
To stream data, we need to launch the kafka streaming package with spark-shell.

You can now type in the scala code that will launch the streaming and connect through Kafka to the MySLQ database :

```java
import org.apache.spark.SparkConf
import org.apache.spark.streaming.StreamingContext
import org.apache.spark.streaming.Seconds
val ssc = new StreamingContext(sc,Seconds(10))
import org.apache.spark.streaming.kafka.KafkaUtils
val kafkaStream = KafkaUtils.createStream(ssc, "zookeeper:2181","spark-streaming-consumer-group", Map("dbaccounts" -&gt; 1))
kafkaStream.print()
ssc.start()
```
The streaming is launched. The last thing to do is insert data into your MySQL database. In the previous mysql container console, type in the following :

```sql
INSERT INTO accounts VALUES (1, "John"), (2, "Sandy");
```

Go back to the Spark container console, and look carefully :

    -------------------------------------------
    Time: XXXXXXXXXXXX ms
    -------------------------------------------
    ({"schema":null,"payload":null},{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"location"}],"optional":false,"name":"accounts"},"payload":{"id":1,"location":"John"}})
    ({"schema":null,"payload":null},{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"location"}],"optional":false,"name":"accounts"},"payload":{"id":2,"location":"Sandy"}})
    
Now we could setup a stream between a MySQL database and Spark through Kafka, the game is yours. I guess it will start by editing your scala code, replacing the kafkaStream.print() by some processing.
