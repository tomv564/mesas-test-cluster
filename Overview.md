
# Cluster systems

Cluster management. 
Aurora (one big machine model).
Kubernetes (more container-style from Google)
Marathon (container-style for Mesos. Ran at AirBnB, no batch jobs)
Mesosphere (owns Marathon and sells another administrative layer over top?)

Marathon set up:
https://www.digitalocean.com/community/tutorials/how-to-configure-a-production-ready-mesosphere-cluster-on-ubuntu-14-04

Service discovery:

Zookeeper is CP, not necessarily Available (airbnb has added caching to fix this?) https://tech.knewton.com/blog/2014/12/eureka-shouldnt-use-zookeeper-service-discovery/
Etcd (Raft based) or Eureka (Netflix)
http://www.simplicityitself.io/getting/started/with/microservices/2015/06/10/service-discovery-overview.html

### Aurora

 The operational hierarchy is:

* Aurora manages and schedules jobs for Mesos to run.
* Mesos manages the individual tasks that make up a job.
* Thermos manages the individual processes that make up a task.

http://tepid.org/tech/01-aurora-mesos-cluster/

#### What does it do? 

Manages jobs (workers or services?)
Publishes them to Zookeeper
Uses Mesos

To run something, run aurora update start $jobkey $jobfile
Job keys: cluster/role/environment/jobname
Job file: python!
Jobs is an array containing: 

* Service tying together (cluster env role name) to task
* Task (SequentialTask) tying processes[] to Resources(cpu =1, ram = 1*MB, disk=8*mb)
* An install Process (name, cmdline) to copy/download/install the app
* A run Process to start it up. 

### Mesos

Abstracts physical hardware into a virtual pool of resources
Also uses Zookeeper
Containers: Supports Docker, what about rkt?

Shutdown:

* if health port exists it should respond to POST /quitquitquit
* POST /abortabortabort
* SIGTERM
* SIGKILL

### Thermos

Manages tasks?

### Docker integration

All nodes need to have docker installed
But how to specify a central location for all slaves to pull image from?
Deployment should be as easy as:
Build app
Push app to registry
(Rolling) restarts to update app from registry
