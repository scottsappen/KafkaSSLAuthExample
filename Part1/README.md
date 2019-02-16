# Part 1

**Part 1
Install Java and Confluent Platform**

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

Now let's install the Confluent platform.

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
http://<your kafka server>:9021
```

That's it. You are ready to move onto the next part.
