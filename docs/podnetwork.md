# Revised PodNetwork API Proposal

## Proposal

`PodNetwork` resource functions as an identifier for an individual pod network. It holds a reference to an object representing a specific pod network implementation, together with other necessary descriptive information of the network. The goal is to give controllers and other Kubernetes APIs a way to uniquely and conveniently identify and reference a specific pod network. 

The referenced object can be a CRD from which a pod network is created, or a CRD that represents a pod network, but its specifics is not required by this API. The reference only holds the group, kind and name of the referenced object.

`PodNetwork` is a cluster-scoped resource. In general, a specific pod network implementation is expected to create and manage a set of `PodNetwork` resources, one for each network it manages. However, the API does not enforce this, and it is possible for users to create and update `PodNetwork` resources directly, though this is tedious and error-prone.

A `PodNetwork` is immutable once created. One network object should only be represented by one `PodNetwork` object. A `PodNetwork` object should be created after the referenced network object is created, and deleted before the referenced network object is deleted.

Once created, `PodNetwork` objects can be used to identify or label resources or attributes that are related to a specific pod network. For example, in a multi-network cluster pods' IPs are only meaningful when associated with a specific pod network; gateways and services should also be defined with their networks explicitly specified. These cases are where `PodNetwork` objects are used as identifiers. This API does not demand how explicitly `PodNetwork` objects are used as identifiers. Rather, we will provide reference implementations and design examples to show how `PodNetwork` objects may be used.

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
    // Provider specifies the network provider responsible for the referenced network implementation.
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
    
    // Name of the referent.
    // +required
    Name string `json:"name"`
    
    // APIGroup of the referent.
    // +required
    ApiGroup string `json:"apigroup"`
    
    // Namespace of the referent.
    // +optional
    Namespace string `json:"namespace,omitempty"`
}
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

As part of its implementation, such a CRD is managed by its own controller. To leverage the newly introduced PodNetwork API, the controller can create a PodNetwork object for each network it manages. For the CRD proto above, A `PodNetwork` should be generated from the original `FooNetwork` CRD like this:

```yaml
apiVersion: multinetwork.networking.x-k8s.io/v1alpha1
kind: PodNetwork
metadata:
  name: foo-net-blue # Controller can decide the naming convention. Imutable nature of PodNetwork ensures that the name is unique for each network.
spec: 
  provider: foo.networking.com
  networkref:
    kind: FooNetwork
    name: blue-net
    apigroup: multinetwork.networking.x-k8s.io
    namespace: default
```

In general, the `FooNetwork` controller should manage `PodNetwork` in the following way:

```
FOR EACH FooNetwork ADDED:
  AddPodNetwork(FooNetwork)

FOR EACH FooNetwork TO-BE-DELETED:
  DeletePodNetwork(FooNetwork)

```
## Reference Implementation

A reference implementation will be added later.


## Background
Over the last few months, two primary iterations of multi-network APIs have been introduced and debated. The latest iteration is the `NetworkKind` API (formerly NetworkClass). It intended to offer a discovery and classification mechanism for Kubernetes controllers to manage multiple pod networks. The goal of that version is specifically stated as to “provide(s) a classification and discovery mechanism that allows Kubernetes APIs and controllers to recognize and integrate multiple pod networks”. Despite this goal, that API has proven inconvenient. `NetworkKind` alone cannot uniquely identify a specific network. In practice, users must employ an additional network name identifier. This reliance on a separate name identifier makes the `NetworkKind` largely redundant, providing utility only when identical network names are shared across different kinds.

The predecessor to `NetworkKind` was the original `PodNetwork` proposal. There it sought to establish an abstract `Network` type as a standard base for all secondary network implementations. While this approach would theoretically solve the identification issue mentioned above by using the `PodNetwork` object itself as the identifier, it faces a significant dilemma: the extreme technical diversity of networks (e.g., L2 vs. L3, RDMA vs. TCP/IP) makes a unified abstract network type nearly impossible to define. Any such type becomes so minimal that nearly all functional configuration must be relegated to implementation-specific key-value pairs, severely limiting its usefulness. Consequently, we observed significant pushback against adopting this API for both new and existing secondary-network implementations.

Drawing from community feedback on prior iterations and with insights gained from planning a reference implementation, we have identified two core guiding principles:

1. **Explicit network identification is highly valuable**: Such a mechanism enables resources like Services, Gateways, and Network Policies to explicitly target a particular network, satisfying a significant community demand.
2. **High-level network abstraction is very difficult**: Abstract network types, due to the technical diversity dilemma mentioned above, offer minimal utility to developers—particularly those managing established network implementations. Consequently, there is little incentive for these developers to migrate toward a universal, high-level abstraction of a `Network`.

In light of these observations, we proposed the current version of `PodNetwork` API that purely serves as a discovery and identification framework. Specifically, we encourage generating PodNetwork resources **from** existing implementations, rather than forcing network implementations to be built **upon** the PodNetwork type. This flexibility is intended to accelerate community adoption.





