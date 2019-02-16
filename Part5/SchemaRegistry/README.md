# Schema Registry

Once you get SSL client auth enabled on the brokers, Schema Registry will continue to work as is on PLAINTEXT port. However, you may want to move it over to the SSL port.

Schema Registry uses Kafka to persist schemas, and so it acts as a client to write data to the Kafka cluster. Therefore, if the Kafka brokers are configured for security, you can (and should) configure Schema Registry to use security.

There are 2 parts to this:

- Connect configuration to the brokers
- Connect's REST endpoint

The first part is required if you want Schema Registry to use SSL in communication with the brokers.

The second part is required if you want to secure the REST endpoint with Schema Registry.

In this example, we will leave HTTP enabled on the default port and enable SSL on a new port. In practice, you can disable HTTP and inter.instance.protocol=https and require 2-way SSL auth. Remember to take care of other components that might use http to Schema Registry as a default (e.g. Control Center, Connect).

<br/>

**Part 1<br/>
Schema Registry configuration to the brokers**

As a sanity check, verify you can access the registry on PLAINTEXT.

```
curl <your kafka server>:8081/subjects
```

Now let's configure the connection to the brokers.

```
sudo vi ../etc/schema-registry/schema-registry.properties

# Secure settings
kafkastore.bootstrap.servers=SSL://<your kafka server>:9093
kafkastore.security.protocol=SSL
kafkastore.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.truststore.jks
kafkastore.ssl.truststore.password=confluent
kafkastore.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
kafkastore.ssl.keystore.password=confluent
```

Restart Confluent

```
./bin/confluent stop

./bin/confluent start
```

Verify all is ok. This should still work.

```
curl <your kafka server>:8081/subjects
```

<br/>

**Part 2<br/>
Connect's REST endpoint**

Now let's configure the SSL REST endpoint.

```
sudo vi ../etc/schema-registry/schema-registry.properties

listeners=http://<your kafka server>:8081,https://<your kafka server>:8890
```

Restart Confluent

```
./bin/confluent stop

./bin/confluent start
```

This should still work because you still allow http on the default port.

```
curl <your kafka server>:8081/subjects
```

And this should now work.

```
curl -vk --key restproxy.key --cert restproxy.certificate.pem --cacert restproxy.certificate.pem "https://<your kafka server>:8890/subjects"
```

If you need help creating the appropriate key and pem files, see the section under RESTProxy labeled "Create the requisite files for SSL communication." It's a similar process.

That's it!
