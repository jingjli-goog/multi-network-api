# Critical User Journey with the New PodNetwork API in GKE

Authors: [Jingjing Li](mailto:jingjingli2004@gmail.com)

Created: May 4, 2026

Last updated: May 8, 2026

**This document is in the public domain.**


## **Overview**

This document outlines the Critical User Journey (CUJ) for orchestrating various features in a multi-network environment, in the future Google Kubernetes Engine after it adopts the recently proposed PodNetwork API. While GKE has successfully provided multi-network support through custom implementations, this initiative aligns GKE with upstream community standards and evolving Kubernetes Enhancement Proposals (KEPs). By utilizing Dynamic Resource Allocation (DRA) primitives, this architecture introduces a standardized, extensible framework for network identification and pod-interface attachment, ensuring consistent behavior across diverse infrastructure environments.

The implementation centers on the PodNetwork API that serves as a centralized administrative handle for network configuration and logical network identification. A DRA-based network implementation is also implemented to work together with the PodNetwork API. The DRA network is responsible for performing hardware discovery, maintaining ResourceSlice resources, and automating complex pod network programming tasks.

The PodNetwork API provides a unified model for multi-network support, and is foundational for improving integration with core Kubernetes primitives such as Services and NetworkPolicies that have traditionally been constrained by single-network domain assumptions. This document serves as an operational example for this API, detailing the interaction between different components of Kubernetes. 

The rest of the document focuses on Alice, the administrator of a Kubernetes cluster managed in GKE, and Bob, a user of the cluster, and their journey of constructing a multi-network platform, and building features on it. We describe what are the required jobs of each role from their respective point of view in this future GKE version. When necessary, we also describe briefly how the features are supposed to be implemented behind the doors. 


## Establish the Multi-network Connection

Alice currently manages a cluster connected to a single network, the Default network.  Alice is asked to create nodes that are connected to a second network also. She is given the specifics of the second network, including the underlying Google Cloud VPC and the subnet, in addition to that for the default pod network. To establish the secondary pod network, she first creates a node pool that are connected to the specified subnet and VPC for the secondary pod network, using a `gcloud` command like the following:


```
gcloud container node-pools create nodes-multi-nic \
--cluster=main-cluster \
--additional-node-network network=vpc-a,subnetwork=subnet1-vpc-a \
--num-nodes=10 \
... ...
```


A set of nodes are then provisioned with two NICs. This step is of course outside the scope of Kubernetes and is platform-specific. 

Alice then moves on to create the secondary pod network inside the cluster. To do this she first creates a `GKENetwork` resource:


```
apiVersion: networking.gke.io/v1
kind: GkeNetwork
metadata:
  name: blue-net
spec:
  type: "L3"
  parametersRef:
    group: networking.gke.io
    kind: GKENetParamSet
    name: "vpc-a"
```


The `GKENetwork` is the CRD of GKE’s DRA-based network implementation. It references a parameter object of type `GKENetParamSet` that references the virtual network to which the NIC is connected. Note that `vpc-a` is basically the virtual network used to create the node-pool above.


```
apiVersion: networking.gke.io/v1
kind: GKENetParamSet
metadata:
  name: "vpc-a-params"
spec:
  vpc: "vpc-a"
  vpcSubnet: "subnet1-vpc-a"
  podIPv4Ranges:
    rangeNames:
    - "yellow-pods"
```


The purpose of the `GKENetParamSet` object is similar to the `NodeInterfaceName` field in the [reference implementation](https://docs.google.com/document/d/1NEBMIczXf6H_2zZLX-SiZqJwWvs0lB2Ds7WLxbwGiQo/edit?tab=t.0#bookmark=id.6q5owqurexb0)[^1]. That is, it helps indicate which NIC of the node the pod network is supposed to use. Under the hood, the Controller of `GKENetwork` collects the information of the node VM from the Google Cloud API, finds out the information of the VPC and the subnet each NIC connects to, and decides which NIC the specific `GKENetwork` resource should use. The spec.`podIPv4Ranges` field is used to decide the IP ranges for this pod network. A custom IPAM is then used to allocate pod CIDRs for each node.

She then inquires about the PodNetwork resource. A `PodNetwork` object pointing to the `GkeNetwork` is already there:


```
apiVersion: multinetwork.networking.x-k8s.io/v1
kind: PodNetwork
metadata:
  name: podnetwork-blue-net
spec:
  provider: networking.gke.io
  networkref:
    apigroup: networking.gke.io
    kind: GKENetwork
    name: "blue-net"
status:
  conditions: "Ready"

```


To make the network ready for supporting services, Alice then reserves an IP range to be used for services. To do so she places an `GKEServiceCIDR` resource:


```
apiVersion: networking.gke.io/v1
kind: GKEServiceCIDR
metadata:
  name: "blue-net-service-ips"
spec:
  podnetwork: "podnetwork-blue-net"
  ip4cidr: "192.168.0.0/26"
```


Notice the field `spec.podnetwork`. It specifies that this service CIDR is reserved in the network referenced by the `PodNetwork` object “podnetwork-blue-net”, the one Alice just checked above. The IPAM would consume this resource and reserve the corresponding IPs in the specific network to be used by services, accordingly.

With these all in place, Alice checks to see if the pod network is ready. In fact, she finds the following `ResourceSlice` for each node of the multi-nic-nodes node pool:


```
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  annotations:
    networking.gke.io/podCIDRs: ["192.168.0.64/29"]
  name: multi-nic-nodes-001-pod-network
spec:
  devices:
    - name: blue-network-resource
      allowMultipleAllocations: true
      basic:
        attributes:
          networking.gke.io/podNetwork:
            string: blue-net
        capacity:
          ips: 
            quantityValue: "8"
  driver: networking.gke.io
  nodeName: nodes-multi-nic-001
  pool:
    name: nodes-multi-nic

```


A device class corresponding to the `blue-net` has also been created:


```
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: "blue-net-device"
spec:
  selectors:
  - cel:
      expression: device.attributes["multinetwork.networking.x-k8s.io"].podnetwork == "podnetwork-blue-net"
```


Alice then handles the network configurations of `blue-net` to a cluster user, Bob. 

Bob intends to build a HTTP service on the secondary pod network `blue-net`. He first needs to launch a set of pods running the service app. To do so, he creates a `ResourceClaimTemplate` to be used by all the app pods. The template will be used to generate `ResourceClaim`s for the specific pod network:


```
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: blue-net-claim-template
spec:
  spec:
    devices:
      requests: 
      - name: network-request
        exactly:
          deviceClassName: blue-net-device
          capacity:
            requests:
              ips: 1
```


Notice that the claim requests a device in the device class named “blue-net-device”. This device class, in its definition above, matches to one with an attribute of “multinetwork.networking.x-k8s.io/podnetwork” that equals “blue-net”. This is in the `ResourceSlice` generated by the network DRA driver.

Bob then deployed the server pods using a `Deployment` resource, each claiming a `ResourceClaim` generated from the aforementioned template. When checked, one of such pod instance may have the following configuration:


```
apiVersion: 
```


The fulfilled `ResourceClaim` looks like the following:


```
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: my-service-app-5b6d7c8d9f-pod-net-claim
spec:
  devices:
    requests: 
    - name: network-request
      exactly:
        deviceClassName: pod-network
        capacity:
          requests:
            ips: 1
status:
...
  devices:
  - device: blue-network-resource
    driver: networking.gke.io
    data:
      podNetwork: podnetwork-blue-net
    networkData:
      hardwareAddress: 5a:9f:d8:84:fb:51
      interfaceName: eth1
      ips:
      - 192.168.0.64/29
    pool: nodes-multi-nic
 
```


Notice the `status.devices[].data` field, which shows the `PodNetwork` the device belongs to. The Pod itself looks like this


```
apiVersion: v1
kind: Pod
metadata:
  annotations:
   ... ...
    networking.gke.io/pod-ips: '[{"podnetwork":"podnetwork-blue-net","ip":"192.168.0.66"},{"podnetwork":"default","ip":"10.60.1.5"}]'
  labels:
    app: my-service
    multinetwork.networking.x-k8s.io/podnetwork: default
    multinetwork.networking.x-k8s.io/podnetwork: podnetwork-blue-net
...
spec:
...
status:
...
```


Note the multi-network related annotations and labels.


<!-- Footnotes themselves at the bottom. -->
## Notes

[^1]:
     To be updated before this doc is sent out.

