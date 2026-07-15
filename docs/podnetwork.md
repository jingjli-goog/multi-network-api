# PodNetwork

## Proposal
The `PodNetwork` resource is designed to serve as a unique identifier for 
networks used by pods. While the underlying networks may use different 
implementations from various providers, this API provides 
a unified interface for Kubernetes APIs and controllers to reference these 
networks while abstracting away their implementation details. Such controllers 
and APIs (e.g., Gateways, Services, and network policies) may then implement 
their functionality in a multi-network environment without the 
implementation-specific knowledge of each network when it is plausible. 
Further more, a `PodNetwork` object holds a reference to the underlying network 
implementation. When implementation details are required, 
controllers can still look up the information from the referenced object. 
In this way, `PodNetwork` helps divide the application implementation into 
a high-level layer—where only abstract network concepts and basic configurations 
(e.g., IP addresses) are needed—and a low-level layer, where network 
implementation details are used. The ultimate goal is to improve the 
portability of applications across different network implementations.

`PodNetwork` is designed to work with network implementations built on Custom 
Resource Definitions (CRDs). A `PodNetwork` object holds a reference to the 
Custom Resource (CR) of the network implementation, although the API allows 
this reference to be empty to support cases where the network implementation 
is implicit and does not depend on a custom resource (CR). For networks 
implemented through Dynamic Resource Allocation (DRA), this API defines a 
series of requirements for the DRA driver to properly integrate with this API,
see the [DRA Integration](#dra-integration) section. Notice that, this API does
not impose requirements on how the network should be implemented.

`PodNetwork` is defined as a cluster-scoped resource. This accommodates both 
namespace-scoped and cluster-scoped network implementations.

We plan to create a new API group called `multinetwork`. Initially, it will 
be placed in the `networking.x-k8s.io` API group, with the goal of eventually 
moving this API under `networking.k8s.io` after proper hardening and approvals.

## API Design

`PodNetwork` resource is described as follows. Note that this proposal introduces 
no dedicated controller the resource, thus no explicit validation is done for `PodNetwork`
objects. We will rely on native Kubernetes resource handling.

```go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Cluster

// PodNetwork identifies a network in a Kubernetes Cluster to which pods connect.
type PodNetwork struct {
    // Standard type and object metadata.
    metav1.TypeMeta   `json:",inline"`
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    // Spec defines the desired state of the PodNetwork.
    // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="This field is immutable"
    Spec PodNetworkSpec `json:"spec"`

    // Status defines the current state of the network
    Status PodNetworkStatus `json:”status”`
}

// PodNetworkSpec defines the desired state of PodNetwork.
type PodNetworkSpec struct {
    // Provider specifies the provider of the network implementation the `PodNetwork` object references.  
    // The network implementation controllers are responsible for maintaining the `status` field of a `PodNetwork` object with corresponding `Provider` field.
    // +required
    // +kubebuilder:validation:MaxLength=250
    // +kubebuilder:validation:Pattern=`^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*\/[A-Za-z0-9\/\-. ~%]+$`
    Provider string `json:"provider"`
    
    // NetworkRef references the underlying specific network implementation.
    // +optional
    NetworkRef *NetworkReference `json:"networkref,omitempty"`
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
    // +optional
    // +listType=map
    // +listMapKey=type
    // +kubebuilder:validation:MaxItems=8
    // +kubebuilder:default={{type: "Ready", status: "Unknown", message: "Waiting for controller", reason: "Pending", lastTransitionTime: "1970-01-01T00:00:00Z"}}
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

## `PodNetwork` Resource

### Naming and Conflicts

The `PodNetwork` resource is designed with an immutable `Spec` field, 
thus creating an object with the same name as an existing `PodNetwork` 
object will not overwrite the existing one. This API does not enforce 
any naming convention. Users may choose their own convention in naming
`PodNetwork` objects. Users should be aware of the possible name 
conflicts when creating `PodNetwork` objects, and handle the conflicts
gracefully. One approach to handle name conflict is to select a name for a
new `PodNetwork` object, verify the name is not used first, then create
the object, and verify that the creation is indeed successful.

### PodNetworkSpec

The `Provider` field in `PodNetworkSpec` specifies which controller should
watch and update the `Status` field of this `PodNetwork` object. The `NetworkRef`
field stores the namespace, API group, kind and name of the CR of the network
implementation. This uniquely identifies the underlying network. When it 
is not set, the `PodNetwork` object refers to an implicit defined network.

### PodNetworkStatus
When created, the `Status.Conditions` should have a field `Ready` with the 
value set properly. The controller 
with matching `Provider` is responsible for monitoring and updating the 
`Status.Conditions` field and make it matching the status of the underlying
network. `Ready` should set to `True` if and only if the network can provide
all the functionalities it has promised. Otherwise, it should be set to
`False`. The default value is `Unknown`, which indicates that it has not been
updated by the proper controller yet.

### Dependency Management
When the CR of a network is referenced by any `PodNetwork` object, it is
the responsibility of the controller of the network CR to handle the
dependency properly. It may choose not to delete the corresponding network 
and the CR when the network is still referenced by any `PodNetwork` 
object, or it may set the `Status.Conditions` field to `False` when
the underlying network is deleted. 

When `PodNetwork` is referenced by objects of other resources, it is the  
responsibility of those resources' controlers to ensure dependency tracking, 
For example, they may use finalizers placed in corresponding `PodNetwork` 
objects.

## DRA Integration

This proposal standardizes the driver implementation of a DRA-based network 
with respect to its integration with the `PodNetwork` API. 

### Device Attribute for `PodNetwork`

This proposal defines one standard device attribute that can be used in the  
`ResourceSlice` resource. This attribute allows `ResourceClaims` to select  
a specific network. The DRA driver of the network should add this attribute
to the `ResourceSlice` objects it creates that represent the network. See
the example below.

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

### ResourceClaim Status

Upon allocation of a device representing the network to a pod requesting 
this network resource, the controller should include a reference to the
`PodNetwork` object in the status of the corresponding `ResourceClaim`. 
To ensure consistency, we require all conforming driver to populate 
the `status.devices.[].data` field with a `podNetwork` property, as defined:

```go
type podStatus struct {
	PodNetwork string `json:"podNetwork"`
}
```
See the example below.

Further, the network implementation should also fill out the 
``status.devices.[].data.networkdata` field, using the corresponding data
of the pod in the specific network. These include hardware address, interface
name and IPs, if applicable for the network. See the 
[definition of NetworkDeviceData](https://kubernetes.io/docs/reference/kubernetes-api/resource/resource-claim-v1/#NetworkDeviceData)

## Example

Consider an existing pod network implementation built from a `FooNetwork` CR:

```yaml
apiVersion: multinetwork.networking.x-k8s.io/v1alpha1
kind: FooNetwork
metadata:
  name: blue-net
  namespace: default
spec: 
  ...
```

The `PodNetwork` object representing this network can be:

```yaml
apiVersion: multinetwork.networking.x-k8s.io/v1alpha1
kind: PodNetwork
metadata:
  name: blue-net
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
      expression: device.attributes["multinetwork.networking.x-k8s.io"].podNetwork == "blue-net"
```

`FooNetwork`'s DRA driver should create a `ResourceSlice` object for each node, 
such as:

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
          string: blue-net
....

```

Note the `spec.basic.attributes[]` field in this `ResourceSlice` object.

Pods should use a `ResourceClaim` like the following to request this
network. The claim and its status, updated after the claim is bound
to the pod, is shown below:

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
      podNetwork: blue-net
    networkdata:
      hardwareAddress: 5a:9f:d8:84:fb:51
      interfaceName: net1
      ips:
      - 10.10.1.2/24
...

```

## Conformance Tests

The conformance tests validate that a network implementation correctly  
integrates with `PodNetwork` and the Kubernetes Resource API. Note that 
the tests here only verify properties related to this API. 

Conformance tests validate:
* `PodNetwork` object lifecycle.
  - For a `PodNetwork` object with matching `Provider`, the network
  controller should update the `Ready` field in `spec.Conditions` to
  `True` if the referenced network exists and works normally. If the
  referenced network does not exist, the controller should update the 
  `Ready` condition to `False`. 

  - When a network is deleted, the corresponding `PodNetwork` is either deleted
  or the `Ready` condition is set to `False`.

  - A `PodNetwork` with non-matching `Provider` field should NOT be updated
  by the controller.

* `ResourceSlice` attributes. 
  - The `ResourceSlice` resource advertised by the controller must include 
  attribute `multinetwork.networking.k8s.io/podNetwork`, with the value set
  to the proper `PodNetwork` object's name.

* `ResourceClaim` status reporting. 
  - The `ResourceClaim`'s device status, after the claim is bound to a pod,
  must include proper data and a `networkdata` field properbly populated.

## Reference Implementation

A reference implementation will be added later.

## Alternatives
### `NetworkKind` API (formerly NetworkClass)
There have been two former iterations of multi-network APIs introduced and debated.  
The latest iteration is the `NetworkKind` API (formerly NetworkClass). It is  
intended to provide a classification and discovery mechanism that allows Kubernetes  
APIs and controllers to recognize and integrate multiple pod networks. However, that  
API has proven inconvenient because `NetworkKind` alone cannot uniquely identify a  
specific network, and in practice, users still need the network name in an identifier.

### Original `PodNetwork`
The predecessor was the original `PodNetwork` proposal. There it sought to establish  
an abstract `Network` type as a standard base for all secondary network implementations.  
However, the extreme technical diversity of networks makes a unified abstract 
network type nearly impossible to define. Consequently, such an abstract  
network type saw significant pushback from the community.


