# Pod Placement

This step covers Kubernetes Security Checklist items under Pod Placement:
- Pod placement is done in accordance with the tiers of sensitivity of the application.
- Sensitive applications are running isolated on nodes or with specific sandboxed runtimes.

One of the principles of zero trust is to assume there will be a breach and to minimize the blast radius.
This approach can be applied to Kubernetes clusters by assuming a container breakout will happen and minimize the blast radius of a container breakout by using node isolation to place pods in accordance to tiers of sensitivity.
A common example is to not place application pods on control plane nodes just in case there is a container breakout from the application pod.
Other examples include isolating highly sensitive application pods in specific nodes or reassuring Pods that use specific hardware are placed on nodes with access to the hardware.

In this step we will use Node Selectors, Taints, and Tolerations to ensure highly sensitive Pods are isolated and to place Pods on a Node based on a label on the Node.

### Verifying the cluster

A cluster is made up of many worker Nodes.

#### 1. Verify the Nodes in your cluster:

```shell,run
kubectl get nodes
```

Expected output:
```text
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   5d    v1.31.0
kind-worker          Ready    <none>          5d    v1.31.0
kind-worker2         Ready    <none>          5d    v1.31.0
```

The environment's cluster has 1 control plane and 2 worker nodes.

#### 2. Create a `pod-placement` Namespace:

```shell,run
kubectl create namespace pod-placement
```

Expected output:
```text
namespace/pod-placement created
```

#### 3. Create a DaemonSet to see a Pod will run on all Nodes in the cluster:

DaemonSets are used in cases where a Pod should run on every Node in the cluster for specific functions like log collection, storage, monitoring, etc.

```shell,run
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
  namespace: pod-placement
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.0
        ports:
        - containerPort: 80
EOF
```

Expected output:
```text
daemonset.apps/nginx-daemonset created
```

A DaemonSet ensures the Nodes of a cluster runs a copy of the Pod -- but does it?

#### 4. Verify if the `nginx-daemonset` Pod is running in all Nodes in the cluster:

```shell,run
kubectl get pods -n pod-placement -o wide
```

Expected output:
```text
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
nginx-daemonset-qkwsv   1/1     Running   0          53s   10.244.1.86   kind-worker2   <none>           <none>
nginx-daemonset-whsbl   1/1     Running   0          53s   10.244.2.6    kind-worker    <none>           <none>
```

The `kind-control-plane` Node is not running a copy of a Pod from the `nginx-daemonset` DaemonSet.
Let's explore why.

### Taints and Tolerations

Taints allow a Node to repel Pods.
While a toleration on Pods allow the scheduler to schedule Pods with matching taints.

Taints are used to dedicate Nodes to a set of users, hardware, application tiers, and others.
Kubernetes can apply taints based on conditions like memory, disk, or pid pressure.
Taints are also applied if Nodes are not reachable or not ready.

Taints are applied to Nodes imperatively with `kubectl taint node node-name key=value:effect`


#### 5. Check if any Nodes have taints:

```shell,run
kubectl get nodes -o custom-columns=NAME:.metadata.name,"Taint Effect":.spec.taints[].effect,"Taint Key":.spec.taints[].key
```

Expected output:
```taint
NAME                 Taint Effect   Taint Key
kind-control-plane   NoSchedule     node-role.kubernetes.io/control-plane
kind-worker          <none>         <none>
kind-worker2         <none>         <none>
```

The `kind-control-plane` Node has a taint. This is why the `kind-control-plane` Node was not running a copy of the Pod from the `nginx-daemonset` DaemonSet.

In the kind cluster, the Taint Effect is `NoSchedule`.
See below on available Effects and what they do.

Available Effects:
  - `NoExecute`: affects running Pods, if no matching toleration then Pod will be evicted.
  - `NoSchedule`: does not affect running Pods, but no new Pods will be scheduled on the node without a matching toleration.
  - `PreferNoSchedule`: PreferNoSchedule is a "preference" or "soft" version of NoSchedule.

kind defaults to taint the control plane Node.
This means that no new Pod will be able to scheduled on the `kind-control-plane` Node unless it has a matching toleration.

#### 6. Delete the DaemonSet:

```shell,run
kubectl delete daemonset nginx-daemonset -n pod-placement
```

Expected output:
```text
daemonset.apps "nginx-daemonset" deleted
```

If we want to deploy the DaemonSet on the `kind-control-plane` Node with the taint, the DaemonSet would need a matching toleration.

#### 7. Create a DaemonSet so that it tolerates the taint on the control plane Node:

The following command deploys a DaemonSet with a toleration that matches the taint on the `kind-control-plane` Node.

```shell,run
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
  namespace: pod-placement
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: Exists
        effect: NoSchedule
      containers:
      - name: nginx
        image: nginx:1.27.0
        ports:
        - containerPort: 80
EOF
```

Expected output:
```text
daemonset.apps/nginx-daemonset created
```

#### 8. Check if the `kind-control-plane` Node is running a Pod from the DaemonSet with the toleration:

```shell,run
kubectl get pods -o wide -n pod-placement
```

Expected output:
```text
NAME                    READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
nginx-daemonset-9lmjk   1/1     Running   0          37s   10.244.2.84    kind-worker          <none>           <none>
nginx-daemonset-d986m   1/1     Running   0          37s   10.244.1.210   kind-worker2         <none>           <none>
nginx-daemonset-wrmbs   1/1     Running   0          37s   10.244.0.25    kind-control-plane   <none>           <none>
```
The DaemonSet has Pods running on all Nodes, including the `kind-control-plane` Node.

#### 9. Delete the DaemonSet:

```shell,run
kubectl delete daemonset nginx-daemonset -n pod-placement
```

```text
daemonset.apps "nginx-daemonset" deleted
```

### Assigning Pods to Nodes with Node Selector

Labels are used on Nodes to identify and group Nodes.
Examples of Node labels include OS, architecture, hardware, security, or other properties.

Pods have a `nodeSelector` field that is used to match with a Node's labels so that Pods will be scheduled on Nodes with the matching label in the `nodeSelector`.

In the next steps, you will constrain a Pod so that it is restricted to run on a specific Node.
The Pod is considered to be a high security tier app and requires to be run on a Node setup with high security guardrails.

#### 11. Explore Node labels:

```shell,run
 kubectl get nodes --show-labels
```

Expected output:
```text
NAME                 STATUS   ROLES           AGE   VERSION   LABELS
kind-control-plane   Ready    control-plane    5d   v1.31.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kind-worker          Ready    <none>           5d   v1.31.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker,kubernetes.io/os=linux
kind-worker2         Ready    <none>           5d   v1.31.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker2,kubernetes.io/os=linux
```

We see various labels to identify the architecture of the Node, the OS, hostname, and node role.

#### 12. Label the `kind-worker` node to idenfity it has low security:

```shell,run
kubectl label node kind-worker security=low
```

Expected output:
```text
node/kind-worker labeled
```

#### 13. Label the `kind-worker` Node to identity it has high security:

```shell,run
kubectl label node kind-worker2 security=high
```

Expected output:
```text
node/kind-worker2 labeled
```

> [!IMPORTANT]
> The lables do not prevent Pods from being scheduled on Nodes, it is used to identify and group Nodes.

#### 14. Verify the labels on the Nodes

```shell,run
kubectl get nodes --show-labels
```

Expected output:
```text
NAME                 STATUS   ROLES           AGE   VERSION   LABELS
kind-control-plane   Ready    control-plane    5d   v1.31.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kind-worker          Ready    <none>           5d   v1.31.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker,kubernetes.io/os=linux,security=low
kind-worker2         Ready    <none>           5d   v1.31.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker2,kubernetes.io/os=linux,security=high
[ec2-user@ip-10-0-12-225 ~]$
```

Great, the labels are applied to the Nodes.

#### 15. Assign a high security Pod to run on the Node labeled it has high security:

Use the Pod's `nodeSelector` field to assign an application that requires high security to run on a Node with high security, `kind-worker2`.

```shell,run
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-security-app
  namespace: pod-placement
  labels:
    app: high-security
spec:
  containers:
  - name: nginx
    image: nginx:1.27.0
    imagePullPolicy: IfNotPresent
  nodeSelector:
    security: high
EOF
```

Expected output:
```text
pod/high-security-app created
```

#### 16. Verify the Pod is running on the high security Node, `kind-worker2`:

```shell,run
kubectl get pods -n pod-placement -o wide
```

Expected output:
```text
NAME                READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
high-security-app   1/1     Running   0          21s   10.244.1.204   kind-worker2   <none>           <none>
```


#### 17. Cleanup:

Delete the `high-security-app` Pod:

```shell,run
kubectl delete pod high-security-app -n pod-placement
```

Expected output:
```text
pod "high-security-app" deleted
```

Delete the `pod-placement` Namespace:

```shell,run
kubectl delete namespace pod-placement
```

Expected output:
```text
namespace "pod-placement" deleted
```

#### 10. Conclusion

> [!IMPORTANT]
> To summarize, we should always have the the zero-trust pillar of lowering the blast radius of a breach (like a container escape) in mind when we deploy Pods.
Sensitive applications like the Kubernetes control plane components are isolated on a control plane Node by a taint. We can bypass the taints with a matching toleration.
We can place Pods due to the application's sensitivity, hardware requirements, or node-local faciliation duties (like monitoring) by using node selectors to match a label on a Node.

