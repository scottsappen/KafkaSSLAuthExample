# REST Proxy

Let's use the Confluent Kafka Rest security plugin to validate the clients accessing the rest proxy and use those principals to authenticate at the broker level with Kafka ACLs.

Currently, only SSL principal propagation is supported.

Clients (SSL) < -- > (SSL) REST Proxy (SASL) < -- > (SASL) Broker

First let's move the Rest Proxy we previously configured to communicate to the brokers over to using SASL instead of SSL.

```
sudo vi ../etc/kafka-rest/kafka-rest.properties

bootstrap.servers=SASL_SSL://<your kafka server>:9094

client.bootstrap.servers=<your kafka server>:9094
client.sasl.mechanism=GSSAPI
client.sasl.kerberos.service.name=ubuntu
client.security.protocol=SASL_SSL
```

Create the REST Proxy JaaS file. In the real world you will use other "real" keytabs and principals specifically created for the Rest Proxy.

```
sudo vi /tmp/kafka_rest_jaas.conf

KafkaClient {
 com.sun.security.auth.module.Krb5LoginModule required
 useKeyTab=true
 storeKey=true
 keyTab="/tmp/kafka.service.keytab"
 principal="ubuntu/<your kafka server>@KAFKA.SECURE";
 };
```

Start REST Proxy

```
KAFKAREST_OPTS="-Djava.security.auth.login.config=/tmp/kafka_rest_jaas.conf" ./confluent start kafka-rest
```

Validate you can still connect successfully

```
curl -vk --key restproxy.key --cert restproxy.certificate.pem --cacert restproxy.certificate.pem "https://<your kafka server>:8888/topics"
```

Now let's enable the REST Proxy Confluent security plugin

```
sudo vi ../etc/kafka-rest/kafka-rest.properties

kafka.rest.resource.extension.class=io.confluent.kafkarest.security.KafkaRestSecurityResourceExtension
confluent.rest.auth.propagate.method=SSL
```

Start REST Proxy

```
KAFKAREST_OPTS="-Djava.security.auth.login.config=/tmp/kafka_rest_jaas.conf" ./confluent start kafka-rest
```

Validate you can still connect successfully.

```
curl -vk --key restproxy.key --cert restproxy.certificate.pem --cacert restproxy.certificate.pem "https://<your kafka server>:8888/topics"
```

Now you can go create your ACLs to configure the levels of authorization you need.

If you get a 401 return error, you have a problem with your principal. Go back and make sure your principals are available on the broker and REST Proxy.
