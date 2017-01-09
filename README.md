# Tyk + Kubernetes integration

This guide will walk you through a Kubernetes-based Tyk setup. It also covers the required steps to configure and use a Redis cluster.

# Requirements

- A fresh Google Cloud Platform project and credentials, see the ["Creating a project via the Cloud Platform Console" section](https://cloud.google.com/resource-manager/docs/creating-project).
- The Google Cloud Platform CLI tool, download [here](https://cloud.google.com/sdk/).

# Getting Started

Clone this repository in your workspace:

```
$ cd ~
$ git clone https://github.com/TykTechnologies/tyk-kubernetes.git
$ cd tyk-kubernetes
```

# Redis setup

Enter the `redis-cluster` directory:

```
$ cd redis-cluster
```

First we will create the disks for our Redis instances:

```
$ gcloud compute disks create --size=10GB \
    'redis-1' 'redis-2' 'redis-3' \
    'redis-4' 'redis-5' 'redis-6'
```

Then we import the Redis configuration:

```
$ kubectl create configmap redis-conf --from-file=redis.conf
```

And initialize the [replica sets](http://kubernetes.io/docs/user-guide/replicasets/) and its pods:

```
$ kubectl create -f replicasets
```

To create the [services](http://kubernetes.io/docs/user-guide/services/) run:

```
$ kubectl create -f services
```

At this point we should have 6 Redis instances, running in each pod. Kubernetes DNS will provide A records for our services, as described [here](http://kubernetes.io/docs/admin/dns/).

## Configuring the cluster

To perform the cluster configuration, we will use a basic Ubuntu pod:

```
$ kubectl run -i --tty ubuntu --image=ubuntu \
    --restart=Never /bin/bash
```

This command may take some time until you get the command prompt.

To install the tools required to configure the cluster, run:

```
$ apt-get update
$ apt-get install -y python2.7 python-pip redis-tools dnsutils
$ pip install redis-trib
```

To configure three masters we use the following command, from the [redis-trib.py](https://github.com/HunanTV/redis-trib.py) tool:

```
$ redis-trib.py create `dig +short redis-1.default.svc.cluster.local`:6379 \
    `dig +short redis-2.default.svc.cluster.local`:6379 \
    `dig +short redis-3.default.svc.cluster.local`:6379
```

Each argument will become the resolved service IP plus the port.

To set a slave for each master we use:

```
$ redis-trib.py replicate --master-addr `dig +short redis-1.default.svc.cluster.local`:6379 --slave-addr `dig +short redis-4.default.svc.cluster.local`:6379
$ redis-trib.py replicate --master-addr `dig +short redis-2.default.svc.cluster.local`:6379 --slave-addr `dig +short redis-5.default.svc.cluster.local`:6379
$ redis-trib.py replicate --master-addr `dig +short redis-3.default.svc.cluster.local`:6379 --slave-addr `dig +short redis-6.default.svc.cluster.local`:6379
```

# Tyk setup

## Gateway setup

Enter the `tyk` directory:

```
$ cd ~/tyk-kubernetes/tyk
```

Create a volume for the gateway:

```
$ gcloud compute disks create --size=10GB tyk-gateway
```

Initialize the Tyk namespace:

```
$ kubectl create -f namespaces/
```

Create config map for `tyk.conf`:

```
$ kubectl create configmap tyk-gateway-conf --from-file=tyk.conf --namespace=tyk
```

Initialize the deployment:

```
$ kubectl create -f deployments/tyk-gateway.yaml --namespace=tyk
```

## Dashboard setup

