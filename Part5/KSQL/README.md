# KSQL

This is a section to show you how to enable the KSQL endpoint to use SSL.

There are 2 parts to this:

- KSQL configuration to the brokers
- KSQL's REST endpoint

The first part is required if you want KSQL to use SSL in communication with the brokers.

The second part is required if you want to secure the REST endpoint with KSQL.

In this example, we will leave HTTP enabled on the default port and enable SSL on a new port. In practice, you can disable HTTP and inter.instance.protocol=https and require 2-way SSL auth. Remember to take care of other components that might use http to KSQL as a default (e.g. Control Center).

<br/>

**Part 1<br/>
KSQL configuration to the brokers**

As a sanity check, verify you can access the KSQL on PLAINTEXT.

```
curl -sX GET "http://<your kafka server>:8088/info" | jq '.'
```

Now let's configure the connection for SSL.

```
sudo vi ../etc/ksql/ksql-server.properties

# Secure settings
bootstrap.servers=<your kafka server>:9093

security.protocol=SSL
ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
ssl.truststore.password=confluent
ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
ssl.keystore.password=confluent
ssl.key.password=confluent
```

Restart KSQL

```
./bin/confluent stop ksql-server

./bin/confluent start ksql-server
```

Verify all is ok at the REST endpoint as well as the KSQL terminal.

```
curl -sX GET "http://<your kafka server>:8088/info" | jq '.'

./ksql
```

<br/>

**Part 2<br/>
KSQL's REST endpoint**

Now let's configure the SSL REST endpoint.

```
sudo vi ../etc/ksql/ksql-server.properties

listeners=http://<your kafka server>:8088,https://<your kafka server>:8891
```

Restart KSQL

```
./bin/confluent stop ksql-server

./bin/confluent start ksql-server
```

This should still work because you still allow http on the default port.

```
curl -sX GET "http://<your kafka server>:8088/info" | jq '.'
```

And this should now work.

```
curl -vk --key restproxy.key --cert restproxy.certificate.pem --cacert restproxy.certificate.pem -sX GET "https://<your kafka server>:8891/info" | jq '.'
```

That's it!
