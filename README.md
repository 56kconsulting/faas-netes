faas-netes
===========

This is a PoC (Proof of Concept) for a FaaS implementation on Kubernetes.

The code in this repository is a daemon or micro-service which can provide the basic functionality the FaaS Gateway requires:

* List functions
* Deploy function
* Delete function
* Invoke function synchronously

Any other metrics or UI components will be maintained separately in the main FaaS project.

Motivation for separate micro-service:

* Kubernetes go-client is 41MB with only a few lines of code
* After including the go-client the code takes > 2mins to compile

So rather than inflating the original project's source-code this micro-service will act as a co-operator or plug-in. Some additional changes will be needed in the main FaaS project to switch between implementations.

There is no planned support for dual orchestrators - i.e. Swarm and K8s at the same time on the same host/network.

### Get started with the code

Let's try it out:

* Create a single-node cluster on our Mac
* Deploy a function manually with `kubectl`
* Build and deploy the FaaS-netes microservice
* Make calls to list the functions and invoke a function


**Create a cluster on Mac:**

```
$ minikube start --vm-driver=xhyve
```

> You can also omit `--vm-driver=xhyve` if you want to use VirtualBox for your local cluster.

**Start a "function"**

* Create a deployment and service pair
* The label `faas_function=true` marks this as a "function"

```
$ kubectl delete deployment/nodeinfo ; \
  kubectl delete service/nodeinfo ; \
  kubectl run --labels="faas_function=true" nodeinfo --port 8080 --image functions/nodeinfo:latest ; \
  kubectl expose deployment/nodeinfo
```

**Build and deploy the development image:**

```
$ eval $(minikube docker-env)
$ docker build -t faas-netesd .
$ kubectl delete service/faas-netesd ; \
   kubectl delete deployment/faas-netesd ; \
   kubectl run faas-netesd --image=faas-netesd:latest --port 8080 --image-pull-policy=Never ; \
   kubectl expose deployment/faas-netesd
```

### Now try it out

**Function List**

This it the route for the function list as used by the FaaS UI / gateway.

```
$ kubectl get service faas-netesd
NAME          CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
faas-netesd   10.0.0.46    <none>        8080/TCP   10s

$ minikube ssh 'curl -s 10.0.0.46:8080/system/functions'

[{"name":"nodeinfo","image":"functions/nodeinfo:latest","invocationCount":0,"replicas":1}]
```

Internally within the cluster the `faas-netesd` service will have the DNS entry of `faas-netesd.default`.

**Invoke a function via Gateway**

```
$ kubectl get service faas-netesd
NAME          CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
faas-netesd   10.0.0.46    <none>        8080/TCP   10s

$ minikube ssh 'curl -s 10.0.0.46:8080/function/nodeinfo'
Hostname: nodeinfo-3504543019-q96rm

Platform: linux
Arch: x64
CPU count: 2
Uptime: 500
```

**Manually scale a function**

Let's scale the deployment from 1 to 2 instances of the nodeinfo function:

```
$ kubectl scale deployment/nodeinfo --replicas=2
```

You can now use the `curl` example from above and you will see either of the two replicas.

#### Get involved

Contributions are welcome - see the contributing guide for [FaaS](https://github.com/alexellis/faas/blob/master/CONTRIBUTING.md).
