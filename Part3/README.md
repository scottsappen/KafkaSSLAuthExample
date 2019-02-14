# Part 3

**Part 1
Configure and test 1-way SSL authentication**

This is 1-way where the client will validate the broker's identity.

First let's setup the CA.
```
/home/ubuntu/confluent-5.1.1
mkdir ssl
cd ssl

openssl req -new -newkey rsa:4096 -days 365 -x509 -subj "/CN=Kafka-Security-CA" -keyout ca-key -out ca-cert -nodes
```

Now we will setup keystores and truststores on the brokers and make sure everything is working on a new secure port.

```
keytool -genkey -keystore kafka.server.keystore.jks -validity 365 -storepass confluent -keypass confluent -dname "CN=<your kafka public server>" -storetype pkcs12

keytool -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass confluent -keypass confluent

openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:confluent

keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file ca-cert -storepass confluent -keypass confluent

keytool -keystore kafka.server.keystore.jks -alias CARoot -import -file ca-cert -storepass confluent -keypass confluent -noprompt

keytool -keystore kafka.server.keystore.jks -import -file cert-signed -storepass confluent -keypass confluent -noprompt
```

At this point, we are done configuring the truststore and keystore. We can now move onto configuring the kafka brokers themselves.

```
sudo vi /etc/kafka/server.properties

listeners=PLAINTEXT://<your kafka public server>:9092,SSL://<your kafka public server>:9093
advertised.listeners=PLAINTEXT://<your kafka public server>:9092,SSL://<your kafka public server>:9093
ssl.keystore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.keystore.jks
ssl.keystore.password=confluent
ssl.key.password=confluent
ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.server.truststore.jks
ssl.truststore.password=confluent
```

Restart Kafka and again verify
```
./confluent stop
./confluent start

openssl s_client -connect <your kafka public server>:9093

grep "EndPoint" <your kafka logs location>/logs/server.log
```

You should see a plaintext and SSl endpoint.
```
EndPoint(<your kafka public server>,ListenerName(SSL),SSL)
```

You can do this on the same kafka server machine for testing, but you could do this on another kafka machine, like your local Mac laptop or another kafka server (to better emulate a "user" connecting remotely to kafka).

For your laptop, it may be something akin to:
```
scp -i ~<your pem file>.pem ubuntu@<your kafka public server>:/home/ubuntu/confluent-5.1.1/ssl/ca-cert .
```

```
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file ca-cert -storepass confluent -keypass confluent -noprompt

vi client.properties

security.protocol=SSL
ssl.truststore.location=<location of your client truststore file>/kafka.client.truststore.jks
ssl.truststore.password=confluent
```

Now let's verify we can connect. As a sanity, first again make sure non-SSL plaintext is working on the original port. Then try with SSL.

```
../bin/kafka-console-consumer --bootstrap-server <your kafka public server>:9092 --topic kafka-security-topic --from-beginning

../bin/kafka-console-producer --broker-list <your kafka public server>:9093 --topic kafka-security-topic --producer.config client.properties

../bin/kafka-console-consumer --bootstrap-server <your kafka public server>:9093 --topic kafka-security-topic --from-beginning --consumer.config client.properties
```

And of course, if you were to use the plaintext 9092 port with a regular consumer console, things would still work because our broker config still allows it. Similarly, if you were to try and connect to SSL port 9093, but not pass the client.properties file, it would not work (although, it does not crash per se, so a little confusing).

Success! You have 1-way SSL authentication (basically SSL encryption) working!

Now let's move onto 2-way SSL.
