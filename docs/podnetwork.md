# PodNetwork

## Proposal
The PodNetwork resource is designed to serve as a unique identifier for individual pod networks. By maintaining a reference to a specific pod network implementation along with its essential descriptive data, this resource enables Kubernetes APIs and controllers to reference and identify specific networks both uniquely and efficiently.

The API remains agnostic to the underlying network implementation; the referenced object may be a CRD representing a network or one from which a network is instantiated. Consequently, the reference requires the group, kind, name and, optionally namespace of the target object.

PodNetwork is defined as a cluster-scoped resource. While the API allows for manual creation and updates, it is recommended that the network implementations should automatically manage a corresponding PodNetwork resource for every network the same implementation manages.

PodNetwork objects are immutable once created. Each network object must correspond to exactly one PodNetwork instance. The lifecycle of this object is strictly tied to the referenced implementation: it should be instantiated after the network object is created and removed prior to that object's deletion.

Once established, PodNetwork resources can be used as identifiers or labels for network-dependent attributes and resources. In multi-network environments, such attributes and resources may include pod IPs, gateways and services. While this API provides the framework for identification, it does not strictly dictate how these identifiers must be utilized. Future design examples and reference implementations will demonstrate the various ways PodNetwork objects can be integrated.

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

As part of its implementation, such CRD objects are managed by a dedicated controller. To integrate with the new PodNetwork API, this controller should generate a corresponding PodNetwork object for every network under its management. Based on the `FooNetwork` example provided, the resulting `PodNetwork` would be structured as follows:

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
