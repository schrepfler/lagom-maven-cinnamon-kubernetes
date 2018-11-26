# Lagom Recipe: Kubernetes deployment with Maven

This recipe demonstrates how to deploy a Lagom application in Kubernetes. It's based on the [reactive-lib](https://github.com/lightbend/reactive-lib) library for Akka Cluster bootstrapping and use a Lagom ServiceLocator based on `reactive-lib` service discovery mechanism. 

The application needs Cassandra and Kafka servers. For demo purposes, we will install a Helm chart with Cassandra and Kafka and let the deployed application connect to it. The provided Kubernetes deployment descriptors are pre-configured to expose Cassandra and Kafka to the application. You may use it as a basis to write descriptors suited for your production environment. 

## Setup

You'll need to ensure that the following software is installed:

* [Docker](https://www.docker.com/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)
* [Minikube](https://github.com/kubernetes/minikube) v0.28.2 or later (Verify with `minikube version`)
* [Helm](https://github.com/kubernetes/helm)

### Minikube Setup

#### 1) Start Minikube and setup Docker to point to Minikube's Docker Engine

> Want a fresh Minikube? Run `minikube delete` before the steps below.

```bash
minikube start --memory 6000 # see memory configuration section below
minikube addons enable ingress
eval $(minikube docker-env)
```

#### 2) Role-Based Access Control (RBAC)

Role-Based Access Control (RBAC) is a method of controlling access to protected resources based on the roles of individual users. Many production clusters and recent versions of Minikube come with RBAC.

Roles consists of sets of permissions for resources such as Pods and Secrets. Users are given roles using role bindings. Note that even a process running inside a pod belong to a special type of user called [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).

An application built using Lightbend Orchestration likely requires a permission to list pods due to the design of current Akka Management Service Discovery. You can find a suitable `rbac.yml` in the `deploy` directory and install it on your Kubernetes cluster with the following command: 

```bash
kubectl apply -f deploy/rbac.yml
```

#### 3) Install Reactive Sandbox

The `reactive-sandbox` includes development-grade (i.e. it will lose your data) installations of Cassandra, 
Elasticsearch, Kafka, and ZooKeeper. It's packaged as a Helm chart for easy installation into your Kubernetes cluster.

The `reactive-sandbox` allows you to easily test your deployment with Cassandra, Kafka, Zookeeper, and Elasticsearch without having to use the production-oriented versions. Because it is a sandbox, it does not persist data across restarts. The sandbox is packaged as a Helm chart for easy installation into your Kubernetes cluster.

```bash
helm init
helm repo add lightbend-helm-charts https://lightbend.github.io/helm-charts
helm repo update
```

Verify that Helm is available (this takes a minute or two):

>  The `-w` flag will watch for changes. Use `CTRL-c` to exit.

```bash
kubectl --namespace kube-system get -w deploy/tiller-deploy
```

```
NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
tiller-deploy   1         1         1            0           8s
tiller-deploy   1         1         1            1           3m
```

Note how we immediately see the `tiller-deploy` pod as being used but it may take 2 or 3 minutes to become available (`3m` in the example above). This delay will depend on your hardware and other running processes. Do not proceed with the steps on this guide until you see the `tiller-deploy` as `AVAILABLE == 1`. Once the pod is listed as available, your can kill this command using `Ctrl+C` to return to your prompt.

Install the sandbox.

```bash
helm install lightbend-helm-charts/reactive-sandbox --name reactive-sandbox --set elasticsearch.enabled=false
```
(Note that for this example, we don't need elasticsearch and therefore we disable it to avoid unnecessary deployments)

Verify that it is available (this takes a minute or two):

>  The `-w` flag will watch for changes. Use `CTRL-c` to exit.

```bash
kubectl get -w deploy/reactive-sandbox
```

```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
reactive-sandbox   1         1         1            0           11s
reactive-sandbox   1         1         1            1           1m
```

Note how we immediately (`11s`) see the `reactive-sandbox ` pod as being used but it could take up to 1 minute (`1m`) to become available. Do not proceed until you see the `reactive-sandbox` as `AVAILABLE == 1`. Kill this command using `Ctrl+C`.

## Deployment

The first step is to build docker images for each of the implementation services. 

```bash
mvn package docker:build 
```

Verify that docker images were locally published. 

```
docker images
```

It should display something similar to:

```
REPOSITORY      TAG                 IMAGE ID            CREATED             SIZE
stream-impl     1.0-SNAPSHOT        20ca397fe693        13 minutes ago      155MB
stream-impl     latest              20ca397fe693        13 minutes ago      155MB
hello-impl      1.0-SNAPSHOT        c49ea023b5e6        13 minutes ago      155MB
hello-impl      latest              c49ea023b5e6        13 minutes ago      155MB
...
```

Next step is to instruct Kubernetes how to deploy the new images. The `deploy` folder contains yml files for both services, `hello-impl` and `stream-impl`, and a ingress file that will expose the Services APIs to the external world. 

```bash
kubectl apply -f deploy/hello.yml
kubectl apply -f deploy/hello-stream.yml
kubectl apply -f deploy/ingress.yml
```

You can verify that all pods are running by calling:

```bash
kubectl get pods
```

It should display something similar to:

```bash
NAME                                          READY     STATUS             RESTARTS   AGE
hello-stream-v1-0-snapshot-67775f58db-2h7jv   1/1       Running            0          27m
hello-stream-v1-0-snapshot-67775f58db-qp2vr   1/1       Running            0          27m
hello-stream-v1-0-snapshot-67775f58db-w97xm   1/1       Running            0          27m
hello-v1-0-snapshot-86cdd499c4-2jklj          1/1       Running            0          27m
hello-v1-0-snapshot-86cdd499c4-6x89c          1/1       Running            0          27m
hello-v1-0-snapshot-86cdd499c4-v5mhm          1/1       Running            0          27m
reactive-sandbox-85bf697cc9-wt7zm             1/1       Running            0          1h
```

Once everyhing is running, you can access the servcices through the external Minikube address.  The service API are exposed on port 80.
You can obtain the external Minikube address, type `minikube ip` in your console.

```shell
$ ~ minikube ip
192.168.99.100
```

In your browser, open the url: [http://192.168.99.100/api/hello/Joe](http://192.168.99.100/api/hello/Joe)  
(replace 192.168.99.100 by the output of `minikube ip` if different).

## Memory usage

This recipe is just an example to illustrate the necessary bits to deploy a Lagom service in Kubernetes. 
This demo is using Minikube as example and therefore it has its limitations. 

We recommended to start Minikube with 6Gb of memory using the command `minikube start --memory 6000`. You can increase this value if your development machine has enough memory. 

The docker containers are limited to 512Mb (see yml file `resources` section) and the jvm running the services are limited to 256Mb (see JAVA_OPTS definition inside the yml file). You can increate those values according to your needs and development environment. 


## Production Setup

The provided yml files are preconfigure to run in Minikube and to point to the Cassandra and Kafka servers running inside the Reactive Sandbox. To adapt it for production, you will need to redefine the respective variables, i.e.: `RP_CASSANDRA_URL` and `RP_KAFKA_URL`, to point to your production servers. 

Moreover, the application secret (`RP_APPLICATION_SECRET`) is set to dummy value in both files. You must change this value for production. It's recommended to use distinct secrets per service. For more information, consult [Play's documentation](https://www.playframework.com/documentation/2.6.x/ApplicationSecret).

### Bintray credentials

Follow [these instructions](https://www.lightbend.com/product/lightbend-reactive-platform/credentials) to set up your Bintray credentials for Maven.

## Running Cinnamon locally

Lagom has a special development mode for rapid development, and does not fork the JVM when using the `lagom:runAll` or `lagom:run` commands in Maven. A forked JVM is necessary to gain metrics for actors and HTTP calls, since those are provided by the Cinnamon Java Agent. This example uses the [`exec-maven-plugin`](https://www.mojohaus.org/exec-maven-plugin/) to run the Lagom production server, enabling the complete set of metrics to be displayed.

You can test this recipe using 2 separate terminals.

On one terminal build and execute the service:

```bash
mvn install                  # builds the service and downloads the Cinnamon agent
mvn -pl hello-impl exec:exec # runs the hello service
```

The output should look something like this:

```
[INFO] [06/22/2018 16:21:58.464] [CoreAgent] Cinnamon Agent version 2.8.7
2018-06-22T06:52:01.687Z [info] akka.event.slf4j.Slf4jLogger [] - Slf4jLogger started
2018-06-22T06:52:02.143Z [info] cinnamon.chmetrics.CodaHaleBackend [sourceThread=main, akkaSource=CodaHaleBackend, sourceActorSystem=application, akkaTimestamp=06:52:02.140UTC] - Reporter com.lightbend.cinnamon.chmetrics.reporter.provided.ConsoleReporter started.
2018-06-22T06:52:03.212Z [info] play.api.Play [] - Application started (Prod)
2018-06-22T06:52:03.970Z [info] play.core.server.AkkaHttpServer [] - Listening for HTTP on /0:0:0:0:0:0:0:0:9000
6/22/18 4:22:07 PM =============================================================

-- Gauges ----------------------------------------------------------------------
metrics.akka.systems.application.dispatchers.akka_actor_default-dispatcher.active-threads
             value = 0
metrics.akka.systems.application.dispatchers.akka_actor_default-dispatcher.parallelism
             value = 8
metrics.akka.systems.application.dispatchers.akka_actor_default-dispatcher.pool-size
             value = 3
metrics.akka.systems.application.dispatchers.akka_actor_default-dispatcher.queued-tasks
             value = 0
metrics.akka.systems.application.dispatchers.akka_actor_default-dispatcher.running-threads
             value = 0
metrics.akka.systems.application.dispatchers.akka_io_pinned-dispatcher.active-threads
             value = 1
metrics.akka.systems.application.dispatchers.akka_io_pinned-dispatcher.pool-size
             value = 1
metrics.akka.systems.application.dispatchers.akka_io_pinned-dispatcher.running-threads
             value = 0
```

To try out the `hello-proxy` service call and see metrics for the HTTP endpoints, as well as the `hello` circuit breaker you can either point your browser to <http://localhost:9000/api/hello-proxy/World> or simply run `curl` from the command line:

```bash
curl -i http://localhost:9000/api/hello-proxy/World
```

The output from the server should now also contain metrics like this:

```
-- Gauges ----------------------------------------------------------------------
...
metrics.lagom.circuit-breakers.hello.state
             value = 3
...
-- Histograms ------------------------------------------------------------------
metrics.akka-http.systems.application.http-servers.0_0_0_0_0_0_0_1_9000.request-paths._api_hello-proxy__id.endpoint-response-time
             count = 1
               min = 481423809
               max = 481423809
              mean = 481423809.00
            stddev = 0.00
            median = 481423809.00
              75% <= 481423809.00
              95% <= 481423809.00
              98% <= 481423809.00
              99% <= 481423809.00
            99.9% <= 481423809.00
...
metrics.lagom.circuit-breakers.hello.latency
             count = 1
               min = 235385490
               max = 235385490
              mean = 235385490.00
            stddev = 0.00
            median = 235385490.00
              75% <= 235385490.00
              95% <= 235385490.00
              98% <= 235385490.00
              99% <= 235385490.00
            99.9% <= 235385490.00
...
```
