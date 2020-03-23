
# Kubernetes (on virtual machines) The Hard Way
## Introduction

This tutorial goes through deploying a local Kubernetes clusters using virtual machines. It is heavily based on Kelsey Hightower's [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way), which was designed for deploying a Kubernetes cluster on Google Kubernetes Engine.

At the start of each section, some comments will be included to explicitly point out any difference between this guide and the original.

## Major differences
* We will be deploying the Kubernetes nodes on local VMs instead of the cloud
* We will only be deploying 1 controller node and 2 worker nodes. As such, we will not be covering the load balancer for the Kubernetes front end.
* [Kube-Router](https://www.kube-router.io/) will be used to provide pod/service networking
* Included section on using [MetalLB](https://metallb.universe.tf/) as bare-metal load balancer to expose loadbalancer type service

## Labs

* [Prerequisites](docs/01-prerequisites.md)
* [Installing the Client Tools](docs/02-client-tools.md)
* [Provisioning Compute Resources](docs/03-compute-resources.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authentication](docs/05-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
* [Bootstrapping the etcd Cluster](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Deploying the DNS Cluster Add-on](docs/12-dns-addon.md)
* [Smoke Test](docs/13-smoke-test.md)
* [Bare-metal load balancer](docs/bare-metal-load-balancer.md)
