# multus-dynamic-networks-controller
This project provides a [Kubernetes](https://kubernetes.io/) [controller](https://kubernetes.io/docs/concepts/architecture/controller/)
granting the ability to plug/unplug network interfaces to / from running pods.

This controller extends the [multus-cni](https://github.com/k8snetworkplumbingwg/multus-cni) functionality, by
listening to pod's network selection elements (i.e. the pod `k8s.v1.cni.cncf.io/networks` annotation); whenever those
change (adding / removing network selection elements), it will invoke the corresponding delegate effectively adding
(or removing) a network interface to a running pod.

Please refer to the
[multus-cni docs](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md#creating-a-pod-that-attaches-an-additional-interface)
for more information on how additional interfaces are added to a pod.

**Disclaimer:** this repo is work in progress, and it currently does **not** provide the advertised feature-set.
The MVP version **will**. You can track progress - and our definition of MVP - via
[the project board](https://github.com/users/maiqueb/projects/1).

## Usage

### Requirements
- a running Kubernetes cluster
- multus deployed (in [thick-plugin](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/thick-plugin.md#multus-thick-plugin) mode) in the Kubernetes cluster
- the `kubectl` binary

### Installation
Use the provided [manifest](manifests/dynamic-networks-controller.yaml) to install the controller on your cluster:

```bash
kubectl apply -f manifests/dynamic-networks-controller.yaml
```

### Removal
Use `kubectl` to remove the controller from your cluster:

```bash
kubectl delete -f manifests/dynamic-networks-controller.yaml
```

### Adding / removing network interfaces
To add (or remove...) network interfaces from a running pod, the user should
simply edit the running pod network selection elements - i.e. the `k8s.v1.cni.cncf.io/networks`
annotation.

Thus, if we had this running pod (and `NetworkAttachmentDefinition`):
```yaml
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan1-config
spec:
  config: '{
            "cniVersion": "0.4.0",
            "plugins": [
                {
                    "type": "macvlan",
                    "capabilities": { "ips": true },
                    "master": "eth1",
                    "mode": "bridge",
                    "ipam": {
                        "type": "static"
                    }
                }, {
                    "type": "tuning"
                } ]
        }'
---
apiVersion: v1
kind: Pod
metadata:
  name: macvlan1-worker1
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            {
                "name": "macvlan1-config",
                "ips": [ "10.1.1.11/24" ]
            }
    ]'
  labels:
    app: macvlan
spec:
  containers:
  - name: macvlan-worker1
    image: docker.io/library/alpine:latest
    command: ["/bin/sleep", "10000"]
```

We would run this example yaml to **add** an interface from it:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: macvlan1-worker1
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            {
                "name": "macvlan1-config",
                "ips": [ "10.1.1.11/24" ]
            },
            {
                "name": "macvlan1-config",
                "interface": "ens4",
            }
    ]'
  labels:
    app: macvlan
spec:
  containers:
    - name: macvlan-worker1
      image: docker.io/library/alpine:latest
      command: ["/bin/sleep", "10000"]
```

And we would run this example yaml to **remove** an interface to it:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: macvlan1-worker1
  annotations:
    k8s.v1.cni.cncf.io/networks: '[]' # this will remove all networks from the pod
  labels:
    app: macvlan
spec:
  containers:
    - name: macvlan-worker1
      image: docker.io/library/alpine:latest
      command: ["/bin/sleep", "10000"]
```

## Configuration
The `multus-dynamic-networks-controller` configuration is encoded in JSON, and allows the following keys:

- `"criSocketPath"`: specify the path to the CRI socket. Defaults to `/run/containerd/containerd.sock`.
- `"criType"`: either `crio` or `containerd`. Defaults to `containerd`.
- `"multusSocketPath"`: specify the path to the multus socket. Defaults to `/var/run/multus-cni/multus.sock`.

The configuration is defined in a `ConfigMap`, which is defined in the
[installation manifest](manifests/dynamic-networks-controller.yaml), and mounted into the pod.

The name of the `ConfigMap` is `dynamic-networks-controller-config`.

## Developer Workflow
Below you can find information on how to push local code changes to a kind cluster.

- change code :)
- start up a kind cluster. I've tested using the [multus repo e2e kind cluster](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/e2e/setup_cluster.sh).
- build image: `IMAGE_REGISTRY=localhost:5000/maiqueb OCI_BIN=podman make img-build`. **NOTE:** this assumes podman is used. `docker` is the default `OCI_BIN`.
- push image to local registry: `podman push localhost:5000/maiqueb/multus-dynamic-networks-controller`
- update manifests to use the generated image: `IMAGE_REGISTRY=localhost:5000/maiqueb make manifests`
- deploy the controller: `kubectl apply -f manifests/dynamic-networks-controller.yaml`
