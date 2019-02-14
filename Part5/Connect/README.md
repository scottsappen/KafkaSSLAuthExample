# Connect

Once you get SSL client auth enabled on the brokers, Connect will continue to work as is on PLAINTEXT port. However, you may want to move it over to the SSL port.

There are 2 parts to this:

- Connect configuration to the brokers
- Connect's REST endpoint

The first part is required if you want Connect to use SSL in communication with the brokers.

The second part is required if you want to secure the REST endpoint with Connect. This is a REST endpoint for Connect, not to be confused with the RESTProxy.

<br/>
**Part 1
Connect configuration to the brokers**

These top-level settings are used by the Connect worker for group coordination and to read and write to the internal topics which are used to track the cluster's state (e.g. configs and offsets).

```
sudo vi ../etc/schema-registry/connect-avro-distributed.properties

# Secure settings
bootstrap.servers=<your kafka server>:9093
security.protocol=SSL
ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
ssl.truststore.password=confluent
ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
ssl.keystore.password=confluent
```

Connect workers manage the producers used by source connectors and the consumers used by sink connectors. So, for the connectors to leverage security, you also have to override the default producer/consumer configuration that the worker uses. Depending on whether the connector is a source or sink connector:

```
producer.bootstrap.servers=<your kafka server>:9093
producer.security.protocol=SSL
producer.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
producer.ssl.truststore.password=confluent
producer.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
producer.ssl.keystore.password=confluent

consumer.bootstrap.servers=<your kafka server>:9093
consumer.security.protocol=SSL
consumer.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
consumer.ssl.truststore.password=confluent
consumer.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
consumer.ssl.keystore.password=confluent
```

Restart Connect

```
./bin/confluent stop connect

./bin/confluent start connect
```

<br/>
**Part 1
Connect's REST endpoint**

First, again for a quick sanity check make sure you can query using non-SSL. This should work as you haven't changed any settings yet.

```
curl http://<your kafka server>:8083/connectors | jq
```

Now let's change over to SSL for the REST endpoint.

```
vi /etc/schema-registry/connect-avro-distributed.properties

listeners=https://<your kafka server>:8889
ssl.client.auth=true
```

Restart Connect

```
./bin/confluent stop connect

./bin/confluent start connect
```

Run a test on port 8889 and make sure you can connect!
```
curl -vk --key restproxy.key --cert restproxy.certificate.pem --cacert restproxy.certificate.pem "https://<your kafka server>:8889/connectors"
```

If you need help creating the appropriate key and pem files, see the section under RESTProxy labeled "Create the requisite files for SSL communication." It's a similar process.

Success!
