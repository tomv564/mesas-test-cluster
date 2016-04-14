
# Goals

Avoid manually deploying to each machine
Scaling
Automatic load balancing
Automatic failover?
Should survive a reboot

## Breaking up into containers, why?

http://rhelblog.redhat.com/2016/03/16/container-tidbits-when-should-i-break-my-application-into-multiple-containers/
One process per container? Why? There are many valid reasons for multiple processes in a container (eg JVM memory efficiency, web servers fork processes all the time.)
"Processes and threads are about kernel resource allocation; containers are about cluster resource allocation."
Also, if application is not easily decomposable then don't waste time refactoring just to containerize.
Example: Splitting up a Java front-end and back-end REST apps.
 * Probably little JVM benefit (why?)
 * REST-api instead of sockets or shared memory.
 * Can scale independently

## Set up a master node

Install Virtualbox and Vagrant

vagrant init debian/jessie64
specify "node1" hostname in vagrant file
vagrant up
add mesosphere source: 
```
# Setup
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E56151BF

# Add the repository
echo "deb http://repos.mesosphere.io/debian jessie main" | 
  sudo tee /etc/apt/sources.list.d/mesosphere.list
sudo apt-get -y update
```

### Add java 
 
(what a pain in the ass!)
http://www.webupd8.org/2014/03/how-to-install-oracle-java-8-in-debian.html

```
echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main" | tee /etc/apt/sources.list.d/webupd8team-java.list
echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main" | tee -a /etc/apt/sources.list.d/webupd8team-java.list
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys EEA14886
apt-get update
```

### Zookeeper

Should already be installed by the above.
configure
`echo "1" > /var/lib/zookeeper/myid`

start
```
sudo service zookeeper start
usr/lib/zookeeper/bin/cli_mt localhost:2181 
```

### Mesos

configure

```echo "test-cluster" | sudo tee  /etc/mesos-master/cluster
test-cluster
```

some error when upping node1
Also, rsync entire package.box to node2 on startup, ugh.

### Slave setup

Disable startup of mesos-master
https://www.digitalocean.com/community/tutorials/how-to-configure-a-linux-service-to-start-automatically-after-a-crash-or-reboot-part-1-practical-examples

Fix master: slave failed to connect because master announced with dank localhost address. edit /etc/mesos-master/ip on the master instance.


### Docker setup

Install docker (both nodes)
``` 
sudo apt-get install docker
```

Make sure starts on boot

`sudo systemctl enable docker.service`

### Docker deployment

```
sudo docker build -t myappname
```

Run:

```
sudo docker run --publish 6060:8080 --name test --rm myappname
```
Are there onbuild packages for sbt? Or have to use the docker-sbt plugin?
Also, the alpine-oraclejdk image is smaller and perhaps preferred.

Distribute:

```
sudo docker save --output=myappname.tar.gz myappname
```

Copy to the /vagrant mounted dir.

```
sudo docker load --input==/vagrant/myappname.tar.gz
```

### Load balancing

Use marathon-lb
Can auto-scale if provided with target rps per instance.
What about health data provided by finagle?


### Really install Docker

https://docs.docker.com/engine/installation/linux/debian/

Then configure mesos:

echo 'docker,mesos' | sudo tee /etc/mesos-slave/containerizers
sudo service mesos-slave restart

https://mesosphere.github.io/marathon/docs/native-docker.html


### docker registry

https://mesosphere.github.io/marathon/docs/recipes.html

use /var/lib/registry
Set up very short on memory (247?)

Constrain to hostname:CLUSTER:node1

Startup issues all the time. Solution

* make sure it is totally killified
* update configuration to use HOST networking
* add parameter "publish", "5000:5000"

It runs but it is useless when also running the local docker daemon in a VM that cannot resolve `node1` or even reach your virtualbox IPs.

Solution: use docker hub

`docker login`
`docker tag kestrel tomv564/kestrel:latest`
`docker push tomv564/kestrel`

# OKAY, it's running. 

Put it on containerPort 6000 and servicePort 6000
How can I resolve this?

Mesos-dns was not set up as a resolver, added to /etc/resolv.conf
now, `dig registration.marathon.mesos` works

TODO: mesos no module named mesos in mesos-ps ??

From https://github.com/mesosphere/mesos-dns/issues/334:
`dig _registration._tcp.marathon.mesos SRV`

Finagle could use a DNS Cluster: https://gist.github.com/agleyzer/6909056

Can't see stdout from registration, what's up?