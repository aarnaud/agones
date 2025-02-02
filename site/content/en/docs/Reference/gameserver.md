---
title: "GameServer Specification"
linkTitle: "Gameserver"
date: 2019-01-03T03:58:48Z
weight: 10
description: >
  Like any other Kubernetes resource you describe a GameServer's desired state via a specification written in YAML or JSON to the Kubernetes API. The Agones controller will then change the actual state to the desired state.
---

A full GameServer specification is available below and in the {{< ghlink href="examples/gameserver.yaml" >}}example folder{{< /ghlink >}} for reference :

```yaml
apiVersion: "stable.agones.dev/v1alpha1"
kind: GameServer
metadata:
  name: "gds-example"
spec:
  ports:
  - name: default
    portPolicy: Static
    containerPort: 7654
    hostPort: 7777
    protocol: UDP
  health:
    disabled: false
    initialDelaySeconds: 5
    periodSeconds: 5
    failureThreshold: 3
  template:
    metadata:
      labels:
        myspeciallabel: myspecialvalue
    spec:
      containers:
      - name: example-server
        image: gcr.io/agones/test-server:0.1
        imagePullPolicy: Always
```

Since Agones defines a new [Custom Resources Definition (CRD)](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) we can define a new resource using the kind `GameServer` with the custom group `stable.agones.dev` and API version `v1alpha1`.

You can use the metadata field to target a specific [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 
but also attach specific [annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) and [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to your resource. This is a very common pattern in the Kubernetes ecosystem.

The length of the `name` field of the Gameserver should not exceed 63 characters.

The `spec` field is the actual GameServer specification and it is composed as follow:

- `container` is the name of container running the GameServer in case you have more than one container defined in the [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/). If you do,  this is a mandatory field. For instance this is useful if you want to run a sidecar to ship logs.
- `ports` are an array of ports that can be exposed as direct connections to the game server container
  - `name` is an optional descriptive name for a port
  - `portPolicy` has {{< feature expiryVersion="0.11.0" >}}two{{< /feature >}}{{< feature publishVersion="0.11.0" >}}three{{< /feature >}} options:
        - `Dynamic` (default) the system allocates a random free hostPort for the gameserver, for game clients to connect to.
        - `Static`, user defines the hostPort that the game client will connect to. Then onus is on the user to ensure that the port is available. When static is the policy specified, `hostPort` is required to be populated.
{{% feature publishVersion="0.11.0" %}}        - `Passthrough` dynamically sets the `containerPort` to the same value a randomly selected hostPort. This will mean that users will need to lookup what port to open through the server side SDK before starting communications.{{% /feature %}}
  - `containerPort` the port that is being opened on the game server process, this is a required field{{< feature publishVersion="0.11.0" >}} for <code>Dynamic</code> and <code>Static</code> port policies, and should not be included in <code>Passthrough</code> configuration.{{< /feature >}}.
  - `protocol` the protocol being used. Defaults to UDP. TCP is the only other option.
- `health` to track the overall healthy state of the GameServer, more information available in the [health check documentation]({{< relref "../Guides/health-checking.md" >}}).
- `template` the [pod spec template](https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#podtemplatespec-v1-core) to run your GameServer containers, [see](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates) for more information.

## GameServer State Diagram

The following diagram shows the lifecycle of a `GameServer`. 

Game Servers are created through Kubernetes API (either directly or through a [Fleet]({{< ref "fleet.md" >}})) and their state transitions are orchestrated by:

- GameServer controller, which allocates ports, launches Pods backing game servers and manages their lifetime
- Allocation controller, which marks game servers as `Allocated` to handle a game session
- SDK, which manages health checking and shutdown of a game server session

![GameServer State Diagram](../../../diagrams/gameserver-states.dot.png)