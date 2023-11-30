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

## Low level details on packet flow

These are the evidence that some packets don't go via the SRL fabric

```
# Node, PODs, LB Service IP info (for reference)

[ec2-user@ip-10-10-3-9 ~]$ kubectl get nodes -o wide
NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION                  CONTAINER-RUNTIME
cluster1       Ready    control-plane   27m   v1.28.3   192.168.49.2   <none>        Ubuntu 22.04.3 LTS   6.1.61-85.141.amzn2023.x86_64   docker://24.0.7
cluster1-m02   Ready    <none>          27m   v1.28.3   192.168.49.3   <none>        Ubuntu 22.04.3 LTS   6.1.61-85.141.amzn2023.x86_64   docker://24.0.7
cluster1-m03   Ready    <none>          26m   v1.28.3   192.168.49.4   <none>        Ubuntu 22.04.3 LTS   6.1.61-85.141.amzn2023.x86_64   docker://24.0.7

[ec2-user@ip-10-10-3-9 ~]$ kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
nginxhello-dc44dc-57p9r   1/1     Running   0          16m   192.168.16.238   cluster1-m03   <none>           <none>
nginxhello-dc44dc-r6chf   1/1     Running   0          16m   192.168.18.126   cluster1-m02   <none>           <none>
nginxhello-dc44dc-rbmzt   1/1     Running   0          16m   192.168.17.123   cluster1       <none>           <none>
[ec2-user@ip-10-10-3-9 ~]$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1      <none>           443/TCP        22m
nginxhello   LoadBalancer   10.97.152.33   192.168.33.226   80:31241/TCP   16m

[ec2-user@ip-10-10-3-9 ~]$ docker network inspect cluster1 | egrep "(Mac|IPv4)"
                "MacAddress": "02:42:c0:a8:31:04",
                "IPv4Address": "192.168.49.4/24",
                "MacAddress": "02:42:c0:a8:31:02",
                "IPv4Address": "192.168.49.2/24",
                "MacAddress": "02:42:c0:a8:31:03",
                "IPv4Address": "192.168.49.3/24",

bridge fdb | grep 02:42:c0:a8:31:02
02:42:c0:a8:31:02 dev vethb253dcd master br-ad531a98605c

# Client4 => LB Svc - Captured on Leaf1 e1-1 (sudo ip netns exec leaf1 tcpdump -nni e1-1 tcp port 80)

root@client4:/ $ curl http://192.168.33.226
Server address: 192.168.18.126:80
Server name: nginxhello-dc44dc-r6chf
Date: 30/Nov/2023:20:11:23 +0000
URI: /
Request ID: 456638c93c60e487927452734c21f080

20:11:23.046431 IP 192.168.2.14.34566 > 192.168.33.226.80: Flags [S], seq 399663693, win 56760, options [mss 9460,sackOK,TS val 1817658588 ecr 0,nop,wscale 7], length 0
20:11:23.046456 IP 192.168.49.2.34566 > 192.168.18.126.80: Flags [S], seq 399663693, win 56760, options [mss 9460,sackOK,TS val 1817658588 ecr 0,nop,wscale 7], length 0
20:11:23.046971 IP 192.168.33.226.80 > 192.168.2.14.34566: Flags [S.], seq 2535097468, ack 399663694, win 65160, options [mss 1460,sackOK,TS val 2236510185 ecr 1817658588,nop,wscale 7], length 0
20:11:23.047667 IP 192.168.2.14.34566 > 192.168.33.226.80: Flags [.], ack 1, win 444, options [nop,nop,TS val 1817658589 ecr 2236510185], length 0
20:11:23.047676 IP 192.168.49.2.34566 > 192.168.18.126.80: Flags [.], ack 2535097469, win 444, options [nop,nop,TS val 1817658589 ecr 2236510185], length 0
20:11:23.047700 IP 192.168.2.14.34566 > 192.168.33.226.80: Flags [P.], seq 1:79, ack 1, win 444, options [nop,nop,TS val 1817658589 ecr 2236510185], length 78: HTTP: GET / HTTP/1.1
20:11:23.047704 IP 192.168.49.2.34566 > 192.168.18.126.80: Flags [P.], seq 0:78, ack 1, win 444, options [nop,nop,TS val 1817658589 ecr 2236510185], length 78: HTTP: GET / HTTP/1.1
20:11:23.048296 IP 192.168.33.226.80 > 192.168.2.14.34566: Flags [.], ack 79, win 509, options [nop,nop,TS val 2236510187 ecr 1817658589], length 0
20:11:23.048410 IP 192.168.33.226.80 > 192.168.2.14.34566: Flags [P.], seq 1:371, ack 79, win 509, options [nop,nop,TS val 2236510187 ecr 1817658589], length 370: HTTP: HTTP/1.1 200 OK
20:11:23.049003 IP 192.168.2.14.34566 > 192.168.33.226.80: Flags [.], ack 371, win 442, options [nop,nop,TS val 1817658590 ecr 2236510187], length 0
20:11:23.049013 IP 192.168.49.2.34566 > 192.168.18.126.80: Flags [.], ack 371, win 442, options [nop,nop,TS val 1817658590 ecr 2236510187], length 0
20:11:23.049425 IP 192.168.2.14.34566 > 192.168.33.226.80: Flags [F.], seq 79, ack 371, win 442, options [nop,nop,TS val 1817658590 ecr 2236510187], length 0
20:11:23.049434 IP 192.168.49.2.34566 > 192.168.18.126.80: Flags [F.], seq 78, ack 371, win 442, options [nop,nop,TS val 1817658590 ecr 2236510187], length 0
20:11:23.049998 IP 192.168.33.226.80 > 192.168.2.14.34566: Flags [F.], seq 371, ack 80, win 509, options [nop,nop,TS val 2236510189 ecr 1817658590], length 0
20:11:23.050497 IP 192.168.2.14.34566 > 192.168.33.226.80: Flags [.], ack 372, win 442, options [nop,nop,TS val 1817658592 ecr 2236510189], length 0
20:11:23.050505 IP 192.168.49.2.34566 > 192.168.18.126.80: Flags [.], ack 372, win 442, options [nop,nop,TS val 1817658592 ecr 2236510189], length 0


What we see:

- request coming from client4 (192.168.2.14) => LB VIP on cluster 1 (192.168.33.226)
- LB sending traffic to POD on cluster1-m03 (192.168.16.238)  ... traffic is NATed with node IP (192.168.49.2) :-( 

=> we don't see the return traffic via our fabric; it goes over the minikube bridge interface

sudo tcpdump -nni vethb253dcd tcp port 80

20:11:23.046960 IP 192.168.18.126.80 > 192.168.49.2.34566: Flags [S.], seq 2535097468, ack 399663694, win 65160, options [mss 1460,sackOK,TS val 2236510185 ecr 1817658588,nop,wscale 7], length 0
20:11:23.048288 IP 192.168.18.126.80 > 192.168.49.2.34566: Flags [.], ack 79, win 509, options [nop,nop,TS val 2236510187 ecr 1817658589], length 0
20:11:23.048400 IP 192.168.18.126.80 > 192.168.49.2.34566: Flags [P.], seq 1:371, ack 79, win 509, options [nop,nop,TS val 2236510187 ecr 1817658589], length 370: HTTP: HTTP/1.1 200 OK
20:11:23.049990 IP 192.168.18.126.80 > 192.168.49.2.34566: Flags [F.], seq 371, ack 80, win 509, options [nop,nop,TS val 2236510189 ecr 1817658590], length 0


Most of the traffic is via the simulated fabric ... but it is a pitty to have those few packets still going over the docker bridge :-(

```