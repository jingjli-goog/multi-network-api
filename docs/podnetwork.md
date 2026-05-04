# PodNetwork

## Proposal
The `PodNetwork` resource is designed to serve as a unique identifier for individual pod networks. By maintaining a reference to a specific pod network implementation along with its essential descriptive data, this resource enables Kubernetes APIs and controllers to reference and identify specific networks both uniquely and efficiently.

The API remains agnostic to the underlying network implementation; the referenced object may be a CRD representing a network or one from which a network is instantiated. Consequently, the reference requires the group, kind, name and, optionally namespace of the target object.

`PodNetwork` is defined as a cluster-scoped resource. `PodNetwork` objects are immutable once created. Each network object must correspond to exactly one `PodNetwork` instance. The lifecycle of this object is strictly tied to the referenced network object: it should be instantiated after the network object is created and removed prior to that object's deletion. The API is designed to work with DRA-based network implementations (see the [DRA Integration](#dra-integration) section). In this case, network implementations should automatically create and manage a corresponding `PodNetwork` resource for every network the same implementation manages. A set of conformance tests enforces DRA-related behavior. Although using DRA based implementations is the primary focus, this API is not restricted to DRA only. It can also be used for other network implementation approaches. 

Once established, `PodNetwork` resources can be used as identifiers or labels for network-dependent attributes and resources. In multi-network environments, such attributes and resources may include pod IPs, gateways and services. While this API provides the framework for identification, it does not strictly dictate how these identifiers must be utilized. Future design examples and reference implementations will demonstrate the various ways `PodNetwork` objects can be integrated.

We plan to create a new API group called `multinetwork`. Currently, it will be placed in the `networking.x-k8s.io` API group. The goal is to place this API under `networking.k8s.io` eventually, after proper approvals.

## API Design

`PodNetwork` object is described as follows:

```go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Cluster

// PodNetwork identifies a pod network in a Kubernetes Cluster.
type PodNetwork struct {
    // Standard type and object metadata.
    metav1.TypeMeta   `json:",inline"`
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    // Spec defines the desired state of the PodNetwork.
    Spec PodNetworkSpec `json:"spec"`
}

// PodNetworkSpec defines the desired state of PodNetwork.
type PodNetworkSpec struct {
    // Provider specifies the network provider responsible for this instance of PodNetwork.
    // +required
    Provider string `json:"provider"`
    
    // NetworkRef references the underlying specific network implementation.
    // +optional
    NetworkRef NetworkReference `json:"networkref,omitempty"`
}

// NetworkReference provides the details to locate the specific network object.
type NetworkReference struct {
    // Kind of the referenced network.
    // +required
    Kind string `json:"kind"`
    
    // Name of the referenced network.
    // +required
    Name string `json:"name"`
    
    // APIGroup of the referenced network.
    // +required
    ApiGroup string `json:"apigroup"`
    
    // Namespace of the referenced network.
    // +optional
    Namespace string `json:"namespace,omitempty"`
}
```

## DRA Integration

This proposal standardizes driver implementations to integrate pod networks with Dynamic Resource Allocation (DRA), facilitating a unified approach to multi-networking in Kubernetes and supporting the rollout of future features.

### Device PodNetwork Attributes

This proposal defines two standard device attributes that can be used in the `ResourceSlice` resource. These attributes allow `ResourceClaims` to select specific pod networks and enable the system to identify devices that attach workloads to a given pod network.

```go
const (
  // StandardDeviceAttributePrefix is the prefix used for standard device attributes.
  StandardDeviceAttributePrefix = "multinetwork.networking.x-k8s.io/" 

  // StandardDeviceAttributePodNetwork is a standard device attribute name
  // which identifies a pod network.
  // The value is a string value referring to the name of an existing PodNetwork cluster-scoped object.
  StandardDeviceAttributePodNetwork resourceapi.QualifiedName = StandardDeviceAttributePrefix + "podNetwork"

  // StandardDeviceAttributePodNetworkNamespace is a standard device attribute name
  // which describes the namespace of a pod network.
  // The value is a string value referring to the namespace of a pod network object.
  // The attribute is optional for a PodNetwork holding a NetworkRef with empty Namespace field. 
  // The attribute is mandatory for a PodNetwork holding a NetworkRef with non-empty Namespace field. 
  StandardDeviceAttributePodNetworkNamespace resourceapi.QualifiedName = StandardDeviceAttributePrefix + "podNetworkNamespace"  
)
```

### ResourceClaim Status

An allocated device is reported in the associated `ResourceClaim` status with a reference to the PodNetwork object. This proposal standardizes for all drivers supporting PodNetwork to properly set the `status.devices[].data` with the `podNetwork` field. The structure for the data should include the following definition:

```go
type podStatus struct {
	PodNetwork string `json:"podNetwork"`
}
```

### Creating `PodNetwork` objects
In DRA based network implementation, the CRD objects representing networks are managed by a dedicated controller. To integrate with the new PodNetwork API, this controller should generate a corresponding PodNetwork object for every network under its management. In general, the controller should manage add/delete `PodNetwork` objects in the following way:

```
FOR EACH Network ADDED:
  AddPodNetwork(Network)

FOR EACH Network TO-BE-DELETED:
  DeletePodNetwork(Network)

```

## Example

Considering an existing pod network implementation built from a `FooNetwork` CRD:

```yaml
apiVersion: multinetwork.networking.x-k8s.io/v1
kind: FooNetwork
metadata:
  name: blue-net
  namespace: default
spec: 
  ...
```

The controller should create a `PodNetwork` object, which is:

```yaml
apiVersion: multinetwork.networking.x-k8s.io/v1alpha1
kind: PodNetwork
metadata:
  name: foo-net-blue # Controller can decide the naming convention. Immutable nature of PodNetwork ensures that the name is unique for each network.
spec: 
  provider: foo.networking.com
  networkref:
    kind: FooNetwork
    name: blue-net
    apigroup: multinetwork.networking.x-k8s.io
    namespace: default
```

A `DeviceClass` can be defined to represent all the resources of `FooNetwork`:

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: class-foo-net-blue
spec:
  selectors:
  - cel:
      expression: device.attributes["multinetwork.networking.x-k8s.io"].podNetwork == "foo-net-blue"
```

`FooNetwork`'s DRA driver should create `ResourceSlice` objects like the following (common fields ignored):

```yaml
apiVersion: resource.k8s.io/v1beta1
kind: ResourceSlice
metadata:
  name: node1-foo-network-blue
spec:
  devices:
  - name: blue-net
    basic:
      attributes:
        multinetwork.networking.x-k8s.io/podNetwork:
          string: foo-net-blue
....

```

Pods should use a `ResourceClaim` like the following to claim such a pod network. Its status, after the claim is satisfied, is also shown below:

```yaml
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaim
metadata:
  name: claim-for-blue-net
spec:
  devices:
    requests: 
    - name: network-request
      exactly:
        deviceClassName: foo-net-blue
status:
  devices:
  - device: eno1
    driver: foo.networking.com
    data:
      podNetwork: foo-net-blue
...

```

## Conformance Tests

Conformance tests validate that a network implementation correctly integrates with `PodNetwork` and the Kubernetes Resource API. These tests ensure that pod networks are discoverable, selectable, and observable using standard Kubernetes mechanisms. Note that the tests here only verify properties related to this API.

Conformance tests validate:
* `PodNetwork` object lifecycle: a network implementation creates `PodNetwork` objects and manages them properly.
  1. A `PodNetwork` object is created once the specific network instance is created.
  2. The `PodNetwork` holds a proper reference to the network CRD object, with the `spec.Provider` field set properly. 
  3. If a `PodNetwork` referencing a specific network is deleted while the referenced network is still alive, it should be recreated right away.
  4. When a network instance itself is deleted, the corresponding `PodNetwork` object should be deleted.

* `ResourceSlice` attributes: the `ResourceSlice` resource advertised by a network implementation must include the following attributes with the value set properly:
  1. `multinetwork.networking.x-k8s.io/podNetwork`.
  2. `multinetwork.networking.x-k8s.io/podNetworkNamespace`, only included if the PodNetwork object is namespace-scoped.

* `ResourceClaim` status reporting. 
  1. A pod network implementation must update the `ResourceClaim`'s device status to include the corresponding `PodNetwork` in the `data` field, after the claim is fulfilled.

## Reference Implementation

A reference implementation will be added later.


## Alternatives
Over the last few months, two primary iterations of multi-network APIs have been introduced and debated. The latest iteration is the `NetworkKind` API (formerly NetworkClass). It intended to offer a discovery and classification mechanism for Kubernetes controllers to manage multiple pod networks. The goal of that version is specifically stated as to “provide(s) a classification and discovery mechanism that allows Kubernetes APIs and controllers to recognize and integrate multiple pod networks”. Despite this goal, that API has proven inconvenient. `NetworkKind` alone cannot uniquely identify a specific network. In practice, users must employ an additional network name identifier. This reliance on a separate name identifier makes the `NetworkKind` largely redundant, providing utility only when identical network names are shared across different kinds.

The predecessor to `NetworkKind` was the original `PodNetwork` proposal. There it sought to establish an abstract `Network` type as a standard base for all secondary network implementations. While this approach would theoretically solve the identification issue mentioned above by using the `PodNetwork` object itself as the identifier, it faces a significant dilemma: the extreme technical diversity of networks (e.g., L2 vs. L3, RDMA vs. TCP/IP) makes a unified abstract network type nearly impossible to define. Any such type becomes so minimal that nearly all functional configuration must be relegated to implementation-specific key-value pairs, severely limiting its usefulness. Consequently, we observed significant pushback against adopting this API for both new and existing secondary-network implementations.

Drawing from community feedback on prior iterations and with insights gained from planning a reference implementation, we have identified two core guiding principles:

1. **Explicit network identification is highly valuable**: Such a mechanism enables resources like Services, Gateways, and Network Policies to explicitly target a particular network, satisfying a significant community demand.
2. **High-level network abstraction is very difficult**: Abstract network types, due to the technical diversity dilemma mentioned above, offer minimal utility to developers—particularly those managing established network implementations. Consequently, there is little incentive for these developers to migrate toward a universal, high-level abstraction of a `Network`.

In light of these observations, we proposed the current version of `PodNetwork` API that purely serves as a discovery and identification framework. Specifically, we encourage generating `PodNetwork` resources **from** existing implementations, rather than forcing network implementations to be built **upon** the `PodNetwork` type. This flexibility is intended to accelerate community adoption.
