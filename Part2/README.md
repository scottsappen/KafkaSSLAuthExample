# Part 2

**Part 2
Setup plaintext un-secure for a quick test**

Let's do a simple test using a non-secure setup, simple Java consumer and producer.
```
vi /etc/kafka/server.properties
```

Uncomment this line and insert your public DNS hostname:
```
advertised.listeners=PLAINTEXT://<your kafka public server>:9092
```

Change this line and insert your public DNS hostname as well:
```
zookeeper.connect=<your kafka public server>:2181
```

Restart Kafka and again verify
```
./confluent start
./confluent stop
```

Make sure you clean up any errors and stop here if you are not successful before continuing.
```
./kafka-topics --zookeeper <your kafka public server>:2181 --list
```

Let's create a topic and produce and consume some data.
```
./kafka-topics --zookeeper <your zookeeper server>:2181 --create --topic kafka-security-topic --replication-factor 1 --partitions 2

./kafka-console-producer --broker-list <your kafka server>:9092 --topic kafka-security-topic

./kafka-console-consumer --bootstrap-server <your kafka server>:9092 --topic kafka-security-topic
```

Success!

Now let's move onto security now that basic un-secure functionality is confirmed.
