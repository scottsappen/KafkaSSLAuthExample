# KafkaSSLAuthExample
A simple example of setting up Kafka (brokers and clients) to do 1-way and 2-way SSL authentication.

1-way is basically SSL encryption where the clients verify the broker certs to establish a SSL connection. The clients are effectively "anonymous" whereas in 2-way authentication, the clients and the brokers have signed certs and both verify each other. In that way, the clients will have "identity" and this will help you when creating ACLs as well.

**Prereqs**

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

You can use whatever boxes and O/S you want.

In this example, we will use an Ubuntu box in AWS for Kafka. Something like this would be fine:<br/>
x4.large, AWS Marketplace (search for Ubuntu), Ubuntu 16.04 LTS - Xenial (HVM). For your Kafka client, you can use the same Ubuntu box and pretend that it is not only a server, but also a client. In this example, we will stick to using a separate laptop that has kafka and the various networking tools installed.

Further, in the real world you may wish to change things such as the use of symbolic links, other tooling like systemctl and changing default ports.

**What we will do**

See the subdirectories in this repo for the details of each part:

- Part 1 - Install Java and Confluent Platform
- Part 2 - Setup plaintext un-secure for a quick test
- Part 3 - Configure and test 1-way SSL authentication
- Part 4 - Configure and test 2-way SSL authentication

That's it. When you're done, I hope you will have enjoyed this walkthrough and learned something new.
