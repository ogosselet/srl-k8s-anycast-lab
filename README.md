# Nokia SR Linux Kubernetes Anycast Lab

In this lab we will explore a topology consisting of a Leaf/Spine [SR Linux](https://learn.srlinux.dev/) Fabric connected to a Kubernetes Cluster.

Our k8s Cluster will feature Cilium. This will unlock the possibility to have **anycast** services in our fabric.

To deploy this lab we will use [Containerlab](https://containerlab.dev/) which help us to effortlessly create complex network topologies and validate features, scenarios... And also, [Minikube](https://minikube.sigs.k8s.io/) which is an open-source tool that facilitates running Kubernetes clusters locally to quickly test and experiment with containerized applications.

The end service we will use on top of the kubernetes cluster is a Nginx HTTP echo server. This service will be deployed and exposed in all the k8s nodes. With simulated clients, we will verify how traffic is distributed among the different nodes/pods.

For a detailed walkthrough of this lab please check the [SR Linux blog](https://learn.srlinux.dev/blog/2023/exposing-kubernetes-services-to-sr-linux-based-ip-fabric-with-anycast-gateway-and-metallb/).

## Topology

<p align="center">
 <img src="images/topology.svg" width="600">
</p>

## Goal

Demonstrate Cilium kubernetes CNI load balancing scenario in a Containerlab+Minikube Lab.

## Features

- Containerlab topology
- Minikube kubernetes cluster (3 nodes)
- Cilium CNI integration (Native routing mode)
- Preconfigured Leaf/Spine Fabric: 2xSpine, 4xLeaf SR Linux switches
- Anycast services
- Linux clients to simulate connections to k8s services (4 clients)

## Requirements

- [Containerlab](https://containerlab.dev/)
- [minikube](https://minikube.sigs.k8s.io)
- [Docker](https://docs.docker.com/engine/install/)
- [SR Linux Container image](https://github.com/nokia/srlinux-container-image)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [cilium](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)

## Deploying the lab

```bash
# clone this repository
git clone https://github.com/srl-labs/srl-k8s-anycast-lab && cd srl-k8s-anycast-lab
```

```bash
# deploy minikube cluster
minikube start --nodes 3 -p cluster1 --network-plugin=cni --cni=false
```

```bash
# deploy containerlab topology
clab deploy --topo srl-k8s-lab.clab.yml
```

```bash
# Prepare minikube for kube proxy replacement
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
docker exec cluster1 bash -c "iptables-save | grep -v KUBE | iptables-restore"
docker exec cluster1-m02 bash -c "iptables-save | grep -v KUBE | iptables-restore"
docker exec cluster1-m03 bash -c "iptables-save | grep -v KUBE | iptables-restore"
```

```bash
# Install Cilium
cilium install --version 1.14.3 --set tunnel=disabled,routingMode=native,kubeProxyReplacement=true,ipv4NativeRoutingCIDR=192.168.0.0/16,ipam.mode=cluster-pool,ipam.operator.clusterPoolIPv4PodCIDRList=192.168.16.0/20
```

Kube-proxy replaced by Cilium
POD IPAM done by Cilium
Native routing


```bash
# Fine tune Cilium installation (Enable BGP control plane)
cilium config set enable-bgp-control-plane true
# Fine tune Cilium installation (Allow native-routing using our eth1 interface)
cilium config set devices eth1 
# Restart the Cilium operator
kubectl get pods -n kube-system --no-headers -o custom-columns=":metadata.name" | grep cilium-operator | xargs kubectl delete -n kube-system pod
# These commands should be integrated in the Install Cilium step but I don't
# find it easy to find the correct set version (helm equivalent) of the Cilium CLI
# arguments  
```

```bash
# Configure Cilium
kubectl label nodes cluster1 bgp-policy=srl-k8s-anycast
kubectl label nodes cluster1-m02 bgp-policy=srl-k8s-anycast
kubectl label nodes cluster1-m03 bgp-policy=srl-k8s-anycast
kubectl apply -f cilium-bgp-policy.yaml
kubectl apply -f cilium-lb-ippool.yaml
```

```bash
# Add k8s HTTP echo deployment and LB service
kubectl apply -f nginx.yaml
```

## Tests

```bash
# check underlay sessions in Spine, leaf switches
A:spine1$ show network-instance default protocols bgp neighbor

# check MetalLB BGP sessions in Leaf switches
A:leaf2$ show network-instance ip-vrf-1 protocols bgp neighbor

# check kubernetes status
kubectl get nodes -o wide

kubectl get pods -o wide

kubectl get svc

# check Cilium BGP peers
cilium bgp peers

# verify BGP sessions in FRR daemon
cluster1$ show bgp summary

# verify running config of FRR daemon
cluster1$ show run

# Define the LB Pool IP assigned to the NGINX LB Service
kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1      <none>          443/TCP        5m16s
nginxhello   LoadBalancer   10.102.6.162   192.168.33.83   80:31321/TCP   40s

# check HTTP echo service
docker exec -it client4 curl http://192.168.33.83
Server address: 192.168.18.57:80
Server name: nginxhello-dc44dc-djjjl
Date: 28/Nov/2023:22:34:37 +0000
URI: /
Request ID: 4a2018487f43df842350c7a432e9ee30

docker exec -it client4 curl http://192.168.33.83
Server address: 192.168.16.70:80
Server name: nginxhello-dc44dc-5bg8h
Date: 28/Nov/2023:22:34:38 +0000
URI: /
Request ID: 63f7546dbfef0a2ca20dee2ea3e8335d

docker exec -it client4 curl http://192.168.33.83
Server address: 192.168.17.172:80
Server name: nginxhello-dc44dc-ql67p
Date: 28/Nov/2023:22:34:40 +0000
URI: /
Request ID: 6dbeeb681a8ce8021b3c7b5883ee9143
# requests are load balanced to different pods
```

## Delete the lab

```bash
# destroy clab topology and cleanup 
clab destroy --topo srl-k8s-lab.clab.yml --cleanup
```

```bash
# delete Minikube cluster
minikube delete --all
```
