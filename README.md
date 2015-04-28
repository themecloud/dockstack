#DockStack

DockStack is a Docker ambassador based on SmartStack http://nerds.airbnb.com/smartstack-service-discovery-cloud/. It uses zookeeper or etcd (WIP) as discovery service.

DockStack is composed of two tools:
- Nerve, a service registration daemon that performs health checks https://github.com/airbnb/nerve
- Synapse, a transparent service discovery framework that proxyfies the connections. It uses HAProxy to proxy the connections https://github.com/airbnb/synapse

## Why ?

Both tools use a YAML config file, which is not ideal to use with Docker. A wrapper script heavily simplifies the generation of this files via CLI.

## How it works ?

![schema]()

## Bootstrap demo

We will create a basic Docker ambassador pattern with two hosts, a Zookeeper, nerve and a MySQL instance on the fist host and Synapse and MySQL client of the second one.

#### On the first host

* Launch a Zookeeper instance

You can easily setup Zookeeper on your host 1 using https://registry.hub.docker.com/u/mbabineau/zookeeper-exhibitor/ but you need AWS crendentials because the image store the configuration in a s3 Bucket, and in this bucket, you have to create a folder that will be the `S3_PREFIX` parameter.

```
docker run -d --name zookeeper -p 8181:8181 -p 2181:2181 -p 2888:2888 -p 3888:3888 \
    -e S3_BUCKET=demo-dockstack \
    -e S3_PREFIX=demo \
    -e AWS_REGION=eu-west-1 \
    -e AWS_ACCESS_KEY_ID=YOUR_ID \
    -e AWS_SECRET_ACCESS_KEY=YOUR_KEY \
    -e ZK_PASSWORD=pass \
    -e HOSTNAME=host1 \
    mbabineau/zookeeper-exhibitor:latest
```

* Spawn a MySQL container

```
host1 $ docker run -d --name mysql -p :3306 -e MYSQL_ROOT_PASSWORD=test mysql
```

* Create an healt-check user

```
host1 $ docker exec -ti mysql mysql -u root -ptest -e "CREATE USER 'haproxy_check'@'%'; FLUSH PRIVILEGES;"
```

* Launch Nerve with the correct `HOST1_IP`.

```
host1 $ docker run \
	-d --name nerve \
	-v /usr/bin/docker:/usr/bin/docker:ro \
	-v /lib64/libdevmapper.so.1.02:/lib/libdevmapper.so.1.02:ro \
	-v /var/run/docker.sock:/var/run/docker.sock \
	-e SERVICE_HOST=HOST1_IP \
	tcio/dockstack-nerve \
	-d zk://HOST1_IP:2181/nerve \
	-s mysql:tcp:mysql:3306:/test
```

#### On the second host

* launch Synapse with the correct `HOST1_IP` value to have the zookeeper configuration, the instance is on host1.

```
host2 $ docker run \
	 -d --name synapse \
	tcio/dockstack-synapse \
	-d zk://HOST1_IP:2181/nerve \
	-s mysql:mysql:/test
```

* Here the fun happens thanks to Synapse, the magic unicorn. We are Running a mysql image because it carries by default mysql-client. The connection is proxified transparently by HAProxy embeded into Synapse.

```
host2 $ docker run -ti --rm --link synapse:db mysql:latest mysql -u root -h db -ptest
```

## TODO

Create more service config files and examples ready to use

## Upstream links:

* Docker Registry @ tcio/dockstack-nerve tcio/dockstack-synapse 
* GitHub @ themecloud/Dockstack

## Authors

- [Alessandro Siragusa](https://github.com/asiragusa)
- [Yves-Marie Saout](https://github.com/dw33z1lP)

##Special thanks

To [Jérôme Petazzoni](https://github.com/jpetazzo) for the great idea!
