# Interbroker SSL communication

This part is easy. You have enabled SSL and you want to use that as part of your inter-broker communication.

All you need to do is indicate this in your broker server.properties files.

To enable SSL inter-broker, all you have to do is update the server.properties file.

```
sudo vi /etc/kafka/server.properties

security.inter.broker.protocol=SSL
```

Restart Confluent

```
./bin/confluent start
```

That's it.
