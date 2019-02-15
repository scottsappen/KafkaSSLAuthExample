# REST Proxy

Once you get SSL client auth enabled on the brokers, the REST Proxy will continue to work as is on PLAINTEXT port. However, you may want to move it over to the SSL port.

There are 2 parts to this:

- Configuring SSL from REST Proxy to the brokers
- Configuring clients to use SSL to the REST Proxy

<br/>

**Part 1<br/>
Configuring SSL from the REST Proxy to the brokers**

First, verify again it works without SSL just as a sanity check.

Again, the REST Proxy is not to be confused with the REST interface that is available with Connect. The latter runs by default on port 8083 whereas the REST Proxy default port is 8082.

This should work without issue.

```
sudo vi ../etc/kafka-rest/kafka-rest.properties

bootstrap.servers=PLAINTEXT://<your kafka server>:9092
listeners=http://<your kafka server>:8888
```

```
curl http://<your kafka server>:8888/topics | jq
```

Now, let's get REST proxy communicating to the brokers over SSL but leave the clients connecting to rest proxy using non-SSL.

```
sudo vi ../etc/kafka-rest/kafka-rest.properties

bootstrap.servers=SSL://<your kafka server>:9093

#SSL encryption between REST proxy and the Kafka cluster
client.bootstrap.servers=<your kafka server>:9093
client.security.protocol=SSL
client.ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.truststore.jks
client.ssl.truststore.password=confluent
client.ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
client.ssl.keystore.password=confluent
ssl.key.password=confluent
```

Restart REST Proxy

```
./bin/confluent stop kafka-rest

./bin/confluent start kafka-rest
```

Test it out!

```
curl http://<your kafka server>:8888/topics | jq
```

Success!

<br/>

**Part 2<br/>
Configuring clients to use SSL to the REST Proxy**

Now let's get the clients talk talk to REST proxy using SSL (or HTTPS).

```
sudo vi ../etc/kafka-rest/kafka-rest.properties

listeners=https://<your kafka server>:8888

#HTTPS between REST clients and the REST Proxy
ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
ssl.keystore.password=confluent
ssl.key.password=confluent
ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.truststore.jks
ssl.truststore.password=confluent
ssl.protocol=TLS
ssl.client.auth=true
```

Restart REST Proxy
```
./bin/confluent stop kafka-rest

./bin/confluent start kafka-rest
```

Create the requisite files for SSL communication.
```
keytool -export -alias mykey -file restproxy.der -keystore kafka.server.keystore.jks -storepass confluent

openssl x509 -inform der -in restproxy.der -out restproxy.certificate.pem

keytool -importkeystore -srckeystore kafka.server.keystore.jks -srcstoretype jks -destkeystore restproxy.keystore.p12 -deststoretype PKCS12 -deststorepass confluent -srcstorepass confluent -noprompt

openssl pkcs12 -in restproxy.keystore.p12 -nodes -nocerts -out restproxy.key -passin pass:confluent
```

If you get errors about the key not matching the cert, double check you are using the right combination of parameters in your commands above. These are just samples.

Verify SSL is working for the connection.

```
curl -vk --key restproxy.key --cert restproxy.certificate.pem --cacert restproxy.certificate.pem "https://<your kafka server>:8888/topics"
```

Success!

Note if you have issues (e.g. handshakes), you may run into a number of problems around installed curl version, supported crypto, etc. You may be best served trying this from a few different types of servers to narrow down issues.
