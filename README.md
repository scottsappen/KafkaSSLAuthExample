# KafkaSSLAuthExample
A simple example of setting up Kafka (brokers and clients) to do 1-way and 2-way SSL authentication.

I recently expanded this repo to include Part 5 which adds in SSL configuration for other Confluent Kafka components.

1-way is basically SSL encryption where the clients verify the broker certs to establish a SSL connection. The clients are effectively "anonymous" whereas in 2-way authentication, the clients and the brokers have signed certs and both verify each other. In that way, the clients will have "identity" and this will help you when creating ACLs as well.

In this example, we will still leave the default 9092 port available as PLAINTEXT to show you how that still works even after enabling security. However, we will setup port 9093 with SSL and use simple consumers and producers to connect to those ports specifically passing in the right set of SSL parameters. For the Kafka client aspect, I use a combination of the Ubuntu server (acting as a client) as well as a laptop that has Confluent Kafka installed (I do some curl SSL commands from it).

**Prereqs**

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

You can use whatever boxes and O/S you want.

In this example, we will use an Ubuntu box in AWS for Kafka. Something like this would be fine:<br/>
x4.large, AWS Marketplace (search for Ubuntu), Ubuntu 16.04 LTS - Xenial (HVM).

Further, in the real world you may wish to change things such as the use of symbolic links, other tooling like systemctl and changing default ports.

**What we will do**

See the subdirectories in this repo for the details of each part:

- Part 1 - Install Java and Confluent Platform
- Part 2 - Setup plaintext un-secure for a quick test
- Part 3 - Configure and test 1-way SSL authentication
- Part 4 - Configure and test 2-way SSL authentication
- Part 5 - Configure other aspects of the Confluent platform with SSL

That's it. When you're done, I hope you will have enjoyed this walkthrough and learned something new.
