# Network Security

This step covers Kubernetes Security Checklist items under Network Security:
- Ingress and egress network policies are applied to all workloads in the cluster
- Default network policies within each namespace, selecting all pods, denying everything, are in place

In this step, we will create several namespaces, test connection between
workloads in various namespaces, create a default deny all Network Policy,
create specific ingress and egress Network Policies and test along the way. As a
bonus, we will learn a little bit about Kubernetes DNS records.

### Namespace and workload setup
===

Create two namespaces and a workload in each namespace to test Network Policies.

#### 1. Create two Namespaces `source` and `target`:

First create the `source` namespace:
```shell,run
kubectl create namespace source
```

Expected output:
```text
namespace/source created
```

Then create the `target` namespace:
```shell,run
kubectl create namespace target
```

Expected output:
```text
namespace/target created
```

#### 2. Create a Pod named `nginx-source` in the `source` Namespace:

```shell,run
kubectl run nginx-source --namespace source --image nginx:1.27.0
```

Expected output:
```text
pod/nginx-source created
```

#### 3. Create a Pod named `nginx-target` in the `target` Namespace:

```shell,run
kubectl run nginx-target --namespace target --image nginx:1.27.0
```

Expected output:
```text
pod/nginx-target created
```

#### 4. Verify the Pods are running:

For the Pod in the `source` namespace:
```shell,run
kubectl get pods -n source
```

Expected output:
```text
NAME           READY   STATUS    RESTARTS   AGE
nginx-source   1/1     Running   0          2m31s
```

For the pod in `target` namespace:
```shell,run
kubectl get pods -n target
```

Expected output:
```text
NAME           READY   STATUS    RESTARTS   AGE
nginx-target   1/1     Running   0          2m39s
```

Great, the Pods are running.

#### 5. Create a Service for `nginx-source` Pod and a Service for the `nginx-target` Pod to expose the Pods which will give use a DNS entry to access the Pods:

For the `target` namespace:
```shell,run
kubectl expose pod nginx-target --port 80 --namespace target
```

Expected output:
```text
service/nginx-target exposed
```

For the `source` namespace:
```shell,run
kubectl expose pod nginx-source --port 80 --namespace source
```

Expected output:
```text
service/nginx-source exposed
```

#### 6. Verify the Services were created:

For the `source` namespace:
```shell,run
kubectl get services -n source
```

Expected output:
```text
NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
nginx-source   ClusterIP   10.96.9.250   <none>        80/TCP    3m12s
```

For the `target` namespace:
```shell,run
kubectl get services -n target
```

Expected output:
```text
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-target   ClusterIP   10.96.131.133   <none>        80/TCP    3m35s
```

#### 7. From the `nginx-source` Pod in the `source` Namespace, reach out to the `nginx-target` Pod in the `target` Namespace

> [!NOTE]
> 	We are using the Service DNS record which are in the format `<service>.<namespace>.svc.cluster.local` by default

```shell,run
kubectl exec nginx-source --namespace source -it -- curl nginx-target.target.svc.cluster.local | head -n 4
```

Expected output:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

That works! The `nginx-source` Pod in the `source` Namespace can reach the `nginx-target` Pod in the `target` Namespace because there are no Network Policies in place.

### Network Policies
====

In the following steps, we will create a default deny all Network Policy for the `target` Namespace then create Network Policies to allow specific ingress and egress to the `target` Namespace.

#### 8. Create a default deny all Network Policy for the target Namespace:

```shell,run
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

Expected output:
```text
networkpolicy.networking.k8s.io/default-deny-all created
```

#### 9. Run the same test , from the `nginx-source` Pod try to reach the `nginx-target` Pod:

```shell,run
kubectl exec nginx-source --namespace source -it -- curl --max-time 10 nginx-target.target.svc.cluster.local | head -n 4
```

We will see that the command times out in 10s.
Expected output:
```text
curl: (28) Failed to connect to 10.244.1.63 port 80 after 10000 ms: Couldn't connect to server
command terminated with exit code 28
```

With the deny all Network Policy for the `target` Namespace, traffic is denied to Pods in the `target` Namespace.

#### 10. Let's create a Network Policy to allow ingress from the `source` Namespace to the `target` Namespace

```shell,run
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-source-to-target
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: source
EOF
```

Expected output:
```text
networkpolicy.networking.k8s.io/ingress-source-to-target created
```

#### 11. Run the same test. From the `nginx-source` Pod try to reach the `nginx-target` Pod:

```shell,run
kubectl exec nginx-source --namespace source -it -- curl --max-time 10 nginx-target.target.svc.cluster.local | head -n 4
```

Expected output:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

That works! With a Network Policy in place that allows inress traffic from the `source` Namespace to the `target` Namespace. The `nginx-source` Pod can reach the `nginx-target` Pod.

#### 12. Let's test if ingress is allowed from a new namespace

Create a `new-source` Namespace:

```shell,run
kubectl create namespace new-source
```

Expected output:
```text
namespace/new-source created
```

#### 13. Create a workload Pod called `nginx-new-source` in the `new-source` Namespace:

```shell,run
kubectl run nginx-new-source --namespace new-source --image nginx:1.27.0
```

Expected output:
```text
pod/nginx-new-source created
```

Wait for the Pod to be ready
```shell,run
kubectl wait --for=condition=ready pod nginx-new-source -n new-source
```

#### 14. Run a test, from `nginx-new-source` Pod try to reach the `nginx-target` Pod:

```shell,run
kubectl exec nginx-new-source --namespace new-source -it -- curl --max-time 10 nginx-target.target.svc.cluster.local | head -n 4
```

Expected output:
```text
curl: (28) Connection timed out after 10000 milliseconds
command terminated with exit code 28
```

Workloads from the `new-source` Namespace can not reach the `target` Namespace.

#### 15. Repeat the test from the `source` Namespace to verify the ingress Network Policy works. From the `nginx-source` Pod try to reach `nginx-target` Pod:

```shell,run
kubectl exec nginx-source --namespace source -it -- curl --max-time 10 nginx-target.target.svc.cluster.local | head -n 4
```

Expected output:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

We confirmed that ingress from only the `source` Namespace is allowed into the `target` Namespace.

#### 16. Since we have a default deny Network Policy for both ingress and egress for the `target` Namespace and only have an ingress Network Policy from the `source` Namespace to the `target` Namespace. Egress traffic from the `target` Namespace to anywhere including the `source` Namespace should be blocked -- let's test this.

From the `nginx-target` Pod, try to reach the `nginx-source` Pod:

```shell,run
kubectl exec nginx-target --namespace target -it -- curl --max-time 10 nginx-source.source.svc.cluster.local | head -n 4
```

Expected output:
```text
curl: (28) Connection timed out after 10000 milliseconds
command terminated with exit code 28
```

#### 17. Create an egress policy for the `target` Namespace to allow egress to the `source` Namespace:

```shell,run
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-target-to-source
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: source
EOF
```

Expected output:
```text
networkpolicy.networking.k8s.io/egress-target-to-source created
```

#### 18. Repeat the egress test from the `nginx-target` Pod to the `nginx-source` Pod:

> [!NOTE]
>    We are using the `nginx-source` Service DNS

```shell,run
kubectl exec nginx-target --namespace target -it -- curl --max-time 10 nginx-source.source.svc.cluster.local | head -n 4
```

Expected output:
```text
curl: (28) Connection timed out after 10000 milliseconds
command terminated with exit code 28
```
The error is not expected unless we realize how the workloads lookup DNS records.

#### 19. Let's repeat the egress test but with the `nginx-source` Pod's virtual IP. We can find the virtual IP of the `nginx-source` Pod with:

```shell,run
kubectl get pods -o wide --namespace source
```

Expected output:
```text
NAME           READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
nginx-source   1/1     Running   0          1h   10.244.1.156   kind-worker   <none>           <none>
```

In this scenario, the `nginx-source` Pod's virtual IP is 10.244.1.156. Your virtual IP will vary.

#### 20. Use the `nginx-source` Pod's virutal IP to try to reach it from the `nginx-target` Pod:

> [!NOTE]
> 		Note the subshell used outputs the `ngixn-source` Pod's virtual IP

```shell,run
kubectl exec nginx-target --namespace target -it -- curl --max-time 10 $(kubectl get pods nginx-source -n source --template={{.status.podIP}}) | head -n 4
```

Expected output:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

It works! So using the Pod's virtual IP works but using the Service's DNS entry does not work but worked when the Namespace (`source`) can access other Namespaces.

#### 21. Pods need access to the `kube-system` Namespace to get Kubernetes cluster DNS records.

Create a Network Policy in the `target` Namespace so workloads can access the `kube-system` Namespace:

```shell,run
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-target-to-kube-system
  namespace: target
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
EOF
```

Expected output:
```text
networkpolicy.networking.k8s.io/allow-all-target-to-kube-system created
```

#### 22. Repeat egress test from the `nginx-target` Pod to the `nginx-source` Pod using the `nginx-source` Service's DNS record:

```shell,run
kubectl exec nginx-target --namespace target -it -- curl --max-time 10 nginx-source.source.svc.cluster.local | head -n 4
```

Expected output:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

#### 23. Conclusion

> [!IMPORTANT]
> To summarize we created a default deny all for ingress and egress for the `target` Namespace.
To allow traffic from the `source` Namespace to the `target` Namespace, we created a Network Policy in the `target` Namespace to allow ingress from the `source` Namespace.
To allow traffic from the `target` Namespace to the `source` Namespace we created a Network Policy in the `target` Namespace to allow egress to the `source` Namespace.
But to use Kubernets cluster DNS records, we also needed a Network Policy in the `target` Namespace to allow igress and egress to the `kube-system` Namespace to lookup cluster DNS records.