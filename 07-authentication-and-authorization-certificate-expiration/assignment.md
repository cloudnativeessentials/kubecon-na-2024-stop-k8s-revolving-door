# Authentication and Authorization: Certificate Expiration

This step covers Kubernetes Security Checklist items under Authentication & Authorization:
- Intermediate and leaf certificates have an expiry date no more than 3 years in the future.

Having a certificate expiration with no more than 3 years ensures certificates reflect up-to-date information and use the latest security standards.

### Kubernetes certificates

Kubernetes requires Public Key Infrastructure (PKI) certificates for authentication over TLS.
Kubeadm, the upstream tool used as "best practice fast paths" to create Kubernetes clusters, automatically generates the certificates.

Kubeadm stores most certificates in `/etc/kubernetes/pki` while user account certificates are stored in `/etc/kubernetes`.

The Kubernetes security checklist recommends certificates to have an expiry date no more than 3 years in the future.
It's also mandatory to rotate the certificates before they expire as expired certificates will cause the Kubernetes cluster to be inaccessible or inoperable.

If we used kubeadm to install a cluster, we can simply use the `kubeadm certs check-expiration` command to check the expiration dates. The command shows expiration and residual time for the client certificates in the `/etc/kubernetes/pki` folder and for the client certificate embeddded in the kubeconfig file used by kubeadm.
The output would be similar to:

```text
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Nov 12, 2025 23:36 UTC   364d                                    no
apiserver                  Nov 12, 2025 23:36 UTC   364d            ca                      no
apiserver-etcd-client      Nov 12, 2025 23:36 UTC   364d            etcd-ca                 no
apiserver-kubelet-client   Nov 12, 2025 23:36 UTC   364d            ca                      no
controller-manager.conf    Nov 12, 2025 23:36 UTC   364d                                    no
etcd-healthcheck-client    Nov 12, 2025 23:36 UTC   364d            etcd-ca                 no
etcd-peer                  Nov 12, 2025 23:36 UTC   364d            etcd-ca                 no
etcd-server                Nov 12, 2025 23:36 UTC   364d            etcd-ca                 no
front-proxy-client         Nov 12, 2025 23:36 UTC   364d            front-proxy-ca          no
scheduler.conf             Nov 12, 2025 23:36 UTC   364d                                    no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Nov 12, 2025 23:36 UTC   1y              no
etcd-ca                 Nov 12, 2025 23:36 UTC   1y              no
front-proxy-ca          Nov 12, 2025 23:36 UTC   1y              no
```

But our environment uses `kind`. Like `kind` and many other Kubernetes distriubtions, kubeadm is not available to easily check certificate expiration.

In this lab, we'll explore how `kind` stores certificates and how to check certification validity dates.

### Certificates in the kind

`kind` runs a Kubernetes nodes as individual containers.

Review the nodes in the Kubernetes cluster:

```shell,run
kubectl get nodes
```

Expected output:
```text
NAME                 STATUS     ROLES           AGE     VERSION
kind-control-plane   NotReady   control-plane   3h51m   v1.31.0
kind-worker          NotReady   <none>          3h51m   v1.31.0
kind-worker2         NotReady   <none>          3h51m   v1.31.0
```


Use the container runtime, Docker, to take a look at the containers that Docker is running.

```shell,run
docker ps
```

Expected output:
```text
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                                                                                       NAMES
d116b5b2ec0c   kindest/node:v1.31.0   "/usr/local/bin/entr…"   4 hours ago   Up 4 hours                                                                                               kind-worker
b1232a742c71   kindest/node:v1.31.0   "/usr/local/bin/entr…"   4 hours ago   Up 4 hours                                                                                               kind-worker2
a4d56194e17f   kindest/node:v1.31.0   "/usr/local/bin/entr…"   4 hours ago   Up 4 hours   0.0.0.0:31234-31235->31234-31235/tcp, 0.0.0.0:32042->32042/tcp, 127.0.0.1:34601->6443/tcp   kind-control-plane
```

The output may not be easy to comprehend.
Use the `--format` option to output using a Go template to output a table with the Names and Image columns:

```shell,run
docker ps --format "table {{.Names}}\t{{.Image}}"
```

Expected output:
```text
NAMES                IMAGE
kind-worker2         kindest/node:v1.31.0
kind-worker          kindest/node:v1.31.0
kind-control-plane   kindest/node:v1.31.0
```

Look at the Names columns. There is a `kind-control-plane`, `kind-worker`, and `kind-worker2` containers.
Each container correlates to a node in the kind cluster.

Explore the `/etc/kubernetes/pki/` directory within the `kind-control-plane` container.

```shell,run
docker exec -it kind-control-plane sh -c "ls /etc/kubernetes/pki/"
```

Expected output:
```text
apiserver-etcd-client.crt     apiserver-kubelet-client.key  ca.crt  front-proxy-ca.crt	    front-proxy-client.key
apiserver-etcd-client.key     apiserver.crt		    ca.key  front-proxy-ca.key	    sa.key
apiserver-kubelet-client.crt  apiserver.key		    etcd    front-proxy-client.crt  sa.pub
```

Like kubeadm, `kind` stores the certificates in `/etc/kubernetes/pki` but within the container.
  - `apiserver-etcd-client.crt` is the client certificate to authenticate to etcd.
  - `apiserver-kubelet-client.crt` is the client certificate for kubelet.
  - `apiserver.crt` is the server certificate for the API server endpoint.
  - `front-proxy-client.crt` client certificate for the front-proxy.

Take a look at the `apiserver-etcd-client.crt` certificate.

```shell,run
docker exec -it kind-control-plane sh -c "cat /etc/kubernetes/pki/apiserver-etcd-client.crt"
```

Expected, similar output:
```text
-----BEGIN CERTIFICATE-----
MIIDDzCCAfegAwIBAgIII4i3qanBAw0wDQYJKoZIhvcNAQELBQAwEjEQMA4GA1UE
AxMHZXRjZC1jYTAeFw0yNDA5MzAxNjMxNTdaFw0yNTA5MzAxNjM2NTdaMCUxIzAh
BgNVBAMTGmt1YmUtYXBpc2VydmVyLWV0Y2QtY2xpZW50MIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAs4blbHHMh/apqk66JzQWVpHlJEiyJR22TdkF3lcU
UIEvkFyKb2Gg21VVIByBD6KTH2M189JsD/NfZj0xUHbhYoWBebux7lpBy1osc7cs
jiOoC2AcOq0sHMZFWUBshxI2VIZTKJ7PjsTitjngxEffxuiFW7p0mT9Px7arLHoS
CR/9qQfVgoXd764+QrZXbu10KUddp5KKislqtFkqitgODwtn+ir49jPCRiCr5Sc4
GHdDegF5sH99gUkdiF4Hi5TlRhvZxI6XTuDYzf3f9EGR49T208EHfE6IQaQcoic5
4n+aQAcwSIWvLfkfLqVBLuLYxLJYKR9tiloRjwQyA0oysQIDAQABo1YwVDAOBgNV
HQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYBBQUHAwIwDAYDVR0TAQH/BAIwADAf
BgNVHSMEGDAWgBS4+TpMV0zR3ndpc2cQAa/QT1fbRTANBgkqhkiG9w0BAQsFAAOC
AQEApd7Yz4ZKk1lBa8e/IozN4LdqdzvJdWpW0As+LCw3PJZ9YsP7b1uBL7OWj570
CJ8rvARb1UKgxMchWU9sBKZZc5bxFXuv2KOObM58XCNa2asqjHzzN2dZJ0izoI4t
HYdp/YNz+K5OD87bMhz0IPSPMsMHUNa6RnJAUhd8HQWQjrQRU0b4EZ5cV0USjTm2
38rnSPUvhlyZH5xoglRQb4DQQAAGYz526glKgput/TXYxmtYy97u8LQVnp04pJa6
2jBLvXtlGs4j8YzjQpydkShJ9R/8Pa4XobRLXoNZYeo3J5PDom4327ikGHF6pWB4
t6bZisuAkBhMGdU7IsEvzMee/g==
-----END CERTIFICATE-----
```

Great, you can access the Kubernetes cluster's certificates but we need to find the validatity dates.
In the following steps, you will check the certificates validity dates to verify they expire within 3 years.

### Certificate validity dates
===

#### 1. Use `openssl` to check the certificates validity dates, start with the `apiserver-etcd-client.crt` certificate

`apiserver-etcd-client.crt` is the control-plane's client certificate to authenticate to etcd.

We previously saw the output of the `apiserver-etcd-client.crt` certificate but need a way to output the validity dates.

Both `kind` and kubeadm use x509 format for the certificates.
Use the `openssl x509` command with the `-startdate`, `-enddate` and `-noout` options.
  - The `-startdate` option prints the notBefore field.
  - The `-enddate` option prints the notAfter field.
  - The `-noout` option does not print the certificate.

```shell,run
docker exec -it kind-control-plane sh -c "cat /etc/kubernetes/pki/apiserver-etcd-client.crt" | openssl x509 -startdate -enddate -noout
```

Expected output:
```text
notBefore=Nov 30 16:31:57 2024 GMT
notAfter=Nov 30 16:36:57 2025 GMT
```

The `notBefore` field shows the start of the validity period -- this may be 1-10 minutes before the creation of the certificate to avoid clocks that are not in sync.
The `notAfter` field shows end date of the validity period of the certificate.
The `notAfter` date should not be more than 3 years from when the certificate was created, the `notBefore` date.

The `apiserver-etcd-client.crt` certificate has a 1 year expiration which satisfies the Kubernetes security checklist.

#### 2. Check the `apiserver-kubelet-client.crt` certificate's validity dates:

`apiserver-kubelet-client.crt` is the control-plane's client certificate for kubelet.

```shell,run
docker exec -it kind-control-plane sh -c "cat /etc/kubernetes/pki/apiserver-kubelet-client.crt" | openssl x509 -startdate -enddate -noout
```

Expected output:
```text
notBefore=Nov 30 16:31:57 2024 GMT
notAfter=Nov 30 16:36:57 2025 GMT
```

Great, the `apiserver-kubelet-client.crt` certificate has a 1 year expiration or validity period.

#### 3. Check the validity dates for the `apiserver.crt` certificate:

`apiserver.crt` is the control-plane's server certificate for the API server endpoint.

```shell,run
docker exec -it kind-control-plane sh -c "cat /etc/kubernetes/pki/apiserver.crt" | openssl x509 -startdate -enddate -noout
```

Expected output:
```text
notBefore=Nov 30 16:31:57 2024 GMT
notAfter=Nov 30 16:36:57 2025 GMT
```

The `apiserver.crt` certificate has a 1 year expiration which satisfies the Kubernetes security checklist.

#### 4. Check the validity dates for the `front-proxy-client.crt` certificate:

`front-proxy-client.crt` client certificate for the front-proxy.

```shell,run
docker exec -it kind-control-plane sh -c "cat /etc/kubernetes/pki/front-proxy-client.crt" | openssl x509 -startdate -enddate -noout
```

Expected output:
```text
notBefore=Nov 30 16:31:57 2024 GMT
notAfter=Nov 30 16:36:57 2025 GMT
```

The `front-proxy-client.crt` certificate has a 1 year expiration which satisfies the Kubernetes security checklist.

### etcd certificates

etcd certificates are used for encrypted traffic between etcd members and clients and are signed by the etcd-signer and come from a certificate authority (CA) generated by the Kubernetes distribution's bootstrap process.

The following certificates are generated and used by etcd and other processes that communicate with etcd:
  - Peer certificates: for communication between etcd members.
  - Client certificates: for encrypted server-client communication.
  - Server certificates: for authenticating client requests by the etd server

etcd certificates in kind are stored in `/etc/kubernetes/pki/etcd` within the control-plane container.
Take a look at the certificates in the `/etc/kubernetes/pki/etcd` within the control-plane container:

```shell,run
docker exec -it kind-control-plane sh -c "ls /etc/kubernetes/pki/etcd"
```

Expected output:
```text
ca.crt	ca.key	healthcheck-client.crt	healthcheck-client.key	peer.crt  peer.key  server.crt	server.key
```

`server.crt` is the server certificate for the etcd server.

#### 5. Check the validity dates for etcd's `healthcheck-client.crt` certificate:
```shell,run
docker exec -it kind-control-plane sh -c "cat /etc/kubernetes/pki/etcd/healthcheck-client.crt" | openssl x509 -startdate -enddate -noout
```

Expected output:
```text
notBefore=Nov 30 16:31:57 2024 GMT
notAfter=Nov 30 16:36:57 2025 GMT
```

etcd's `healthcheck-client.crt` certificate has a 1 year expiration which satisfies the Kubernetes security checklist.

#### 6. Check the validity dates for etcd's `server.crt` certificate:

```shell,run
docker exec -it kind-control-plane sh -c "cat /etc/kubernetes/pki/etcd/server.crt" | openssl x509 -startdate -enddate -noout
```

```text
notBefore=Nov 30 16:31:57 2024 GMT
notAfter=Nov 30 16:36:57 2025 GMT
```

etcd's `server.crt` certificate has a 1 year expiration which satisfies the Kubernetes security checklist.

#### 7. Conclusion

> [!IMPORTANT]
> To summarize, we confirmed our cluster's certificates expire within 3 years. Certificates should be rotated before the certificate expires. When using kubeadm, kubeadm renews the certificates when the control plane is upgraded.