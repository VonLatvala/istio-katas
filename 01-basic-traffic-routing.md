[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #sentences #kiali)

# Routing Traffic with Istio

## Learning goals

- Using Virtual Services
- Using Destination Rules

## Introduction

This exercise introduce you to the basics of traffic routing with Istio. 
We are going to deploy all the services for our sentences application 
and a new version `v2` of the **name** service. This will demonstrate normal 
kubernetes load balancing between services. 

Then we are going to use two Istio custom resource definitions([CRD's](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)) which are
the building blocks of Istio's traffic routing functionality to route traffic to 
the desired workloads.

These are the [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/) and the [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/) CRD's.

> :bulb: If you have not completed exercise 
> [00-setup-introduction](00-setup-introduction.md) you **need** to label 
> your namespace with `istio-injection=enabled`.

### VirtualService

You use virtual services to route traffic to kubernetes services. 

A VirtualService is used in addition to the normal Kubernetes service object.
A VirtualService defines a set of traffic routing rules to apply when a host 
is addressed. 

Each routing rule defines matching criteria for traffic of a 
specific protocol. If the traffic is matched, then it is sent to a named 
destination **service** or subset/version of it.

> **Expand the example below for more details.**

<details>
    <summary> Example </summary>

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myservice-route
spec:
  hosts:
  - myservice               # Apply these rules for traffic to this host
  gateways:
  - mesh
  http:
  - route:
    - destination:
        host: myservice     # Send to this Kubernetes Service
        subset: v1          # but only this subset of the Service
```

- The **http** block is an [HTTPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute) 
containing the routing rules for HTTP/1.1, HTTP/2 and gRPC traffic. 

> You can also use [TCPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#TCPRoute) 
> and [TLSRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#TLSRoute) 
> blocks for configuring routing.

- The **hosts** field is the user addressable destination that the routing rules 
apply to. It is the address used by a client when attempting to connect to a service.
This is **virtual** and doesn't actually have to exist. For example 
You could use it for consolidating routes to all services for an application. 

- The **destination** field specifies the **actual** destination of the routing 
rule and **must** exist. In kubernetes this is a **service** and generally 
takes a form like `reviews`, `ratings`, etc.
    
- The **weight** field can be used to specify the distribution of traffic,
    Within a single route.

- The `mesh` field in the gateways block is a reserved keyword used to imply 
**all** sidecars in the mesh.

- The subset block is the name of a subset within the service and **must** be 
defined in a **DestinationRule**.

</details>

### DestinationRule

Destination rules configure **what** happens to traffic for a destination 
defined in a virtual service.

You can think of virtual services as **how** you route your traffic to a given 
destination, and then you use destination rules to configure **what** happens 
to traffic for that destination.

One of the most common uses of `DestinationRule` is to specify named service **subsets**.

For example, grouping all of a service instances **versions**. You can then 
use these **subsets** in a virtual service to control traffic to different versions.

> **Expand the example below for more details.**

<details>
    <summary> Example </summary>

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: myservice      # These rules apply to this host. Typically a Kubernetes Service
  subsets:
  - name: v1           # Define a named subset 'v1'
    labels:
      version: v1      # Labels on 'host', e.g. labels on the Kubernetes Service
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

> Destination rules are applied by Istio **after** virtual service routing 
> rules are **evaluated**, so they apply to the traffic’s “real” destination.

> :bulb: If you have not completed exercise 
> [00-setup-introduction](00-setup-introduction.md) you **need** to label 
> your namespace with `istio-injection=enabled`.

</details>

## Exercise: Routing To Specific Version

We will look a bit into how a VirtualService and a DestinationRule can control 
the flow of traffic in your namespace. We spin up two versions of the name 
service in the namespace, and use the VirtualService and DestinationRule kinds 
to control the traffic.

### Overview

A general overview of what you will be doing in the **Step By Step** section.

- Deploy the sentences application services with kubectl

> You will deploy two versions of the `name` service.

- Producing traffic to the sentences application

- Use Kiali to observe the traffic flow

- Create a VirtualService to route **all** traffic to the native k8's service 
for name

- Create a DestinationRule to send traffic to a specific version(workload) of 
the name service

### Step By Step

Expand the **Tasks** section below to do the exercise.

<details>
    <summary> Tasks </summary>

#### Task: Deploy sentences app WITH TWO VERSIONS of the name services

___


```console
kubectl apply -f 01-basic-traffic-routing/start/
kubectl apply -f 01-basic-traffic-routing/start/name-v1
kubectl apply -f 01-basic-traffic-routing/start/name-v2
```

#### Task: Run loop-query.sh

___


```console
./scripts/loop-query.sh
```

#### Task: Observe the traffic in Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu.

![50/50 split of traffic](images/kiali-blue-green-anno.png)

What you are seeing here is kubernetes load balancing between PODS.
Kubernetes, or more specifically the `kube-proxy`, will load balance in 
either a *round robin* or *random* pattern depending on whether it is 
running in *user space* proxy mode or *IP tables* proxy mode.

You rarely want traffic routed to two version in an uncontrolled 
fashion.

So why is this happening?

> :bulb: Take a look at the label selector for the name service.
> It doesn't specify a version.  Compare this with the labels on the `name` Deployments by using `kubectl get deploy --show-labels`

#### Task: Create a destination rule and apply it

___


Create a destination rule called `name-dr.yaml` in 
`01-basic-traffic-routing/start/` and apply it.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: name-destination-rule
spec:
  host: name       # Apply to the name Kubernetes Service
  exportTo:
  - "."
  subsets:
  - name: name-v1  # Define a subset 'name-v1'
    labels:
      version: v1  # That consists of the destinations (i.e. Pods) with this label
  - name: name-v2
    labels:
      version: v2
```
The above destination rule says, when combined with a virtual service, **what** 
I want to do is send traffic to a workload **labeled** with either `v1` or `v2`.

```console
kubectl apply -f 01-basic-traffic-routing/start/name-dr.yaml
```
Applying the destination rule has no effect at this point because there is no 
virtual service including the destination rule.

> :bulb: To avoid 503 errors **always** apply destination rules and changes to 
> destination rules **prior** to changing virtual services.

#### Task: Create a `VirtualService` to route ALL traffic to version 1 of the name service

___


Create a virtual service called `name-vs.yaml` in 
`01-basic-traffic-routing/start/` and apply it.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: name-route
spec:
  hosts:
  - name
  exportTo:
  - "."
  gateways:
  - mesh
  http:
  - route:
    - destination:
        host: name
        subset: name-v1
```

> The `host` field in the above yaml is the kubernetes short name for the service. 
> Istio will translate the short name based one the **namespace** of the rule. 
> E.g. if the virtual service is in namespace `default` the short name name will 
> be interpreted as `name.default.svc.cluster.local`. What will happen if the 
> **name** service is in the namespace `student1`?

```console
kubectl apply -f 01-basic-traffic-routing/start/name-vs.yaml
```

Immediately after applying this, you will see the output from the
`loop-query.sh` change and strings with '(v2)' in them are no longer
received.

Go to **Graph** menu item in Kiali and select the **Versioned app graph** 
from the drop down menu and observe the traffic flow. It may take a minute 
before fully complete but you should see the traffic being routed to the 
`name-v1` **service**.

> :bulb: Make sure to select `Idle Edges`, `Service Nodes` and 
> `Virtual Services` in the Display drop down.

![Basic virtual service route](images/basic-route-vs.png)

You can see in Kiali that the virtual service combined with the destination 
rule subsets routes traffic to the name workload labeled `v1` even though the 
name service has no versions defined in the selector.

#### Task: Add a route to version 2 of the name service as the **first** route

___

Add a destination to the new service in the `name-vs.yaml` you 
created before. But place it **before** the `name-v1` service and apply it.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: name-route
spec:
  hosts:
  - name
  exportTo:
  - "."
  gateways:
  - mesh
  http:
  - route:
    - destination:
        host: name
        subset: name-v2
  - route:
    - destination:
        host: name
        subset: name-v1
```

```console
kubectl apply -f 01-basic-traffic-routing/start/name-vs.yaml
```

Immediately after applying this, you will see the output from the
`loop-query.sh` change and now only strings with '(v2)' in them are
received.

#### Task: Use the versioned app graph to observe route precedence in Kiali

___


Go to **Graph** menu item in Kiali and select the **Versioned app graph** 
from the drop down menu and observe the traffic flow. You will see that 
traffic is now being routed to the version 2 service.

![Routing precedence](images/basic-route-precedence-vs.png)

Routing rules are evaluated in **sequential** order from top to bottom, with 
the first rule in the virtual service definition being given highest priority. 

Reorder the destination rules so that service `name-v1` will be evaluated 
first and apply the changes.

```console
kubectl apply -f 01-basic-traffic-routing/start/name-vs.yaml
```

Go to **Graph** menu item in Kiali and select the **Versioned app graph** 
from the drop down menu and observe the traffic flow.Traffic should once more 
be routed to the `name-v1` service.

![Virtual service and destination rule](images/basic-route-vs.png)

</details>

# Summary

In this exercise you learned what a virtual service is and how to route traffic 
to a destination service in kubernetes with an 
[HTTPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute).

> Kubernetes only supports traffic distribution based on instance scaling. I.e. with the Kubernetes Service definition, we would need to scale the `name-v1` or `name-v2` Deployments to shift traffic.

There is a lot more virtual services can do for traffic distribution like 
match conditions for HTTPRoute on headers, uri, schemes, etc. HTTP redirects, 
rewrites and more. We will looking at some of these in following exercises.

See the [documentation](https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService) 
for more details.

You also saw how a destination rule could be used to determine **what** 
happens to traffic routed to a kubernetes service using labels identifying 
workload versions. But you can also set **traffic policies** on a destination 
rule to apply load balancing policies, connection pool settings, etc.

See the [documentation](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule) 
for more details.

Main takeaways are:

* A virtual services **hosts** field is a user addressable destination and can 
be virtual.

* A virtual services **destination** must exist exist. In Kubernetes this will 
be a Kubernetes Service.

* A destination rule defines traffic policies that apply for the intended 
service **after** routing has occurred with a virtual service. E.g. route 
all traffic to v1, different load balancing modes for v1 and v2, etc.

# Cleanup

```console
kubectl delete -f 01-basic-traffic-routing/start/name-v2
kubectl delete -f 01-basic-traffic-routing/start/name-v1
kubectl delete -f 01-basic-traffic-routing/start/
```
