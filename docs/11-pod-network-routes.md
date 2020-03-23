# Provisioning Pod Network Routes

[Original link](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/11-pod-network-routes.md)

* Modified this section to set up kube-router instead
___

At this point pods can not communicate with other pods running on different nodes due to missing routes.

In this lab you will set up `kube-router` which automatically managed pod-to-pod communications across nodes.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## Prerequisites

From the [kube-router guide](https://github.com/cloudnativelabs/kube-router/blob/master/docs/generic.md#kube-router-on-generic-clusters):

* All pod networking CIDRs are allocated by `kube-controller-manager`, hence it needs to be configured to allocate pod CIDRs by passing --allocate-node-cidrs=true flag and providing a cluster-cidr (i.e. by passing --cluster-cidr=10.32.0.0/12 for e.g.).
* Both kube-apiserver and kubelet must be run with `--allow-privileged=true` option
  ```
  --network-plugin=cni
  --cni-conf-dir=/etc/cni/net.d
  ```

## Setup kube-router

```
KUBERNETES_PUBLIC_ADDRESS=$(grep "controller-0" /etc/hosts | awk '{print $1}') \
CLUSTERCIDR=10.200.0.0/16 \
APISERVER=https://$KUBERNETES_PUBLIC_ADDRESS:6443 \
sh -c 'curl https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/generic-kuberouter-all-features.yaml -o - | \
sed -e "s;%APISERVER%;$APISERVER;g" -e "s;%CLUSTERCIDR%;$CLUSTERCIDR;g"' | \
kubectl apply -f -
```

Verify that `kube-router` daemonset is successfully created

```
kubectl get pods --selector=k8s-app=kube-router -n kube-system
```

```
NAME                READY   STATUS    RESTARTS   AGE
kube-router-vv496   1/1     Running   9          11d
kube-router-z6d6w   1/1     Running   8          11d
```

## Routes

Get the POD CIDRs of all worker nodes:

```
kubectl get nodes -o=jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.spec.podCIDR}{'\n'}"
```

```
worker-0        10.200.1.0/24
worker-1        10.200.0.0/24
```

Verify that the routes to POD CIDRs are installed on the workers. **Run the following commands on the worker nodes**:

```
ip route show
```

There should a route for a local pod CIDR through `kube-bridge`, and a route for the remote pod CIDR through the node's network interface

```
...
10.200.0.0/24 via 10.240.0.21 dev eth1 proto 17
10.200.1.0/24 dev kube-bridge proto kernel scope link src 10.200.1.1
...
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
