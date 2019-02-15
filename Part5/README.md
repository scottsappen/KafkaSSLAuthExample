# Part 5

**Part 5<br/>
Configure other aspects of the Confluent platform with SSL**

This part will cover enabling SSL on a number of other Confluent components.

See the sub-directories for more information specific to those components. Recall, ZooKeeper does not support TLS at this stage.

- Kafka brokers (inter-broker communication to SSL)
- Confluent Control Center
- Kafka Connect and the REST interface
- Confluent REST Proxy
- KSQL REST interface
- Schema Registry

Note that in a lot of sections it will ask you to restart Confluent, as such:

```
./bin/confluent start
```

That is really for convenience. You can stop and start individual components like brokers or Control Center.
