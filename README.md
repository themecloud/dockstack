#DockStack

DockStack is a Docker ambassador based on SmartStack http://nerds.airbnb.com/smartstack-service-discovery-cloud/. It uses zookeeper or etcd (WIP) as discovery service.

DockStack is composed of two tools:
- Nerve, a service registration daemon that performs health checks https://github.com/airbnb/nerve
- Synapse, a transparent service discovery framework that proxyfies the connections. It uses HAProxy to proxy the connections https://github.com/airbnb/synapse

## Why ?

Both tools use a YAML config file, which is not ideal to use with Docker. A wrapper script heavily simplifies the generation of this files via CLI.

## How it works ?

## Bootstrap demo

* Install Zookeeper

You can easily setup Zookeeper on your host 1 using https://registry.hub.docker.com/u/mbabineau/zookeeper-exhibitor/ This image embed a zookeeper.

* Spawn a MySQL container

```
host1 $ docker run -d --name mysql -p :3306 -e MYSQL_ROOT_PASSWORD=test mysql
```

* Create an healt-check user

```
host1 $ docker exec -ti mysql mysql -u root -ptest -e "CREATE USER 'haproxy_check'@'%'; FLUSH PRIVILEGES;"
```

* Launch Nerve

```
host1 $ docker run \
	-d --name nerve
	-v /usr/bin/docker:/usr/bin/docker:ro \
	-v /lib64/libdevmapper.so.1.02:/lib/libdevmapper.so.1.02:ro \
	-v /var/run/docker.sock:/var/run/docker.sock \
	-e SERVICE_HOST=HOST1_IP \
	tcio/dockstack-nerve \
	-d zk://HOST1_IP:2181/nerve \
	-s mysql:tcp:mysql:3306:/test
```

* On the other host, launch Synapse

```
host2 $ docker run \
	 -d --name synapse
	tcio/dockstack-synapse
	-d zk://HOST2_IP:2181/nerve \
	-s mysql:mysql:/test
```

* Here the magic happens! (Running a mysql image, as it carries by default mysql-client)

```
host2 $ docker run -ti --rm --link synapse:db mysql:latest mysql -u root -h db -ptest
```

The connection is proxified transparently by HAProxy

## TODO

Create more service config files and examples ready to use

## Upstream links:

Docker Registry @ tcio/dockstack-nerve tcio/dockstack-synapse 
GitHub @ themecloud/Dockstack

## Authors

- [Alessandro Siragusa](https://github.com/asiragusa)
- [Yves-Marie Saout](https://github.com/dw33z1lP)

##Special thanks

To [Jérôme Petazzoni](https://github.com/jpetazzo) for the great idea!
