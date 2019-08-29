# Lab 17 - Daemonset

A DaemonSet make sure that all or some kubernetes Nodes run a copy of a Pod. When a new node is added to the cluster, a Pod is added to it to match the rest of the nodes and when a node is removed from the cluster, the Pod is garbage collected. Deleting a DaemonSet will clean up the Pods it created. In simple terms, this is what a DaemonSet is. As the name implies, it allow us to run a daemon on every node.

## Why use a DaemonSet

Ok now that we know what it is, what are some use cases and why would you need it.

* running a cluster storage daemon, such as glusterd, ceph, on each node.
* running a logs collection daemon on every node, such as fluentd or logstash.
* running a node monitoring daemon on every node, such as Prometheus Node Exporter, collectd, Datadog agent etc.

This list above can be expanded to so many other use cases like a more complex set up where we use multiple DaemonSets for a single type of daemon, but with different flags and/or different memory and cpu requests for different hardware types.

## Task 0: Creating a namespace

Create a namespace for this lab:

```
kubectl create ns lab-17

namespace "lab-17" created
```

## Task 1: Working with DaemonSets

Just like any other manifest in kubernetes, apiVersion, kind, and metadata fields are required. To show the other fields in the manifest, we will deploy an example of fluentd-elasticsearch image that we want running on every node. The idea is that we want to have a daemon of this on every node collecting logs for us and sending it to ES.

demo.yaml:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: lab-17
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: gcr.io/fluentd-elasticsearch/fluentd:v2.5.1
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

There are certain things to keep in mind when using DaemonSets:

* When a Daemonset is created the .spec.selector can not be changed, if you do, it will break things.
* You must specify a pod selector that matches the labels of the .spec.template.
* You should not normally create any Pods whose labels match this selector, either directly, via another DaemonSet, or via other controller such as ReplicaSet. Otherwise, the DaemonSet controller will think that those Pods were created by it.

It is worth noting that you can deploy a DaemonSet to run only on some nodes and not all of them if you specify a .spec.template.spec.nodeSelector. It will be deployed to any node that matches the selector.

Now let’s run `kubectl create -f demo.yaml -n lab-17` to deploy the example.

## Task 2: Make sure it is running

```
$ kubectl get daemonset -n lab-17
NAME              DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   3         3         3         3            3           <none>          59s
```

First let us see how many nodes we have by running “kubectl get nodes” to see the identity of our nodes.

```
$ kubectl get node
NAME                 STATUS    ROLES     AGE       VERSION
node2                Ready     <none>    92d       v1.10.3
node1                Ready     <none>    92d       v1.10.3
node3                Ready     <none>    92d       v1.10.3
```

Now to confirm, we want to make sure we have all pods running and also to make sure they are running on every node.

```
$ kubectl get pod -o wide -n lab-17
NAME                             READY     STATUS    RESTARTS   AGE       IP           NODE
fluentd-es-demo-bfpf9            1/1       Running   0          1m       10.0.0.3    node3
fluentd-es-demo-h4w85            1/1       Running   0          1m       10.0.0.1    node1
fluentd-es-demo-xm2rl            1/1       Running   0          1m       10.0.0.2    node2
```

## Task 3: Cleaning up

In order to delete our daemonset use the following command:

```
kubectl delete daemonset fluentd-elasticsearch -n lab-17
daemonset.extensions "fluentd-elasticsearch" deleted

kubectl delete ns lab-17
namespace "lab-08" deleted
```