# Lab 18 - StatefulSets

If you have a stateless app you want to use a deployment. However, for a stateful app you might want to use a StatefulSet. Unlike a deployment, the StatefulSet provides certain guarantees about the identity of the pods it is managing (that is, predictable names) and about the startup order. Two more things that are different compared to a deployment: for network communication you need to create a headless services and for persistency the StatefulSet manages a persistent volume per pod.

## Task 0: Creating a namespace

Create a namespace for this lab:

```
kubectl create ns lab-18

namespace "lab-18" created
```

## Task 1: Working with StatefulSets

Let’s start with creating the stateful app, that is, the StatefulSet along with the persistent volumes and the headless service:

stateful.yaml

```
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mehdb
spec:
  selector:
    matchLabels:
      app: mehdb
  serviceName: "mehdb"
  replicas: 2
  template:
    metadata:
      labels:
        app: mehdb
    spec:
      containers:
      - name: shard
        image: quay.io/mhausenblas/mehdb:0.6
        ports:
        - containerPort: 9876
        env:
        - name: MEHDB_DATADIR
          value: "/mehdbdata"
        livenessProbe:
          initialDelaySeconds: 2
          periodSeconds: 10
          httpGet:
            path: /status
            port: 9876
        readinessProbe:
          initialDelaySeconds: 15
          periodSeconds: 30
          httpGet:
            path: /status?level=full
            port: 9876
        volumeMounts:
        - name: data
          mountPath: /mehdbdata
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mehdb
  labels:
    app: mehdb
spec:
  ports:
  - port: 9876
  clusterIP: None
  selector:
    app: mehdb
```

Now let’s run `kubectl create -f stateful.yaml -n lab-18` to deploy the example.

## Task 2: Make sure it is running

```
$ kubectl get sts,po,pvc,svc -n lab-18
NAME                     DESIRED   CURRENT   AGE
statefulset.apps/mehdb   2         2         3m22s

NAME                                                       READY   STATUS        RESTARTS   AGE
pod/mehdb-0                                                1/1     Running       0          3m22s
pod/mehdb-1                                                1/1     Running       0          2m28s
pod/quickstart-nginx-ingress-controller-5649859885-vvkhd   0/1     Terminating   0          125m

NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/data-mehdb-0   Bound    pvc-aab23874-c8c2-11e9-b7ce-42010a84026a   1Gi        RWO            standard       3m22s
persistentvolumeclaim/data-mehdb-1   Bound    pvc-cb158331-c8c2-11e9-b7ce-42010a84026a   1Gi        RWO            standard       2m28s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.0.16.1    <none>        443/TCP    144m
service/mehdb        ClusterIP   None         <none>        9876/TCP   3m22s
```

Now we see that every pod has his own volume attached. So they every seperate pod has it's own seperate state.
We can also learn from this that the second pod only starts creating after the first one is fully operational.

### Scaling up

In one terminal window, watch the Pods in the StatefulSet.
```
kubectl get pods -w -l app=mehdb -n lab-18
```
In another terminal window, use kubectl scale to scale the number of replicas to 4.
```
kubectl scale sts web --replicas=4 -n lab-18
statefulset.apps/mehdb scaled
```
Examine the output of the kubectl get command in the first terminal, and wait for the three additional Pods to transition to Running and Ready.

```
kubectl get pods -w -l app=mehdb -n lab-18
NAME      READY   STATUS    RESTARTS   AGE
mehdb-0   1/1     Running   0          69m
mehdb-1   1/1     Running   0          68m
mehdb-2   0/1     Pending   0          2s
mehdb-2   0/1   Pending   0     3s
mehdb-2   0/1   Pending   0     3s
mehdb-2   0/1   ContainerCreating   0     3s
mehdb-2   0/1   Running   0     33s
mehdb-2   1/1   Running   0     73s
mehdb-3   0/1   Pending   0     0s
mehdb-3   0/1   Pending   0     0s
mehdb-3   0/1   Pending   0     3s
mehdb-3   0/1   Pending   0     3s
mehdb-3   0/1   ContainerCreating   0     3s
mehdb-3   0/1   Running   0     13s
```

The StatefulSet controller scaled the number of replicas. As with StatefulSet creation, the StatefulSet controller created each Pod sequentially with respect to its ordinal index, and it waited for each Pod’s predecessor to be Running and Ready before launching the subsequent Pod.

### Scaling Down

In one terminal, watch the StatefulSet’s Pods.
```
kubectl get pods -w -l app=mehdb -n lab-18
```
In another terminal, use kubectl patch to scale the StatefulSet back down to 2 replicas.
```
kubectl patch sts mehdb -p '{"spec":{"replicas":2}}' -n lab-18
statefulset.apps/mehdb patched
```

```
kubectl get pods -w -l app=mehdb -n lab-18
NAME      READY   STATUS        RESTARTS   AGE
mehdb-0   1/1     Running       0          75m
mehdb-1   1/1     Running       0          74m
mehdb-2   1/1     Running       0          72m
mehdb-3   1/1     Running       0          70m
mehdb-3   0/1   Terminating   0     6m4s
mehdb-3   0/1   Terminating   0     6m5s
mehdb-2   0/1   Terminating   0     6m5s
mehdb-2   0/1   Terminating   0     6m5s
```
The controller deleted one Pod at a time, in reverse order with respect to its ordinal index, and it waited for each to be completely shutdown before deleting the next.

Get the StatefulSet’s PersistentVolumeClaims.

```
kubectl get pvc -l app=mehdb -n lab-18
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mehdb-0   Bound    pvc-aab23874-c8c2-11e9-b7ce-42010a84026a   1Gi        RWO            standard       84m
data-mehdb-1   Bound    pvc-cb158331-c8c2-11e9-b7ce-42010a84026a   1Gi        RWO            standard       83m
data-mehdb-2   Bound    pvc-66405527-c8cc-11e9-b7ce-42010a84026a   1Gi        RWO            standard       14m
data-mehdb-3   Bound    pvc-91883b0f-c8cc-11e9-b7ce-42010a84026a   1Gi        RWO            standard       13m
```

There are still four PersistentVolumeClaims and four PersistentVolumes. When exploring a Pod’s stable storage, we now know that the PersistentVolumes mounted to the Pods of a StatefulSet are not deleted when the StatefulSet’s Pods are deleted. This is still true when Pod deletion is caused by scaling the StatefulSet down.

## Task 3: Cleaning up

In order to delete our daemonset use the following command:

```
kubectl delete ns lab-18
namespace "lab-18" deleted
```
