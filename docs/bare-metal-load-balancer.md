# Bare-metal load balancer
This guide assumes you already have the NGINX deployment set up across the worker nodes. These pods use the [stenote/nginx-hostname](https://hub.docker.com/r/stenote/nginx-hostname/) image which enables us to quickly determine which node we are hitting.

```
kubectl get pods --selector=app=nginx -o wide
```
> output
```
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-5bf9cb8466-qtwbl   1/1     Running   1          24h   10.200.0.41   worker-1   <none>           <none>
nginx-deployment-5bf9cb8466-xjm28   1/1     Running   1          24h   10.200.1.24   worker-0   <none>           <none>
```

# Introduction

[MetalLB](https://metallb.universe.tf/) is a load-balancer solution for Kubernetes deployments that does not run on a cloud provider. It is responsible for:

* allocating IP addresses for Kubernetes load-balancer services
* setting up `metallb-system/speaker` daemonset on each worker node to advertise the IP addresses to an external router over BGP

We will be setting up a [BIRD](https://bird.network.cz/) router to peer with the `speaker` pods and learn the routes to the cluster service.

# Details
Load balancer external IP range: 192.168.100.0/24

BGP design:
BGP router | Node | AS number | Peering IP
--- | --- | --- | ---
BIRD router | director| 65000 | 10.240.0.2
metallb-system/speaker | worker-0 | 65001 | 10.240.0.20
metallb-system/speaker | worker-1 | 65001 | 10.240.0.21


# Setup MetalLB
Follow the [Installation By Manifest](https://metallb.universe.tf/installation/) instructions from the [MetalLB](https://metallb.universe.tf/) site

```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
```

Configure MetalLB by creating a configMap. Note that `data.config.address-pools[*].addresses` can be replaced with your own list of external IP addresses.

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - my-asn: 65001
      peer-asn: 65000
      peer-address: 10.240.0.2
    address-pools:
    - name: my-ip-space
      avoid-buggy-ips: true
      protocol: bgp
      addresses:
      - 192.168.100.0/24
EOF
```

Verify MetalLB pods are running
```
kubectl get pods -n metallb-system -o wide
```
> output
```
NAME                          READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
controller-75689c4588-vhfrd   1/1     Running   0          6h17m   10.200.1.28   worker-0   <none>           <none>
speaker-fxwwc                 1/1     Running   0          6h17m   10.240.0.21   worker-1   <none>           <none>
speaker-gblbc                 1/1     Running   0          6h17m   10.240.0.20   worker-0   <none>           <none>

```

Expose a load balancer service for the NGINX deployment
```
kubectl expose deployment nginx-deployment --type=LoadBalancer --name=nginx-lb
```

Verify the load balancer service has been created
```
k get svc nginx-lb
```
> output
```
NAME       TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
nginx-lb   LoadBalancer   10.32.0.233   192.168.100.1   80:30358/TCP   3h42m
```

# Setup BIRD BGP router
Install BIRD
```
sudo yum install bird
```

Configure BIRD as a BGP router peered with the cluster worker nodes over eBGP (different AS numbers). Modify BIRD config in `/etc/bird.conf` with the following:
```
cat <<EOF | sudo tee /etc/bird.conf
# Override router ID
router id 10.240.0.2;

# This protocol is needed to get the interfaces of the router.
protocol device {
    scan time 10;
};

protocol kernel {
    scan time 60;
    import none;
    export all;   # Actually insert routes into the kernel routing
}

template bgp k8s_worker {
    local as 65000;
    import all;
}

protocol bgp k8s_worker_0 from k8s_worker {
    neighbor 10.240.0.20 as 65001;
}

protocol bgp k8s_worker_1 from k8s_worker {
    neighbor 10.240.0.21 as 65001;
}
EOF
```

Start the BIRD service
```
sudo systemctl enable bird
sudo systemctl start bird
```

# Verification
Verify that the BGP connection is established
```
sudo birdc show protocols | grep BGP
```
> output
```
k8s_worker_0 BGP      master   up     14:30:39    Established
k8s_worker_1 BGP      master   up     14:30:39    Established
```

Verify routes to exposed service are received by BIRD router
```
sudo birdc show route
```
> output
```
BIRD 1.6.8 ready.
192.168.100.1/32   via 10.240.0.20 on eth1 [k8s_worker_0 16:08:11] * (100) [AS65001?]
                   via 10.240.0.21 on eth1 [k8s_worker_1 16:08:09] (100) [AS65001?]
```

Verify routes to exposed service are installed into the kernel routing table
```
ip route show 192.168.100.1
```
> output
```
192.168.100.1 via 10.240.0.20 dev eth1 proto bird
```

Verify load-balancing from outside the cluster
```
for i in {1..4};do curl -s 192.168.100.1;done
```
> output
```
nginx-deployment-5bf9cb8466-xjm28
nginx-deployment-5bf9cb8466-qtwbl
nginx-deployment-5bf9cb8466-xjm28
nginx-deployment-5bf9cb8466-qtwbl
```