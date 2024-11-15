# Seccomp

## Using RuntimeDefault as a seccomp profile
Seccomp (secure computing mode) is a security feature in the Linux kernel that
can restrict the syscalls a process can make. In "filter" mode, it operates by
defining a classic BPF filter that dictates which syscalls are allowed and which
are denied for a particular process or set of processes. Seccomp BPF can operate
in various modes, including:
- `SCMP_ACT_ALLOW`: Only explicitly allowed syscalls are permitted.
- `SCMP_ACT_ERRNO`: Specific syscalls are denied, and all others are allowed.

### What is the `RuntimeDefault` Seccomp Profile?

The `RuntimeDefault` seccomp profile is a predefined profile provided by container runtimes that enforces a standard set of syscall restrictions on containers. When a container is started with the `RuntimeDefault` profile, it inherits the default security settings defined by the container runtime.

**Key Features of `RuntimeDefault`:**
- **Predefined Security:** The `RuntimeDefault` profile includes a set of default rules that are designed to provide a reasonable balance between security and functionality.
- **No Need for Custom Profiles:** Using `RuntimeDefault` simplifies security management as users do not need to create and maintain custom seccomp profiles.
- **Consistent Across Environments:** The default settings are consistent across different environments, ensuring a baseline level of security.

### Why Use `RuntimeDefault`?

1. **Ease of Use:** Since it is a predefined profile, you do not need to create and manage custom seccomp profiles. This reduces the complexity of security configuration.
2. **Baseline Security:** It provides a default level of security that is suitable for many use cases, helping to protect containers from common syscall-related vulnerabilities.
3. **Compatibility:** It is designed to work seamlessly with most containerized applications, minimizing the risk of breaking functionality due to overly restrictive syscall policies.

### How to Apply the `RuntimeDefault` Profile

You can apply the `RuntimeDefault` seccomp profile to a Kubernetes Pod by specifying it in the pod's security context. Here’s an example of how to create a pod that uses the `RuntimeDefault` seccomp profile

Let's walk through a practical example of applying the `RuntimeDefault` seccomp profile in a Kubernetes workshop.

### Steps to use the RuntimeDefault seccomp profile
===

#### 1. **Create a Namespace for Seccomp Testing:**

```sh,run
kubectl create namespace seccomp-test
```

#### 2. **Deploy a Pod without the `RuntimeDefault` Seccomp Profile:**

```sh,run
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: no-runtime-default
  namespace: seccomp-test
spec:
  containers:
  - name: no-runtime-default-demo
    image: busybox
    command: ["sh", "-c", "echo Hello from the no runtime default seccomp demo; sleep 3600"]
EOF
```

The output should be similar to:
```text
pod/no-runtime-default created
```

#### 3. **Deploy a Pod with the `RuntimeDefault` Seccomp Profile:**

```sh,run
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: runtime-default
  namespace: seccomp-test
spec:
  containers:
  - name: runtime-default-demo
    image: busybox
    command: ["sh", "-c", "echo Hello from the runtime default seccomp demo; sleep 3600"]
    securityContext:
      seccompProfile:
        type: RuntimeDefault
EOF
```

The output should be similar to:
```text
pod/runtime-default created
```

#### 4. **Verify the Pod Status:**

Verify that the pod is deployed and running
```sh,run
kubectl get pods -n seccomp-test
```

The output should be similar to:
```text
NAME                   READY   STATUS    RESTARTS   AGE
no-runtime-default     1/1     Running   0          8s
runtime-default        1/1     Running   0          5s
```

Now, describe the pod to see the seccomp profile setting
```sh,run
kubectl describe pod runtime-default -n seccomp-test
```

The output should be similar to:
```text
Name:             runtime-default
Namespace:        seccomp-test
Priority:         0
Service Account:  default
Node:             kind-worker/172.18.0.2
Start Time:       Wed, 07 Aug 2024 18:45:53 +0000
Labels:           <none>
Annotations:      <none>
Status:           Running
IP:               10.244.1.5
IPs:
  IP:  10.244.1.5
Containers:
  runtime-default-demo:
    Container ID:    containerd://b492fde387731ac6aa1248e6472b8911551ee218f0a321f2818411fa780f7e49
    Image:           busybox
    Image ID:        docker.io/library/busybox@sha256:9ae97d36d26566ff84e8893c64a6dc4fe8ca6d1144bf5b87b2b85a32def253c7
    Port:            <none>
    Host Port:       <none>
    SeccompProfile:  RuntimeDefault
    Command:
      sh
      -c
      echo Hello from the runtime default seccomp demo; sleep 3600
[...]
```

#### 5. **Exec into the Pod to Test without restricted syscalls:**

Let's test the seccomp feature by running a restricted call from the pod.
First, exec into the pod a restricted syscall using the followig command:

```sh,run
kubectl exec -it no-runtime-default -n seccomp-test -- sh
```

Type `id` to check your current user
```sh,run
id
```

The output should be similar to

```text
uid=0(root) gid=0(root) groups=0(root),10(wheel)
```

Then run `unshare --user` with
```sh,run
unshare --user
```

You should see the shell prompt changing from `#` (root) to `$` (regular user),
and if you check your id with the `id` cmd:

```sh,run
id
```

The output should be similar to:

```text
uid=65534(nobody) gid=65534(nobody) groups=65534(nobody),65534(nobody)
```

You can see that the `unshare` syscall was successful.

Type `exit` to quit the shell session in the user namespace, and then exit to
quit the shell session from the Pod.

```sh,run
exit
exit
```

#### 6. **Exec into the Pod to Test Restricted Syscalls:**

Let's test the seccomp feature by running a restricted call from the pod.
First, exec into the pod a restricted syscall using the followig command:

```sh,run
kubectl exec -it runtime-default -n seccomp-test -- unshare --user
```

In simpler terms, the unshare --user command allows you to create a new user
namespace, in which you could be root (when using -r), but only within that
isolated space

The output should be similar to:

```text
unshare: unshare(0x10000000): Operation not permitted
```

So in this case, the `RuntimeDefault` seccomp profile prevented you to use the
`unshare` syscall and thus the `unshare` binary utility got an "Operation not
permitted" error from the syscall.

More about `unshare --user` command,
```text
NAME
      unshare - run program in new namespaces

SYNOPSIS
      unshare [options] [program [arguments]]

DESCRIPTION
      The unshare command creates new namespaces (as specified by the
      command-line options described below) and then executes the specified
      program. If program is not given, then "${SHELL}" is run (default:
      /bin/sh).

      By default, a new namespace persists only as long as it has member
      processes. A new namespace can be made persistent even when it has no
      member processes by bind mounting /proc/pid/ns/type files to a filesystem
      path. A namespace that has been made persistent in this way can
      subsequently be entered with nsenter(1) even after the program terminates
      (except PID namespaces where a permanently running init process is
      required). Once a persistent namespace is no longer needed, it can be
      unpersisted by using umount(8) to remove the bind mount. See the EXAMPLES
      section for more details.
OPTIONS
  -r, --map-root-user
          Run the program only after the current effective user and group IDs
          have been mapped to the superuser UID and GID in the newly created
          user namespace. This makes it possible to conveniently gain
          capabilities needed to manage various aspects of the newly created
          namespaces (such as configuring interfaces in the network namespace or
          mounting filesystems in the mount namespace) even when run
          unprivileged. As a mere convenience feature, it does not support more
          sophisticated use cases, such as mapping multiple ranges of UIDs and
          GIDs. This option implies --setgroups=deny and --user. This option is
          equivalent to --map-user=0 --map-group=0.
          -r
  -U, --user[=file]
        Create a new user namespace. If file is specified, then the namespace is made persistent by creating a bind mount at file.
```

### Conclusion
===

> [!IMPORTANT]
> By following these steps, you can see how the `RuntimeDefault` seccomp profile
helps in restricting certain syscalls, thus enhancing the security of your
Kubernetes workloads with little efforts. For further learning, you can explore
this [tutorial](https://kubernetes.io/docs/tutorials/security/seccomp/).