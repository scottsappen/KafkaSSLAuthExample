# KafkaSSLAuthExample
A simple example of setting up Kafka (brokers and clients) to do 1-way and 2-way SSL authentication. 1-way is basically SSL encryption where the clients verify the broker certs to establish a SSL connection. The clients are effectively "anonymous" whereas in 2-way authentication, the clients and the brokers have signed certs and both verify each other. In that way, the clients will have "identity" and this will help you when creating ACLs as well.

**Prereqs**

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

You can use whatever boxes and O/S you want.

In this example, we will use an Ubuntu box in AWS for Kafka. Something like this would be fine:<br/>
x4.large, AWS Marketplace (search for Ubuntu), Ubuntu 16.04 LTS - Xenial (HVM). For your Kafka client, you can use the same Ubuntu box and pretend that it is not only a server, but also a client. In this example, we will stick to using a separate laptop that has kafka and the various networking tools installed.

Further, in the real world you may wish to change things such as the use of symbolic links, other tooling like systemctl and changing default ports.

**What we will do**

Part 1 - Install Java
<br/>
Part 2 - Install Confluent platform
<br/>
Part 3 - Setup plaintext un-secure for a quick test
<br/>
Part 4 - Configure and test 1-way SSL authentication
<br/>
Part 5 - Configure and test 2-way SSL authentication

<br/><br/>

**Part 1
Install Java**

Let's install Java and some handy tools we can use later. Some of those tools may already be installed for you.

```
sudo apt-get install -y wget net-tools netcat tar openjdk-8-jdk

java -version
```

You can alternatively install another Java if you want, such as:
```
sudo add-apt-repository ppa:webupd8team/java
sudo apt update
sudo apt install oracle-java8-installer
```

<br/><br/>

**Part 2
Install Confluent platform**

```
curl -O http://packages.confluent.io/archive/5.1/confluent-5.1.1-2.11.tar.gz
tar -xvf confluent-5.1.1-2.11.tar.gz
cd confluent-5.1.1/
cd bin
./confluent start
```

That was simple. You should not have any issues. If you get errors, confirm you started on an instance with enough resources (e.g. RAM). Check out a few things...
```
./confluent status
./kafka-topics --zookeeper localhost:2181 --list
```
Navigate to Confluent Control Center in your browser:
```
http://<your kafka public server>:9021
```

<br/><br/>

**Part 3
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
./kafka-topics --zookeeper <your kafka public server>:2181 --create --topic kafka-security-topic --replication-factor 1 --partitions 2

./kafka-console-producer --broker-list <your kafka public server>:9092 --topic kafka-security-topic

./kafka-console-consumer --bootstrap-server <your kafka public server>:9092 --topic kafka-security-topic
```

Success!

Now let's move onto security now that basic un-secure functionality is confirmed.

<br/><br/>

**Part 4
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

<br/><br/>

**Part 5
Configure and test 2-way SSL authentication**

This is 2-way where the broker will validate the client's identify. This requires us to generate certs for our clients, but also reconfigure the brokers to require SSL authentication.

Mac Sappenfield
```
keytool -genkey -keystore kafka.client.keystore.jks -validity 365 -storepass confluent -keypass confluent -dname "CN=mylaptop" -alias my-local-pc -storetype pkcs12

keytool -keystore kafka.client.keystore.jks -certreq -file client-cert-sign-request -alias my-local-pc -storepass confluent -keypass confluent
```

Ubuntu
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
```

Mac Sappenfield
```
keytool -keystore kafka.client.keystore.jks -alias my-local-pc -import -file client-cert-signed -storepass confluent -keypass confluent -noprompt
```

Ubuntu
```
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

I hope you enjoyed this walkthrough.
