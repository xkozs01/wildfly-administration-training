# Lab - WildFly - Clustering and load balancing

## Task 1: Run HA cluster with a load-balancer

**What:** We will run several WildFly servers with deployed distributable application

**How:**

* We will run servers with `counter` and `counter-distributable` 
  applications. They differ just in the content of `web.xml` deployment
  descriptor. Both applications are simple request counters for the given user.

* Build both applications with `mvn install`

* To make deploying in cluster simpler, we will prepare a new docker image
  with the applications. There is already a `Dockerfile`, which defines
  the format of the new image.
  * there is also a JBoss CLI script `modify-ha-config.cli` which replaces multicast JGroups discovery (`MPING` protocol)
    by file-based discovery mechanism (`FILE_PING` protocol). It also uses static IP configuration in `mod_cluster`
    Undertow filter.
* To prepare the Docker image run:
```
# copy the application WAR files close to the Dockerfile
cp ~/wildfly-administration-training/applications/{counter,counter-distributable}/target/*.war \
   ~/wildfly-administration-training/labs/11_clustering/

# create the new docker image with name wildfly-cluster
docker build -t wildfly-cluster ~/wildfly-administration-training/labs/11_clustering
```

* Run the load-balancer node first:
```
# run load-balancer in a container from our new image, map host port 8080 to the loadbalancer:
docker run -p 8080:8080 -it --rm wildfly-cluster

# run WildFly with HA profile
wildfly/bin/standalone.sh -c standalone-load-balancer.xml -b $MY_IP -bprivate $MY_IP
```


* Start 2 WildFly servers with HA profile (run these commands in 2 terminals):
```
# run container from our new image
# It shares /tmp folder between the containers (and host) to make FILE_PING work properly
docker run -v /tmp:/tmp -it --rm wildfly-cluster

# run WildFly with HA profile
wildfly/bin/standalone.sh -c standalone-ha.xml -b $MY_IP -bprivate $MY_IP
```

* Open the **local** application addresses in browser:
  * http://localhost:8080/counter/
  * http://localhost:8080/counter-distributable/
* reload each application page few times and check which server handles the request
  * Stop the handling server (Ctrl-C)

**Expected result:**
Both applications should still work, but the one which is not distributable
starts the counter from zero.
The distributable application shares the session state across the cluster
and counter should still grow.
