## Introduction to Charts

This section aims to provide an introduction to the concept of Charts, their relation to Kubernetes and how this is relevant to all your deployments.

### Kubernetes Overview

Running web applications on Kubernetes requires the creation of multiple resources of different types.

Example Kubernetes resource types:

- **Pods**: Kubernetes does not manage containers directly, instead containers are grouped into a resource type called a `Pod`. This allows combining single purpose containers as modular components forming diverse and powerful web applications.
- **ReplicationControllers**: To ensure the desired count of Pods for a certain application version are running at all times, Kubernetes uses the `ReplicationController` concept. If a Pod dies (for whichever reason), the ReplicationController managing the Pod is responsible for creating another replica. (note: ReplicationControllers were superseded by `ReplicaSets` since v1.2, apart from supporting more advanced label queries they function mostly identical)
- **Deployments**: To orchestrate rolling updates of the applications running in Pods controlled by a ReplicaSet, the ReplicaSet controlling older version Pods needs to be slowly scaled down and a new ReplicaSet controlling newer version pods needs to be slowly scaled up. In Kubernetes, this orchestration is declarative and fully automated through `Deployment` resource types.
- **Services**: To expose the web applications running in a Kubernetes cluster, a `Service` resource type needs to be created. Service objects are used for discovering and routing trafic to each `Pod` resource in a way similar to a Layer 4 load balancer.

For version control, each resource type is stored as code in the form of a serialized object (under git as `json` or `yaml` format).

As the number of technology stacks increases, managing these bundles of seralized objects becomes a challenge, enter Helm.

### Kubernetes Helm Overview

To manage the different resource types as well as define packages of multiple resources configured to work together, Honestbee uses the powerfull k8s package manager called `Helm`. Helm allows us to bundle the serialized form of all the resources making up a web application and template out the parameters.

The resulting packages are referred to as `Charts` and allow deployments of tech stacks meeting different requirements with varying degrees of complexity.

Helm is also used to search repository servers for specific charts and manage the Charts installed to our Kubernetes cluster (this includes managing the rolling updates of our applications).