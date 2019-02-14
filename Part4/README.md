# Part 4

**Part 4
Configure and test 2-way SSL authentication**

This is 2-way where the broker will validate the client's identify. This requires us to generate certs for our clients, but also reconfigure the brokers to require SSL authentication.

```
keytool -genkey -keystore kafka.client.keystore.jks -validity 365 -storepass confluent -keypass confluent -dname "CN=mylocalclient" -alias my-local-client -storetype pkcs12

keytool -keystore kafka.client.keystore.jks -certreq -file client-cert-sign-request -alias my-local-client -storepass confluent -keypass confluent
```

Now you need to get your client-cert-sign-request signed by the CA. In the real world you could email this to the CA and have then sign it. But in our case, we will upload to the kafka server and sign it there.

For your laptop, it may be something akin to:
```
scp -i ~<your pem file>.pem client-cert-sign-request ubuntu@<your kafka public server>:/tmp/
```

Now on the kafka server, sign your cert request

```
openssl x509 -req -CA ca-cert -CAkey ca-key -in /tmp/client-cert-sign-request -out /tmp/client-cert-signed -days 365 -CAcreateserial -passin pass:confluent
```

Now back on our local laptop, download that signed cert by the CA and the CA cert itself and import both into the client local keystore.

```
scp -i ~<your pem file>.pem ubuntu@<your kafka public server>:/tmp/client-cert-signed .

keytool -keystore kafka.client.keystore.jks -alias CARoot -import -file ca-cert -storepass confluent -keypass confluent -noprompt

keytool -keystore kafka.client.keystore.jks -alias my-local-client -import -file client-cert-signed -storepass confluent -keypass confluent -noprompt
```

Now that we are finished with the certs, we must now configure our brokers to enforce SSL authentication.

```
vi /etc/kafka/server.properties

ssl.client.auth=required
```

On the client machine, you would now create a client-ssl properties file to connect with passing in the appropriate keystore and truststore information.

```
vi client-ssl-auth.properties

security.protocol=SSL
ssl.truststore.location=<location of your client truststore file>/kafka.client.truststore.jks
ssl.truststore.password=confluent
ssl.keystore.location=<location of your client keystore file>/kafka.client.keystore.jks
ssl.keystore.password=confluent
ssl.key.password=confluent
```

Now test.

This will work still.

```
../bin/kafka-console-consumer --bootstrap-server <your kafka public server>:9092 --topic kafka-security-topic --from-beginning
```

But notice this will not work because SSL auth is required, not just 1-way SSL.
```
../bin/kafka-console-consumer --bootstrap-server <your kafka public server>:9093 --topic kafka-security-topic --from-beginning --consumer.config client.properties
```

So let's use the correct new properties file to produce and consume!
```
../bin/kafka-console-producer --broker-list <your kafka public server>:9093 --topic kafka-security-topic --producer.config client-ssl-auth.properties

../bin/kafka-console-consumer --bootstrap-server <your kafka public server>:9093 --topic kafka-security-topic --from-beginning --consumer.config client-ssl-auth.properties
```

Success! That's it.
