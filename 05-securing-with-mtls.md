[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #TLS #mutual-tls #PKI-domains #ingress-gateway #VirtualService #Gateway #PeerAuthentication #DestinationRule)

# Securing with Mutual TLS

## Learning goals

- Secure communication between services
- Secure communication with mesh external services

## Introduction

These exercises will demonstrate how to use mutual TLS inside the mesh between
PODs with the Istio sidecar injected and also from mesh-external services
accessing the mesh through an ingress gateway.

You will be using several Istio custom resource 
definitions([CRD's](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)) 
for this.

- [PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/#PeerAuthentication-MutualTLS) - 
Applies to requests that a service **receives**

- [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule) - 
What type of TLS sidecar **sends**

- [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings) - 
Specify TLS settings for external service

Istio provides two types of authentication policies of which *peer 
authentication* is one of them. 

Peer authentication is used for **service-to-service** authentication 
to verify the client making the connection and secure the 
service-to-service communication. Istio uses 
[mutual TLS](https://en.wikipedia.org/wiki/Mutual_authentication)(mTLS) 
for transport authentication. 

This provides a **strong identity** for each workload. 

> Istio has it's own certificate authority(CA) and securely provisions strong 
> identities to every workload with X.509 certificates.

<details>
    <summary> More About Workload Identity </summary>

Istio provisions keys and certificates through the following flow:

- istiod offers a gRPC service to take certificate signing requests (CSRs).

- When started, the Istio agent creates the private key and CSR, and then 
sends the CSR with its credentials to istiod for signing.

- The CA in istiod validates the credentials carried in the CSR. Upon 
successful validation, it signs the CSR to generate the certificate.

- When a workload is started, Envoy requests the certificate and key from 
the Istio agent in the same container via the Envoy secret discovery 
service (SDS) API.

- The Istio agent sends the certificates received from istiod and the private 
key to Envoy via the Envoy SDS API.

- Istio agent monitors the expiration of the workload certificate. 

> The above process repeats periodically for certificate and key rotation.

</details>

Peer authentication is normally enabled at the `mesh` level, as in our 
training infrastructure, which means traffic is secured by **default** 
for all services in the cluster **without** requiring code changes. 

There are four modes that can be set for mTLS. They are `UNSET`, `DISABLE`, 
`PERMISSIVE` and `STRICT`.

<details>
    <summary> More About mTLS Modes </summary>

- `UNSET` - Modes is inherited from parent, defaults to PERMISSIVE

- `DISABLE` - Connection is **not** tunneled 

- `PERMISSIVE` - Connection can be either plaintext or mTLS tunnel

- `STRICT` - Connection is an mTLS tunnel (TLS with client cert must be presented)

> PeerAuthentication defines how traffic will be tunneled (or not) to the sidecar.

</details>

> :bulb: If you have not completed exercise 
> [00-setup-introduction](00-setup-introduction.md) you **need** to label 
> your namespace with `istio-injection=enabled`.

## Exercise: Mutual TLS Inside the Mesh

You will deploy all the services for the sentences application to observe 
the affects of mTLS configuration. You will first observe that the default 
mesh wide configuration will be inherited by your namespace. Afterwards you 
will apply mTLS configuration to **your** namespace and observe the affects 
of STRICT and PERMISSIVE policies.

### PeerAuthentication

The PeerAuthentication CRD is used to specify the mTLS settings for a workloads 
**requests**.

An example of a peer authentication policy is seen below:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: foo
spec:
  mtls:
    mode: PERMISSIVE
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: foo
spec:
  selector:
    matchLabels:
      app: myapp
  mtls:
    mode: STRICT
```

The above policy says to allow **both** plain-text and mTLS traffic for **all** 
workloads in the `foo` namespace but **require** mTLS for the workload `myapp` 
in the `foo` namespace.

- The `namespace` definition scopes the policy to the defined namespace.
If a policy is **not** namespace scoped it applies to **all** workloads 
within the mesh.

> There can be only one **mesh-wide** peer authentication policy, and only one 
> **namespace-wide** peer authentication policy per namespace. If you configure 
> more then Istio will **ignore the newer policies**.

- The `selector` determines the workload to apply the PeerAuthentication to.

> You can also specify the mTLS mode at the **port** level **if** you specify 
a `selector`. 

### DestinationRule

The DestinationRule CRD is used to specify the TLS settings for upstream 
connections. E.g. traffic the workload sends. These settings are common for 
both HTTP and TCP connections. 

In a destination rule you use the `tls` keyword instead of the `mtls` keyword 
under `spec.trafficPolicy`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: upstream-mtls
spec:
  host: upstream-app
  trafficPolicy:
    tls:                        # tls instead of mtls
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

The modes are `DISABLE`, `SIMPLE`, `MUTUAL` and `ISTIO_MUTUAL`. For services 
within the mesh `ISTIO_MUTUAL` is the natural choice as it uses Istio's 
generated certs and they do not need to be specified in the destination rule.

<details>
    <summary> More About TLS Modes </summary>

- `DISABLE` - Do not setup a TLS connection to the upstream endpoint.

- `SIMPLE` - Originate a TLS connection to the upstream endpoint.

- `MUTUAL` - Secure connections to the upstream using **mutual** TLS by 
presenting client certificates for authentication.

> You **must** specify the certificates when modes is set to MUTUAL.

- `ISTIO_MUTUAL` - Same as MUTUAL except you do **not** specify the 
client certificates as the certs generated for mTLS by Istio are used.

</details>

### Overview

A general overview of what you will be doing in the **Step By Step** section.

- Deploy the sentences application services and see the effect of mesh wide mTLS config

- Apply a STRICT and PERMISSIVE mTLS policy to **your** namespace and observe 
the affects

- Observe the affects on a service without a sidecar

- Configure TLS to an upstream service

### Step by Step

Expand the **Tasks** section below to do the exercise.

<details>
    <summary> Tasks </summary>

#### Task: Deploy the sentences application and observe sidecars

___


Run a for loop to substitute placeholders for environment variables and deploy 
the services along with an ingress gateway and virtual service for routing 
inbound traffic.

```console
for file in 05-securing-with-mtls/start/*.yaml; do envsubst < $file | kubectl apply -f -; done
```

Observe that the services are ready and we have a sidecar container per POD and 
the sentences service is exposed with NodePort.

> :bulb: We have also configured an entry point with a Gateway and VirtualService 
> but are also exposing it as a NodePort to demonstrate the effects of the policies.

```console
kubectl get pods,svc
```

It should look

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/age-v1-6d5946d477-dbx4j         2/2     Running   0          24s
pod/name-v1-6d59555fff-m9khm        2/2     Running   0          23s
pod/name-v2-7bffd6cd8f-46jwb        2/2     Running   0          23s
pod/sentences-v1-6bc9b4875b-rm6bj   2/2     Running   0          22s

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/age         ClusterIP   172.20.134.163   <none>        5000/TCP         25s
service/name        ClusterIP   172.20.59.167    <none>        5000/TCP         24s
service/sentences   NodePort    172.20.92.234    <none>        5000:30962/TCP   23s
```

#### Task: Run the script `./scripts/loop-query.sh`

___


Run the loop-query.sh script without parameters to reach the service over the 
NodePort.

```console
./scripts/loop-query.sh
```

#### Task: Observe the traffic flow with Kiali

___

Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Check the display options as shown in the image below. Especially the **Security** 
checkbox. 

Notice the lock icons, which indicate whether the traffic is encrypted between 
the services.

Select one of the **arrows** with a lock icon. You will be able to see on the 
sidebar that the traffic is mTLS secured. Notice that traffic from the 
`loop-query.sh` script(unknown) to the sentences frontend service is plain text 
HTTP traffic as we are using a NodePort to access it.

![Kiali default mTLS](images/kiali-mtls-default.png)

#### Task: Create peer authentication policy in your namespace

___

Create a file called `peer-authentication.yaml` in 
`05-securing-with-mtls/start/`.

Paste the following yaml into the file.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: $STUDENT_NS
  namespace: $STUDENT_NS
spec:
  mtls:
    mode: STRICT
```

> :bulb: Note `$STUDENT_NS` in the above yaml. It is important that the 
> PeerAuthentication CRD is scoped to **your individual** namespace. There can 
> only be one mesh wide and one namespace wide policy at a time.

Substitute the placeholder with the value of the environment variable and apply 
the output with kubectl.

```console
envsubst < 05-securing-with-mtls/start/peer-authentication.yaml | kubectl apply -f -
```

You should lose your connection.

```
Aramis (v2) is 52 years.
Egon is 60 years.
Aramis (v2) is 35 years.
curl: (56) Recv failure: Connection reset by peer
```

You just applied a STRICT policy **requiring** all traffic to be over mTLS but 
are accessing the frontend service directly over a NodePort and the traffic is 
plain-text over HTTP.

Change the mode to `PERMISSIVE` in the `peer-authentication.yaml` file you just 
created.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: $STUDENT_NS
  namespace: $STUDENT_NS
spec:
  mtls:
    mode: PERMISSIVE
```

Apply the changed policy.

```console
envsubst < 05-securing-with-mtls/start/peer-authentication.yaml | kubectl apply -f -
```

#### Task: Observe the traffic flow with Kiali

___

Re-run the `loop-query.sh` script as before so we access the sentences frontend 
service over the NodePort as before.

```console
./scripts/loop-query.sh
```

Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Check the display options as shown in the image below. Especially the 
**Security** checkbox. 

Notice that traffic is plain text HTTP traffic and is now allowed as 
`PERMISSIVE` mode allows both plain-text and mTLS.

![Kiali default mTLS](images/kiali-mtls-default.png)

#### Task: Route traffic through the gateway

___

An ingress gateway entry point in the `istio-ingressgateway` was
created when you deployed the sentence application services in the
first task. Route the traffic through the gateway by running the
loop-query.sh script with the option -g.

```console
./scripts/loop-query.sh -g $STUDENT_NS.sentences.$TRAINING_NAME.eficode.academy
```

#### Task: Observe the traffic flow with Kiali

___

Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Select the `istio-ingress` namespace along with your namespace and select the
display checkboxes as shown below.

![Kiali with no sidecars](images/kiali-mtls-gw.png)

We can now see that all the traffic within the mesh is mTLS. As the traffic is 
now being routed through the ingress gateway it is being secured over mTLS by 
the gateways standalone envoy proxy towards the sentences frontend service. 

When `PERMISSIVE` mode is enabled both plain-text traffic and mTLS traffic are 
allowed but mTLS **will be applied where possible**. 

There is no real reason, besides for demonstration purposes, to expose the 
sentences frontend service as a NodePort. Change the sentences service type to 
`ClusterIP` in the `sentences.yaml` file in `05-securing-with-mtls/start` folder.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sentences
    mode: sentence
    app.kubernetes.io/part-of: sentences
  name: sentences
spec:
  ports:
  - name: http-sentences
    port: 5000
    protocol: TCP
    targetPort: 5000
    appProtocol: http
  selector:
    app: sentences
    mode: sentence
  type: ClusterIP       # <---------
---
```

#### Task: Apply STRICT policy

___

As all of the traffic is now going through the ingress gateway you can apply 
`STRICT` mode to your policy now. Change it in the `peer-authentication.yaml` 
file you created.

Apply your changes.

```console
for file in 05-securing-with-mtls/start/*.yaml; do envsubst < $file | kubectl apply -f -; done
```

> While migrating an application to full mTLS, it may be useful to start with 
> a `PERMISSIVE` mTLS mode which allow a mix of mTLS and un-encrypted and 
> un-authenticated traffic.

#### Task: Disable sidecar for the age service

___

What do you think will happen if a service has **no sidecar**?

Modify the age service so it has no sidecar by adding an annotation to the file 
`age.yaml` in `05-securing-with-mtls/start` folder to disable sidecar injection.

The deployment section of the file should look like.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: sentences
    mode: age
    version: v1
  name: age-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sentences
      mode: age
      version: v1
  template:
    metadata:
      labels:
        app: sentences
        mode: age
        version: v1
      annotations:                          # Annotations block
        sidecar.istio.io/inject: 'false'    # True to enable or false to disable
    spec:
      containers:
      - image: praqma/istio-sentences:v1
        name: age
        env:
        - name: "SENTENCE_MODE"
          value: "age"
---
```

Apply the change.

```console
kubectl apply -f 05-securing-with-mtls/start/age.yaml
```

Check that POD for the age service has no sidecar.

```console
kubectl get pod
```
You should see it terminating and a new one running with only one container.

```
NAME                                READY   STATUS        RESTARTS   AGE
pod/age-v1-54fc4584d6-k94zk         1/1     Running       0          13s
pod/age-v1-6d5946d477-f7fdp         2/2     Terminating   0          57m
```

You really shouldn't see any interruptions from the `loop-query.sh` script 
running in the terminal.

If a POD has no sidecar then the PeerAuthentication policy **cannot be applied**.

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Select the `istio-ingress` namespace along with your namespace and select the
display checkboxes as shown below.

As the age service has no sidecar you will, **eventually**, not be able to see 
traffic flowing to the age service in the graph even though it is still flowing 
over HTTP.

![Kiali mTLS no age sidecar](images/kiali-mtls-no-age-service.png)

Remove the annotation to disable sidecar injection from the age service and redeploy it.

```console
kubectl apply -f 05-securing-with-mtls/start/age.yaml
```

#### Task: Configure TLS to an upstream service

___


To show how we can control upstream mTLS settings with a DestinationRule, we
create one that uses mTLS towards `v2` of the `name` service and no mTLS for
`v1`. 

> :bulb: Note that we now need to use a `PERMISSIVE` Policy.

Modify the `peer-authentication.yaml` in `05-securing-with-mtls/start/` 
to use `PERMISSIVE` mode.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: $STUDENT_NS
  namespace: $STUDENT_NS
spec:
  mtls:
    mode: PERMISSIVE
```

Substitute the placeholder with the value of the environment variable and apply 
the output with kubectl.

```console
envsubst < 05-securing-with-mtls/start/peer-authentication.yaml | kubectl apply -f -
```

Note, that a DestinationRule *will not* take effect until a route rule
explicitly sends traffic to a subset. So you need both a virtual 
service and a destination rule. 

Create a file called `name-vs-dr.yaml` in `05-securing-with-mtls/start/` 
directory.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: name
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
        port:
          number: 5000
        subset: name-v1
      weight: 50
    - destination:
        host: name
        port:
          number: 5000
        subset: name-v2
      weight: 50
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: name
spec:
  host: name
  exportTo:
  - "."
  subsets:
  - name: name-v1
    labels:
      version: v1
    trafficPolicy:
      tls:
        mode: DISABLE         # Plain text traffic
  - name: name-v2
    labels:
      version: v2
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL    # Mutual TLS using Istio generated certs
```

Apply the virtual service and destination rule with the mTLS settings.

```console
kubectl apply -f 05-securing-with-mtls/start/name-vs-dr.yaml
```

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. Select the checkboxes as shown in the below image.

Now we see a missing padlock on the traffic towards `v1` of the name service. 
(It might take a second to update, but if you select the connection,
you may already be able to see the "percentage of mTLS" traffic decreasing.)

![No mTLS towards v1](images/kiali-mtls-destrule-anno.png)

</details>

## Exercise: Mutual TLS From External Clients

This exercise extends what we have done so far to demonstrate a common 
use case of configuring mTLS with a service **external** to the mesh. A 
service running in a different mesh for example. 

In order to do this we will create a Certificate authority and certificates 
to enable strong workload identity.

The TLS settings to control this will be defined in the 
[Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/#Gateway) 
CRD.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-unique-gateway
  namespace: istio-ingress
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: MUTUAL
      credentialName: server-side-tls-secret
    hosts:
    - "myapp.example.com"

```

- The port and tls blocks define the protocol and TLS mode to apply the 
configuration to. In this exercise you will be using the `SIMPLE` and 
the `MUTUAL` TLS modes. See this link for more information about 
[Server TLS modes](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode).

- The credentialName represents a kubernetes secret containing the server side 
certificate.

> :bulb: The secret **must** be in the **same** namespace as the ingress gateway 
> controller. The gateway must be **namespaced** to be in the same namespace as 
> the kubernetes secret. So **both** must be in the `istio-ingress` namespace in 
> the above example and the gateway **must** be unique for this exercise.

### Overview

A general overview of what you will be doing in the **Step By Step** section.

- Generate needed certificate authority(CA) and certificates 

- Require `MUTUAL` TLS for port `443`

- Modify the virtual service to point to the gateway in `istio-ingress` namespace

- Test the traffic using a script in both TLS and mTLS modes

### Step by Step

Expand the **Tasks** section below to do the exercise.

<details>
    <summary> Tasks </summary>

#### Task: Delete the gateway created in the previous exercise

___

Ensure that the gateway and virtual service for the `sentences` 
service from the first exercise is removed.

```console
kubectl delete -f 05-securing-with-mtls/start/sentences-ingress-gw.yaml
kubectl delete -f 05-securing-with-mtls/start/sentences-ingress-vs.yaml
```

#### Task: Generate needed certificate authority(CA) and certificates

___


Execute the `generate-certs.sh`script.

```console
./scripts/generate-certs.sh
```
You should see namespace specific certs for both the server side and 
client side in your workspace.

There should also be a kubernetes secret in the `istio-ingress` namespace. 

```console
kubectl get secrets -n istio-ingress
```

It should be pre-pended with your namespace and look something like below.

```
NAME                              TYPE      DATA   AGE
student1-sentences-tls-secret        Opaque    3      112m
```
> :bulb: DO NOT touch any other secrets in the `istio-ingress` namespace!

#### Task: Modify the gateway to configure `MUTUAL` TLS for port `443`

___


Modify the file `05-securing-with-mtls/start/sentences-ingress-gw.yaml` so 
that it will be namespaced to the `istio-ingress` namespace and configure it 
for `MUTUAL` TLS on port 443.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: $STUDENT_NS-sentences                           # <----- Use your namespace to make it unique
  namespace: istio-ingress                              # <----- Must be in the istio-ingress namespace
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:                                               # <----- Add the port block with the port and protocol
      number: 443
      name: https
      protocol: HTTPS
    tls:                                                # <----- Add the tls block
      mode: MUTUAL                                      # <----- TLS mode
      credentialName: $STUDENT_NS-sentences-tls-secret  # <----- Add the kubernetes secret
    hosts:
    - "$STUDENT_NS.sentences.$TRAINING_NAME.eficode.academy"
```

#### Task: Modify the virtual service to point to the gateway in `istio-ingress` namespace

___


The gateway is now namespaced to the `istio-ingress` namespace so you need 
to tell the virtual service which namespace the gateway is located in.

Modify the file `05-securing-with-mtls/start/sentences-ingress-vs.yaml`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: sentences
spec:
  hosts:
  - "$STUDENT_NS.sentences.$TRAINING_NAME.eficode.academy"
  gateways:
  - istio-ingress/$STUDENT_NS-sentences  # <----- Use the gateway in the istio-ingress namespace
  http:
  - route:
    - destination:
        host: sentences
```

#### Task: Apply the changes to the virtual service and gateway

___


Once you have done the modifications to the gateway and virtual service, 
substitute the placeholders with environment variable(s) and apply with kubectl.

```console
envsubst < 05-securing-with-mtls/start/sentences-ingress-gw.yaml | kubectl apply -f -
envsubst < 05-securing-with-mtls/start/sentences-ingress-vs.yaml | kubectl apply -f -
```

#### Task: Run `loop-query-mtls.sh https+mtls`

___


The `loop-query-mtls.sh` script uses the `client` certificates that were 
generated for the mtls connection. 

```console
scripts/loop-query-mtls.sh https+mtls
```

Note the curl options printed when the script starts. The certificate 
authority **and** the client cert, which is required for mTLS, are specified.

It will look *something* like below but with the CA and client certs that can 
be found in your workspace as the below has been edited for simplicity.

```
-------------------------------------
Using ingress gateway with label: app=istio-ingressgateway
Using URL: https://student1.sentences.istio.eficode.academy:443/
Using curl options: '--resolve sentences.istio.eficode.academy:443 --cacert eficode.academy.crt --cert client.crt --key client.key'
-------------------------------------
```

#### Task: Run `loop-query-mtls.sh https`

___


```console
scripts/loop-query-mtls.sh https
```

**You should see an OpenSSL error**! Note the curl options printed when the 
script starts. **Only** the **certificate authority** is passed as simple TLS 
only requires server side authentication. But we have configured our gateway 
to use `MUTUAL` TLS. E.g.both client and server side authenticate.

```
-------------------------------------
Using ingress gateway with label: app=istio-ingressgateway
Using URL: https://student1.sentences.istio.eficode.academy:443/
Using curl options: '--resolve student1.sentences.istio.eficode.academy:443 --cacert eficode.academy.crt'
```

Change the TLS mode to `SIMPLE` and apply the change.

```console
envsubst < 05-securing-with-mtls/start/sentences-ingress-gw.yaml | kubectl apply -f -
```

Plain `https` will now work. 

> You can still pass the `https+mtls` argument to the script and a connection 
> will be made. Only server side authentication will be done. E.g. Istio will 
> simply **ignore** the client certs passed through the curl options.

</details>

### Summary

PKI (Public Key Infrastructure) does not necessarily mean, that we are using
internet-scoped public/private key encryption. In this exercise we have seen how
we can leverage Istio's **internal PKI** to implement mTLS inside the Istio mesh
between PODs with Istio sidecars. We have also seen how to setup mTLS for Istio
ingress gateways.

The main takeaways are:

- Peer authentication policies apply to traffic a workload **receives**

- Peer authentication policies only apply to workloads with a sidecar

- Peer authentication policies apply to the **all** workloads within the 
mesh if **not** namespaced

- The `STRICT` policy means strict and a `PERMISSIVE` policy is a good way to get started

- Destination rules configure TLS for a workloads upstream traffic

- When securing external traffic through ingress gateways you need to 
consider namespaces in relation to kubernetes secrets and gateway controllers.

## Cleanup

```console
for file in 05-securing-with-mtls/start/*.yaml; do envsubst < $file | kubectl delete -f -; done
```
