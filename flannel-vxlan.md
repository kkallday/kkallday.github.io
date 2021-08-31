# Deploying flannel with VXLAN backend

[Flannel](https://github.com/flannel-io/flannel) is a simple overlay networking
implementation that provides a way for hosts to communicate across multiple
networks (or network segments). It is most commonly used to provide L3 IP
connectivity between containers running on multiple hosts. Flannel is widely
used as a network fabric for Kubernetes due to its simplicity.

This post will demonstrate how to deploy flannel using the VXLAN backend to
create a virtual overlay network that spans two subnetworks.

## Architecture

![Diagram of multiple cells and each one hosting multiple containers](/assets/diagrams/flannel-vxlan/fig-1.png)

In the diagram above, there are 2 virtual machines (cells) hosting multiple
containers. If container A wants to make an HTTP request to a server running on
container E, how would packets traverse the network? Flannel addresses part of
this problem and Linux virtual networking devices address the rest.

![Diagram of final architecture](/assets/diagrams/flannel-vxlan/fig-2.png)

### Flannel's responsibility

A flannel daemon will run on each cell and obtain a subnet lease for a specific
range. For instance, cell 1 could get `172.16.10.0/24` and cell 2
`172.16.20.0/24`.

Furthermore, flannel will create a tap/tun device that will perform packet
encapsulation so that packets can traverse between both subnetworks. When a
container running on cell 1 wants to make an HTTP request to a container
running on cell 2 (in a different subnet), cell 1's routing table will instruct
the kernel to send those packets to the flannel tap/tun device. The device will
wrap the packet in a UDP packet with the destination set to cell 2's IP and
port `4789` (default VXLAN) and send packet out of the cell through the gateway.
Upon receiving that packet, cell 2's flannel tap/tun device will de-capsulate
the packet.

### Bridge/Virtual Ethernet Device responsibility

After cell 2 receives the packet, how does it get routed to the correct
container? This is where the bridge and virtual ethernet devices come into
play.

Cell 2's flannel tap/tun device will receive the UDP packet on port `4789` and
decapsulate it. After decapsulation, the routing rules define what device on
cell 2 will receive the packet. On cell 2, there is a routing rule that states
that packets that a destined for the range `172.16.20.0/24` should go to the
bridge device.

The bridge device acts as a L2 router (similar to your home router). Each
container has its own network stack and therefore, has its own virtual ethernet
device and these virtual ethernet devices are hooked up to the bridge.
Container E has a virtual ethernet IP of `172.16.20.2` therefore, it will
receive the HTTP request packets.

## Deployment

To accomplish this, I did the following:

- Created a virtual private network
- Deployed a single-node [etcd](http://etcd.io) cluster (Flannel uses etcd as a database)
- Created a Linux virtual bridge on each node
- Created Linux virtual ethernet device pairs for each container
- Ran Flannel's daemon (`flanneld`) on each node
- Deployed this architecture on Google Cloud via [Terraform](https://terraform.io)

For demonstration purposes, I only created namespaces instead of full blown
containers. A "container" is an abstraction that uses Linux features called
"cgroups" and "namespaces" to provide virtualized isolation.

In this post, the term "cell" refers to an instance that hosts one or more containers.

**This demo is safe for production use. It does not take into account important
security precautions.**

## Setup Google Cloud Credentials to be used by Terraform

For simplicity, I used Terraform to create the necessary cloud infrastructure
instead of manually clicking through the Google Cloud Console.

1. Install [Terraform](https://terraform.io) onto your local computer.

1. Create a [Google Cloud](https://console.cloud.google.com) Project. This
   allows you to easily delete all the infrastucture in a few clicks.

1. Enable [Google Compute Engine](https://console.developers.google.com/apis/library/compute.googleapis.com)
for your project.

1. Create a [service account](https://console.cloud.google.com/apis/credentials/serviceaccountkey)
   and give it the "Project Editor" role. I named mine "terraformer". Download
   the key as JSON. **This key provides access to your project so keep it
   confidential!**

Now you have a key that can be used by Terraform to create the cloud
infrastructure.

## Create Infrastructure

1. Clone the [terraforming-flannel-vxlan repo](https://github.com/kkallday/terraforming-flannel-vxlan)
   containing the terraform script to create the infrastructure

1. Fill out all variables in `terraform.tfvars`. All variables are required.

1. To create infrastructure, run `terraform apply`

The following section describes the infrastructure was created.

### Creating VPC

A VPC network with two subnetworks was created. This simulates two different
network segments to demonstrate packets originating from a container are
exiting the host node and entering the other other container by going into the
other container's host node.

### Create static IPs

Three static IPs were created so that configuring flannel is easier and none of
the IPs change after re-deploy.

- etcd gets `10.0.1.2`
- node 1 gets `10.0.1.3`
- node 2 gets `10.0.2.3`

### Configure VPC Firewall

In order to SSH, the VPC firewall needs to allow TCP traffic via port 22.

Furthermore, traffic is allowed to flow between any instance on any port within
the VPC. This is not good for security/production however, for demonostration
purposes this is fine.

When flannel encapsulates packets, they are sent to the destination node's port
`4789` via UDP. etcd received client traffic on port `2379` via TCP.

### Deploy etcd and flannel nodes

Three instances are deployed: one for etcd and two for flannel/node to host
containers. Each of these instances obtain static IPs (described above).

### Start etcd

In order to start etcd, do the following:

1. `gcloud compute ssh --project <NAME OF YOUR PROJECT> --zone us-central1-a etcd`

1. In the instance, run:

  ```
  $ sudo chmod u+x /terraforming-flannel-vxlan/scripts/etcd-init.sh
  $ sudo /terraforming-flannel-vxlan/scripts/etcd-init.sh
  ```

This script will do the following:

1. Download etcd binaries to `/usr/local/bin/`

1. Write a systemd unit file `/etc/systemd/system/etcd.service`

1. Start etcd using systemd

The etcd configuration can be found in the systemd unit file. It configures
etcd to listen for client traffic on `10.0.1.2:2379`. This will be used by
`flanneld` to read and store configuration. You'll also notice that V2 API is
enabled. The flannel daemon uses the v2 (not v3) API when creating and getting
data from etcd.

### Start flanneld

In order to start flanneld, do the following:

1. `gcloud compute ssh --project <NAME OF YOUR PROJECT> --zone us-central1-a cell-1`

1. In the instance, run:

  ```
  $ sudo chmod u+x /terraforming-flannel-vxlan/scripts/cell-flannel-init.sh
  $ sudo /terraforming-flannel-vxlan/scripts/cell-flannel-init.sh
  ```

This script will do the following:

1. Download flanneld binary to `/usr/local/bin/`

1. Writes flanneld configuration to etcd (described below)

1. Write a systemd unit file `/etc/systemd/system/flanneld.service`

1. Start flanneld using systemd

1. Repeat step 2 on cell 2

When `flanneld` starts, the daemon that will read the subnet configuration from
etcd and take a subnet from the range specified in the configuration. etcd was
loaded with the following configuration (etcd key:
`/coreos.com/network/config`)

```
{
       "Network": "172.16.0.0/16",
       "SubnetLen": 24,
       "SubnetMin": "172.16.10.0",
       "SubnetMax": "172.16.99.0",
       "Backend": {
               "Type": "vxlan",
               "Port": 8472
       }
}
```

The first cell will take, at random, a `/24` subnet within `172.16.0.0/16`.
For example, it may take `172.16.24.0/24`. This cell would then receive traffic
for IPs between `172.16.24.0` to `172.16.24.255`. As mentioned in the
architecture section, each node will host one or more containers and in this
case, containers that are created in this node should be assigned an IP in the
aforementioned range.

### Bridge and Virtual Ethernet Configuration on flannel nodes

Linux bridges allow you to create "virtual switch" within a single machine.
This is useful because flannel only helps route packets to the correct node.
Flannel does not have responsibility for what happens to the packets after they
have arrived on the machine.

There are multiple containers running on a node and each of those containers
need to be allocated an IP. To allocate an IP, a network interface is needed,
therefore, each container will be given a virtual ethernet interface and it
will be connected to the bridge device.

For instance, lets imagine node 1 has a subnet lease of `172.16.10.0/24`, a
bridge device is created and assigned `172.16.10.1/24`, container 1's virtual
ethernet interface is assigned `172.16.10.2`, and container 1's virtual
ethernet is assigned `172.16.10.3`.

In order to setup the bridge and virtual ethernet devices, do the following:

1. `gcloud compute ssh --project <NAME OF PROJECT> --zone us-central1-a cell-1`

1. In the instance, run:

  ```
  $ sudo chmod u+x /terraforming-flannel-vxlan/scripts/cell-bridge-veth-init.sh
  $ sudo /terraforming-flannel-vxlan/scripts/cell-bridge-veth-init.sh
  ```

1. Repeat on node 2

This creates the bridge and veth devices and two network namespaces on each
node. It also launches a python HTTP server listening on port `8000` in each namespace.

![Diagram of bridge/veth architecture](/assets/diagrams/flannel-vxlan/fig-3.png)

## Try it out

Try making an HTTP request from container A on node 1 to container C on node 2.

1. `gcloud compute ssh --project <NAME OF YOUR PROJECT> --zone us-central1-b cell-2`

1. On node 2, run `ip addr` to get the bridge `br0` IP address

1. `gcloud compute ssh --project <NAME OF YOUR PROJECT> --zone us-central1-a cell-1`

1. On node 1, run `sudo ip netns exec con_a curl -v http://<bridge IP except last octet>.2:8000`

You should get a sucessful response.

*Debugging tip:* run `sudo tshark -i flannel.1` to check if packets are reaching
the tap/tun device or `sudo tshark -i br0` to check if packets are reaching the
bridge device.


![Diagram of final architecture](/assets/diagrams/flannel-vxlan/fig-2.png)

## Clean Up

Run `terraform destroy` or delete the Google Cloud Project.

## References

Kristen Jacbos gave an [excellent talk](https://youtu.be/6v_BDHIgOY8) along
with [repo of scripts for setting up virtual ethernet and bridge devices](https://github.com/kristenjacobs/container-networking).
