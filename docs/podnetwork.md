# PodNetwork

## Proposal
The `PodNetwork` resource is designed to serve as a unique identifier for individual pod networks. By maintaining a reference to a specific pod network implementation along with its essential descriptive data, this resource enables Kubernetes APIs and controllers to uniquely and efficiently identify specific networks.

The API remains agnostic to the underlying network implementation; the referenced object may be a CRD representing a network or one from which a network is instantiated. Consequently, the reference requires the group, kind, name, and optionally, the namespace of the target object.

`PodNetwork` is defined as a cluster-scoped resource. `PodNetwork` objects are immutable once created. Each network object must correspond to exactly one `PodNetwork` instance. The API is designed to work with network implementations based on Dynamic Resource Allocation (DRA, see the [DRA Integration](#dra-integration) section). Network implementations can automatically create and manage a corresponding `PodNetwork` resource for every network they manage. The specific implementation is also responsible for managing the life cycle of the `PodNetwork` resource. A set of conformance tests enforces how Multi-Network interacts with DRA.

Once established, `PodNetwork` resources can be used as identifiers for network-dependent attributes and resources. In multi-network environments, these may include pod IPs, network policies, gateways, and services. While this API provides the framework for identification, it does not strictly dictate how these identifiers must be utilized. Future design examples and reference implementations will demonstrate the various ways `PodNetwork` objects can be integrated.

We plan to create a new API group called `multinetwork`. Initially, it will be placed in the `networking.x-k8s.io` API group. The goal is to eventually move this API under `networking.k8s.io` after proper hardening and approvals.

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

    // Status defines the current state of the network
    Status PodNetworkStatus `json:”status”`
}

// PodNetworkSpec defines the desired state of PodNetwork.
type PodNetworkSpec struct {
    // Provider specifies the network provider responsible for this instance of PodNetwork. 
    // The name must be a domain-prefixed path (e.g., "networking.gke.io/multinetwork") 
    // to avoid collisions.
    // +required
    // +kubebuilder:validation:MaxLength=250
    // +kubebuilder:validation:Pattern=`^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*\/[A-Za-z0-9\/\-. ~%]+$`
    Provider string `json:"provider"`
    
    // NetworkRef references the underlying specific network implementation.
    // +optional
    NetworkRef NetworkReference `json:"networkref,omitempty"`
}

type PodNetworkStatus struct {
    // Conditions describe the current conditions of the referenced network.
    //
    // Currently, the only known condition is
    //
    // * "Ready"
    //
    // A "Ready” condition means the referenced network functions normally. Network
    // implementations should decide 
    // services, gateways, etc., can be used normally. An empty Condition field means the
    // network is NOT ready. 
    //
    // Notes for implementors:
    //
    // Conditions are a listType `map`, which means that they function like a
    // map with a key of the `type` field _in the k8s apiserver_.
    //
    // This means that implementations must obey some rules when updating this
    // section.
    //
    // * Implementations MUST perform a read-modify-write cycle on this field
    //   when modifying it. That is, when modifying this field, implementations
    //   must be confident they have fetched the most recent version of this field,
    //   and ensure that changes they make are on that recent version.
    // * Implementations MUST NOT remove or reorder Conditions that they are not
    //   directly responsible for. For example, if an implementation sees a Condition
    //   with type `special.io/SomeField`, it MUST NOT remove, change or update that
    //   Condition.
    // * Implementations MUST always _merge_ changes into Conditions of the same Type,
    //   rather than creating more than one Condition of the same Type.
    // * Implementations MUST always update the `observedGeneration` field of the
    //   Condition to the `metadata.generation` of the `PodNetwork` at the time of update creation.
    // * If the `observedGeneration` of a Condition is _greater than_ the value the
    //   implementation knows about, then it MUST NOT perform the update on that Condition,
    //   but must wait for a future reconciliation and status update. (The assumption is that
    //   the implementation's copy of the object is stale and an update will be re-triggered
    //   if relevant.)
    //
    // +optional
    // +listType=map
    // +listMapKey=type
    // +kubebuilder:validation:MaxItems=8
    Conditions []metav1.Condition
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

This proposal standardizes driver implementations to integrate pod networks with DRA, facilitating a unified approach to multi-networking in Kubernetes and supporting the rollout of future features.

### Device PodNetwork Attributes

This proposal defines one standard device attribute that can be used in the `ResourceSlice` resource. This attribute allows `ResourceClaims` to select a specific PodNetwork and enable the system to identify devices that attach workloads to it.

```go
const (
  // StandardDeviceAttributePrefix is the prefix used for standard device attributes.
  StandardDeviceAttributePrefix = "multinetwork.networking.k8s.io/ " 

  // StandardDeviceAttributePodNetwork is a standard device attribute name
  // which identifies a pod network.
  // The value is a string value referring to the name of an existing PodNetwork cluster-scoped object.
  StandardDeviceAttributePodNetwork resourceapi.QualifiedName = StandardDeviceAttributePrefix + "podNetwork"
  
)
```

### Creating `PodNetwork` objects
In DRA based network implementations, a specialized controller manages the custom resource objects that represent networks. To ensure compatibility with the PodNetwork API, this controller may  automatically produce a matching PodNetwork object for each network it oversees. The following logic outlines how the controller should generally handle the addition and removal of `PodNetwork` resources:

```
FOR EACH Network ADDED:
  AddPodNetwork(Network)

FOR EACH Network TO-BE-DELETED:
  DeletePodNetwork(Network)

```
### ResourceClaim Status

Upon allocation of a device representing a specific network to a pod, the controller should include a reference to the `PodNetwork` object within the status of the corresponding `ResourceClaim`. To ensure consistency, this design requires all PodNetwork-compatible drivers to populate the `status.devices.[].data` field with the `podNetwork` property. The data structure is defined as follows:

```go
type podStatus struct {
	PodNetwork string `json:"podNetwork"`
}
```

Further, the network implementation should also fill out the ``status.devices.[].data.networkdata` field.

## Example

Considering an existing pod network implementation built from a `FooNetwork` custom resource:

```yaml
apiVersion: multinetwork.networking.x-k8s.io/v1alpha1
kind: FooNetwork
metadata:
  name: blue-net
  namespace: default
spec: 
  ...
```

The controller could then create a `PodNetwork` object, which is:

```yaml
apiVersion: multinetwork.networking.x-k8s.io/v1alpha1
kind: PodNetwork
metadata:
  name: foo-net-blue # If created by the controller, it can decide the naming convention.
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
        deviceClassName: class-foo-net-blue
status:
  devices:
  - device: eno1
    driver: foo.networking.com
    data:
      podNetwork: foo-net-blue
    networkdata:
      networkData:
      hardwareAddress: 5a:9f:d8:84:fb:51
      interfaceName: net1
      ips:
      - 10.10.1.2/24
...

```

## Conformance Tests

Rather than implementing a standalone controller for PodNetwork, we will rely on native Kubernetes resource handling. The `PodNetwork` resource is intended for use with DRA-based network implementations. Consequently, we are establishing conformance tests to validate the behavior of these network DRA drivers. These tests verify that pod networks can be discovered, selected, and observed through standard Kubernetes mechanisms, focusing specifically on API-related properties rather than general DRA driver behaviors.

Conformance tests validate:
* `PodNetwork` object lifecycle.
  1. When created, a `PodNetwork` holds `spec` field, with the `spec.Provider` field set properly, and, if exist, the `spec.NetworkRef` field should reference the proper network custom resource object. 
  2. When a `PodNetwork` is created, its `spec.Conditions` field ultimately contains `Ready` condition with value true.
  5. When a network is deleted, the corresponding `PodNetwork` is either deleted or with `Ready` condition with value false, or without the `Ready` condition.

* `ResourceSlice` attributes: the `ResourceSlice` resource advertised by a network implementation must include the following attributes with the value set properly:
  1. `multinetwork.networking.k8s.io/podNetwork`.

* `ResourceClaim` status reporting. 
  1. A pod network implementation must update the `ResourceClaim`'s device status to include proper data and populate `networkdata` field.

## Reference Implementation

A reference implementation will be added later.

## Summary
To summarize, the `PodNetwork` API will have the following features:

- `PodNetwork` allows us to define network-specific behaviors that can be used and referenced by other Kubernetes modules, such as Service, Gateway, Network Policy, etc.

- The actual implementation of the networks referenced by `PodNetwork` objects are not part of this API. It remains the network implementation's responsibility to be portable across platforms.

- `PodNetwork` itself does not define its behavior with Service, Gateway, Network Policy, etc. Rather, the implementations of Service, Gateway, Network Policy, etc. will define their behavior with `PodNetwork`. `PodNetwork` facilitates the portability of such features if only the network implementation it referenced is portable.

- In this proposal we avoid invasive designs, such as adding an extra `PodNetwork` reference to all of the existing Kubernetes resources like Pod, Service, Gateway, Network Policy, etc. Rather, related resources will find their respective ways to use `PodNetwork`, many without any API change.

## Alternatives
### `NetworkKind` API (formerly NetworkClass)
There have been two former iterations of multi-network APIs introduced and debated. The latest iteration is the `NetworkKind` API (formerly NetworkClass). It is intended to provide a classification and discovery mechanism that allows Kubernetes APIs and controllers to recognize and integrate multiple pod networks. However, that API has proven inconvenient because `NetworkKind` alone cannot uniquely identify a specific network, and in practice, users still need the network name in an identifier.

### original `PodNetwork`
The predecessor was the original `PodNetwork` proposal. There it sought to establish an abstract `Network` type as a standard base for all secondary network implementations. However, the extreme technical diversity of networks makes a unified abstract network type nearly impossible to define. Consequently, such an abstract network type saw significant pushback from the community.


