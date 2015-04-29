We will create a basic Docker ambassador pattern with two hosts, a Zookeeper, nerve and a RabitMQ instance on the fist host and Synapse and MySQL client of the second one.

![ambassador](https://raw.githubusercontent.com/themecloud/dockstack/master/ambassador.jpg)

#### On the first host

* Launch a Zookeeper instance

You can easily setup Zookeeper on your host 1 using https://registry.hub.docker.com/u/mbabineau/zookeeper-exhibitor/ but you need AWS crendentials because the image store the configuration in a s3 Bucket, and in this bucket, you have to create a folder that will be the `S3_PREFIX` parameter.

```
docker run -d --name zookeeper \
    -p 8181:8181 -p 2181:2181 -p 2888:2888 -p 3888:3888 \
    -e S3_BUCKET=demo-dockstack \
    -e S3_PREFIX=demo \
    -e AWS_REGION=eu-west-1 \
    -e AWS_ACCESS_KEY_ID=YOUR_ID \
    -e AWS_SECRET_ACCESS_KEY=YOUR_KEY \
    -e ZK_PASSWORD=pass \
    -e HOSTNAME=host1 \
    mbabineau/zookeeper-exhibitor:latest
```

* Spawn a RabbitMQ container

```
host1 $ docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 tutum/rabbitmq
```

* Launch Nerve with the correct `HOST1_IP`.

```
docker run \
    -d --name nerve \
    -v /usr/bin/docker:/usr/bin/docker:ro \
    -v /lib64/libdevmapper.so.1.02:/lib/libdevmapper.so.1.02:ro \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -e SERVICE_HOST=HOST1_IP \
    --link rabbitmq:rabbitmq \
    tcio/dockstack-nerve \
    -d zk://HOST1_IP:2181/nerve \
    -s rabbitmq:rabbitmq:rabbitmq:5672:/rabbitmq
```

#### On the second host

* launch Synapse with the correct `HOST1_IP` value to have the zookeeper configuration, the instance is on host1.

```
docker run \
     -d --name synapse \
    tcio/dockstack-synapse \
    -d zk://HOST1_IP:2181/nerve \
    -s rabbitmq:rabbitmq:/rabbitmq
```

* When can remotely access to the RabbitMQ server thanks to Synapse :

```
host2 $ docker run -ti --link synapse:rabbitmq debian:jessie bash
apt-get update && apt-get install rabbitmq && nmap -p 5672 rabbitmq
...
PORT     STATE SERVICE
5672/tcp open  amqp
```
