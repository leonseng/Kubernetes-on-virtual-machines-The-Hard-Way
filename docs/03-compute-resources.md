# Provisioning Compute Resources

[Original link](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md)

Instead of setting up the compute and networking on GCloud, we will be spinning up VMs locally. Due to local resource constaints, we will only be using 1 controller and 2 workers.
___

## IP Design

Nodes | eth0 - OAM IP | eth1 - INTERNAL (Cluster) IP
:---: | :---: | :---:
controller-0 | 192.168.238.100 | 10.240.0.10
worker-0 | 192.168.238.110 | 10.240.0.20
worker-1 | 192.168.238.111 | 10.240.0.21
Director* | 192.168.238.10 | 10.240.0.2

\* The Director is not part of the cluster but is used for external interactions with the cluster

Additional subnets:

Description | IP ranges
--- | :---:
Service CIDR | 10.32.0.0/24
Pod CIDR | 10.200.0.0/16
LoadBalancer Public IP addresses | 192.168.100.0/24

## VM Setup

4 VMs have been created using the CentOS 7 Minimal ISO image. Each VM has been configured as follows:

* Attach each VM to the following networks:
  * eth0 - OAM (192.168.238.0/24)
  * eth1 - INTERNAL (10.240.0.0/24)
* Update `/etc/hosts` file with all cluster nodes
  ```
  $ cat /etc/hosts
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

  10.240.0.10  controller-0
  10.240.0.20  worker-0
  10.240.0.21  worker-1
  10.240.0.22  worker-2
  ```
* Disable SELINUX else containerd can't create containers
* Disable firewalld
  ```
  $ sudo systemctl stop firewalld
  $ sudo systemctl disable firewalld
  ```
* (Optional) If your setup is behind a corporate proxy, you will need to setup a proxy for K8s nodes to download images from Docker registry. We will be using a CNTLM proxy on the director, listening on `http://192.168.238.10:3128/`

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
