# Control Center

Once you get SSL client auth enabled on the brokers, you will need to enable Control Center SSL auth as well.

This is required because Confluent Control Center uses Kafka Streams as a state store, so if all the Kafka brokers in the cluster backing Control Center are secured, then the Control Center application also needs to be secured.

There are 2 parts to this:

- Confluent Metrics Reporter: system health
- Confluent Monitoring Interceptors: streams monitoring

The first part is required if you want to see the system health in Control Center and otherwise make it useful. Without it, Control Center will not do you much good.

The second part however is optional and only needed if you want to use the Data Streams capability. This is highly recommended because you can see real-time consumer/producer health.

<br/>
**Part 1
Confluent Metrics Reporter**

Let's get the basics completed first.

```
sudo vi /etc/kafka/server.properties

metric.reporters=io.confluent.metrics.reporter.ConfluentMetricsReporter
confluent.metrics.reporter.bootstrap.servers=<your kafka server>:9093
```

#Control Center security settings
```
confluent.metrics.reporter.security.protocol=SSL
confluent.metrics.reporter.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
confluent.metrics.reporter.ssl.truststore.password=confluent
confluent.metrics.reporter.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.keystore.jks
confluent.metrics.reporter.ssl.keystore.password=confluent
confluent.metrics.reporter.ssl.key.password=confluent
```

Restart Confluent

```
./bin/confluent start
```

Verify Control Center is now working with SSL enabled.

You can use a regular console consumer to make sure data is flowing into the appropriate topic as a simple examination.

```
./kafka-console-consumer --bootstrap-server <your kafka server>:9092 --topic _confluent-metrics --from-beginning
```

Also, visit your browser and make sure you see system health is ok. You can use the following API endpoint to show the latest time reported by Control Center. If this returns {} then something else is wrong and you should troubleshoot further.

```
http://<your kafka server>:9021/2.0/metrics/1nsiSOthSuWWWhPmxuQ2Qw/maxtime

{"timestamp":1550072274540}
```

If you are curious what that translates to, you can use free utilities such as https://currentmillis.com/.

Note: if you are running Confluent CLI, the bin/confluent start will sometimes overwrite your settings with unintended results.

Example: the confluent.metrics.reporter.bootstrap.servers=<your kafka server>:9093 is overwritten. You will notice in your /tmp/confluent.<logdir>/kafka/server.log that it will try to error out connecting to localhost:9093. That is because confluent start script appends it localhost even if you have this defined. To resolve,

```
sudo vi bin/Confluent
```
Comment the section that writes in localhost.

```
#echo "confluent.metrics.reporter.bootstrap.servers=localhost:${kafka_port}" \
#    >> "${service_dir}/${service}.properties"
```

Restart Confluent

```
./bin/confluent start
```

<br/>
**Part 1
Data Streams**

Let's work on the optional data streams part and do some simple testing. To get the Data Streams working, you must configure your consumer and producer apps to use the interceptor jars.

```
sudo vi /etc/kafka/server.properties

metric.reporters=io.confluent.metrics.reporter.ConfluentMetricsReporter
confluent.metrics.reporter.bootstrap.servers=<your kafka server>:9093
```

#Control Center security settings
```
confluent.controlcenter.streams.security.protocol=SSL
confluent.controlcenter.streams.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
confluent.controlcenter.streams.ssl.truststore.password=confluent
confluent.controlcenter.streams.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.keystore.jks
confluent.controlcenter.streams.ssl.keystore.password=confluent
confluent.controlcenter.streams.ssl.key.password=confluent
```

Restart Confluent

```
./bin/confluent start
```

Let's create a sample topic to work with.

```
./kafka-topics --zookeeper <your zookeeper server>:2181 --create --topic kafka-security-topic --replication-factor 1 --partitions 2
```

Let's setup a producer and consumer to work with the interceptors.

```
sudo vi ../ssl/producer.properties

interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor
confluent.monitoring.interceptor.bootstrap.servers=<your kafka server>:9093
confluent.monitoring.interceptor.security.protocol=SSL
confluent.monitoring.interceptor.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
confluent.monitoring.interceptor.ssl.truststore.password=confluent
confluent.monitoring.interceptor.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.keystore.jks
confluent.monitoring.interceptor.ssl.keystore.password=confluent
confluent.monitoring.interceptor.ssl.key.password=confluent
security.protocol=SSL
ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
ssl.truststore.password=confluent
ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.keystore.jks
ssl.keystore.password=confluent
ssl.key.password=confluent
```

```
sudo vi ../ssl/consumer.properties

interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor
confluent.monitoring.interceptor.bootstrap.servers=<your kafka server>:9093
group.id=c3_example_consumer_group
confluent.monitoring.interceptor.security.protocol=SSL
confluent.monitoring.interceptor.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
confluent.monitoring.interceptor.ssl.truststore.password=confluent
confluent.monitoring.interceptor.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.keystore.jks
confluent.monitoring.interceptor.ssl.keystore.password=confluent
confluent.monitoring.interceptor.ssl.key.password=confluent
security.protocol=SSL
ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
ssl.truststore.password=confluent
ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.keystore.jks
ssl.keystore.password=confluent
ssl.key.password=confluent
```

The "confluent.monitoring" parts are required for data streams whereas the ssl parts are just because we are connecting with a secure client on 9093.

Now let's run a simple producer and consumer. Notice in this producer example, we are using really small message sizes. That is only because the tiny "micro" box this is running on is extremely limited in terms of available space.

```
export CLASSPATH=/home/ubuntu/confluent-5.1.1/share/java/monitoring-interceptors/monitoring-interceptors-5.1.1.jar

./kafka-producer-perf-test --topic kafka-security-topic --num-records 10000000 --record-size 1 --throughput 100 --producer-props bootstrap.servers=<your kafka server>:9093 interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor acks=all --producer.config ../ssl/producer.properties

export CLASSPATH=/home/ubuntu/confluent-5.1.1/share/java/monitoring-interceptors/monitoring-interceptors-5.1.1.jar

./kafka-console-consumer --bootstrap-server <your kafka server>:9093 --topic kafka-security-topic --consumer.config ../ssl/consumer.properties
```

Note: if you see WARN messages about supplied configs that are not appropriate, those can be safely ignored (needs to be cleaned up in future releases). These are required values to get working.

```
[2019-02-13 18:51:33,365] WARN The configuration 'confluent.monitoring.interceptor.security.protocol' was supplied but isn't a known config.
```

One thing to note, you do not have to use the interceptors. This command still works fine using SSL without the data streams interceptor capability.
```
./kafka-console-consumer --bootstrap-server <your kafka server>:9093 --topic kafka-security-topic --consumer.config ../ssl/client-ssl-auth.properties --from-beginning
```

Now one thing to mention, if you were to try and use the interceptors with --from-beginning and expected to see something without running a producer you would be mistaken. You need to run the producer to see data. That is because you have the interceptors configured to be used.
```
./kafka-console-consumer --bootstrap-server <your kafka server>:9093 --topic kafka-security-topic --consumer.config ../ssl/consumer.properties --from-beginning
```
