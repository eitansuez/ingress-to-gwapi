# Migration

The main concerns relating to migrating from Ingress to the Gateway API are:

1. Identifying and selecting a controller that implements the new Kubernetes Gateway API.
1. Translating the Ingress configurations to Gateway API resources.
1. Vetting the revised configurations.
1. Switching over from the old gateway to the new one without downtime.

In addition, it is desirable to have the flexibility to transition one host name at a time.
In other words, in large enterprise environments, we need to accommodate teams transitioning to the new API at their own pace.

## The Gateway API

The Kubernetes Gateway API improves upon the Ingress model in a number of ways:

- Routing and Gateway-specific concerns are separate concerns, and are configured through distinct resources.  Routes attach to Gateways.
- Personas are taken into account: platform administrators configure gateways while applications teams self-service routing rules for their apps.
- The creation of the Gateway resource also provisions the gateway, on-demand.

This last point affords us the flexibility to use either shared gateways or dedicated ones for specific applications.
Certain applications require a higher degree of isolation.  This can be imposed by regulatory or other requirements, where traffic flowing to the target application cannot pass through a shared component, perhaps for security reasons, or to avoid other types of issues that stem from "noisy neighbors."
On the other hand, for applications that do not require it, it's simpler and more cost effective to use a single shared gateway.

## Gateway API compatible controller

The Gateway API documentation [lists](https://gateway-api.sigs.k8s.io/implementations/) implementations that conform to it.
For this migration we will use [kgateway](https://kgateway.dev/), an open-source project recently contributed to the CNCF.

### Install kgateway

Let us use the instructions from the [quickstart](https://kgateway.dev/docs/quickstart/) to install kgateway.

1. Apply the Gateway API CRDs:

    ```shell
    kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
    ```

1. Install kgateway's own CRDs with Helm:

    ```shell
    helm upgrade -i --create-namespace --namespace kgateway-system --version v2.0.0 kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds
    ```

1. Install kgateway:

    ```shell
    helm upgrade -i --namespace kgateway-system --version v2.0.0 kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway
    ```

We can verify the installation by listing deployments in the newly-created `kgateway-system` namespace:

```shell
kubectl get deploy -n kgateway-system
```

We can also list GatewayClass resources, which are the Gateway API's equivalent to the IngressClass concept:

```shell
kubectl get gatewayclass
```

The output will show a GatewayClass named `kgateway` (and another one for provisioning waypoints, which is out of scope at the moment).

We now have a running control plane.

The next step is to provision a gateway; for that we use the Gateway resource.

## Translate the Ingress configurations

Before we dive in to creating the necessary Gateway API resources, we should consider options.

We can try to use a tool to translate the existing Ingress resources.
When the number or quantity of configuration files is large, and where the alternative of drafting the configurations by hand can be both tedious and error-prone, using a tool is usually the better option.
On the other hand, tools are often incomplete, they don't always implement a spec fully, and are more mechanistic, in that they don't have intimate understanding of the original and target APIs and their nuances.

As long as the quantity of configuration is small, the translations can be performed manually.
For large numbers of files, we can try to use a tool in combination with human review and testing to ensure correctness.

These days we have a third option:  leveraging an LLM to translate our configurations.
This can be a viable option, but also comes with caveats:  it may be a security risk to divulge internal configurations if using an external provider.
Also LLMs are not impervious to making mistakes, and so their output needs to be reviewed and sometimes corrected.

### Explore `ingress2gateway`

The Kubernetes community provides a tool named [ingress2gateway](https://kubernetes.io/blog/2023/10/25/introducing-ingress2gateway/) designed specifically for this purpose.

Follow the instructions to install the tool's CLI on your machine.

The tool provides a number of options to control scope and source of configuration:  we can have it inspect a running cluster and look at Ingress objects applied directly to the cluster, or we can point it at an input file on disk and have it translate that.  We can also scope the search for Ingress resources to a namespace, or have it look at all namespaces.

Here are a number of alternative invocations of the tool to generate configuration:

Look for Ingress resources in the entire cluster:

```shell
ingress2gateway print --providers ingress-nginx --all-namespaces
```

Scope the search to the `httpbin` namespace:

```shell
ingress2gateway print --namespace httpbin --providers ingress-nginx
```

Or, for `bookinfo`:

```shell
ingress2gateway print --namespace bookinfo --providers ingress-nginx
```

Use an input file and translate that:

```shell
ingress2gateway print --input-file ./httpbin-https-ingress.yaml --namespace httpbin --providers ingress-nginx
```

Or, for `bookinfo`:

```shell
ingress2gateway print --input-file ./bookinfo-https-ingress.yaml --namespace bookinfo --providers ingress-nginx
```

## Review and vet the generated configurations

The biggest difference by far between Ingress and Gateway API resources is that a single Ingress configuration maps to a pair of resources:  Gateway and an HTTPRoute.

Let us study the output from `ingress2gateway`:

```shell
ingress2gateway print --providers ingress-nginx --all-namespaces
```

Like most tools, this one appears to perform direct translations:  for every Ingress resource, it produces a Gateway and an HTTPRoute.
The tool does not perform intelligent analysis of the cluster, and will not take into account that there are two distinct applications where a plausible configuration would be to provision a single, shared gateway.

```yaml title="generated-config.yaml" linenums="1"
--8<-- "generated-config.yaml"
```

The other glaring difference between what the tool produced compared to what a human would, is the repetition of the `backendRefs` section across multiple routing rules, failing to realize that the same configuration can be simplified to a single rule with multiple `matches` and a single `backendRef`.

Finally, the tool has no way of knowing (or does not provide a way to specify) what `gatewayClassName` to use for the translated resource, and so we much edit that value as well (to `kgateway`).

In summary, using `ingress2gateway` is useful and insightful, but produces only a starting point for review, and evaluation.
We must then make implementation decisions and, from the generated output, derive and craft the final Gateway API resource artifacts.

We opt to provision a single gateway, with two separate HTTPRoute resources, one per application.

## Provision the Gateway

Study the following Gateway resource configuration:

```yaml title="gateway.yaml" linenums="1"
--8<-- "gateway.yaml"
```

We make a number of design decisions:

- Decide to provision a single gateway, and place it in the `kgateway-system` namespace, controlled (and accessible only) by cluster administrators.
- Define a port 80 listener.  We will later define a routing rule to redirect all HTTP requests to HTTPS.
- Specify two listeners, one for each application (`httpbin` and `bookinfo`), matching on the corresponding hostname, each configured to terminate TLS, and each referencing its corresponding TLS certificate.

These decisions have implications:  the secrets we created previously exist alongside each Ingress resource, in their corresponding application namespaces.  We must now place a copy of these secrets in `kgateway-system` to make it accessible to the new Gateway.

```shell
kubectl create secret tls httpbin-cert -n kgateway-system \
  --cert=httpbin.crt --key=httpbin.key
```

And for `bookinfo`:

```shell
kubectl create secret tls bookinfo-cert -n kgateway-system \
  --cert=bookinfo.crt --key=bookinfo.key
```

Apply the Gateway resource:

```shell
kubectl apply -f gateway.yaml
```

List the deployments running in `kgateway-system`:

```shell
kubectl get deploy -n kgateway-system
```

Applying the Gateway resource triggered the provisioning of the Envoy proxy deployment `my-gateway`.

Also note the accompanying LoadBalancer-type service with external IP address:

```shell
kubectl get svc -n kgateway-system
```

## Configure routing

### `httpbin`

Study the below HTTPRoute for `httpbin`:

```yaml title="httpbin-route.yaml" linenums="1"
--8<-- "httpbin-route.yaml"
```

The above configuration is essentially what `ingress2gateway` produced, but with the following differences:

- Bind to the gateway `my-gateway`, provisioned in the `kgateway-system` namespace
- Explicitly attach the route to the `httpbin-https` listener
- Remove redundant `matches` and `hostnames` clauses.

Apply the route:

```shell
kubectl apply -f httpbin-route.yaml
```

### `bookinfo`

Study the below HTTPRoute for `bookinfo`:

```yaml title="bookinfo-route.yaml" linenums="1"
--8<-- "bookinfo-route.yaml"
```

Aside from binding to the `bookinfo-https` listener on the gateway in `kgateway-system`, the big difference above, compared to what the tool produced, is the elimination of the repeated or duplicate `backendRef` section:  we can specify a single routing rule with multiple path matches, all routing to the same, single `productpage` backend reference.

Apply the route:

```shell
kubectl apply -f bookinfo-route.yaml
```

### HTTP redirect to HTTPS

Study the below routing rule:

```yaml title="http-redirect.yaml" linenums="1"
--8<-- "http-redirect.yaml"
```

Note:

- We place this HTTPRoute in `kgateway-system`; it is not a concern of the application development teams.
- The routing rule binds to (applies to) the `http` listener only.
- The rule uses the [RequestRedirect](https://gateway-api.sigs.k8s.io/reference/spec/#httprequestredirectfilter) filter to redirect the request to the https scheme, otherwise preserving the original URL.

## Test the configuration

When we begin testing our setup, the first thing we learn is that the routes did not attach to the gateway:

```shell
kubectl get httproute -n httpbin httpbin-route -o yaml
```

Here is the output of the `status` section:

```yaml linenums="1" title="route status" hl_lines="4-9"
status:
  parents:
  - conditions:
    - lastTransitionTime: "2025-04-16T22:15:18Z"
      message: ""
      observedGeneration: 1
      reason: NotAllowedByListeners
      status: "False"
      type: Accepted
    - lastTransitionTime: "2025-04-16T22:15:18Z"
      message: ""
      observedGeneration: 1
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: kgateway.dev/kgateway
    parentRef:
      group: gateway.networking.k8s.io
      kind: Gateway
      name: my-gateway
      namespace: kgateway-system
      sectionName: httpbin-https
```

The condition _"Accepted=False"_ with the reason _"NotAllowedByListeners"_ stems from the fact that our gateway is not configured to allow routes from different namespaces to attach to it.

It stems from the decision we made to use a shared gateway and to place it in a separate namespace.
Application namespaces must be designated such that routes defined within them are permitted to attach.

Here is a revised Gateway resource with explicit `allowedRoutes` rules for each listener:

```yaml title="gateway-allows-routes.yaml" linenums="1" hl_lines="13-15 24-29 38-43"
--8<-- "gateway-allows-routes.yaml"
```

The convention is to label each namespace with `self-serve-ingress="true"` to allow both applications to define routes against the shared gateway:

```shell
kubectl label ns httpbin self-serve-ingress=true
```

And:

```shell
kubectl label ns bookinfo self-serve-ingress=true
```

Check the routes again:

```shell
kubectl get httproute -n httpbin httpbin-route -o yaml
```

This time the _Accepted_ condition should be _True_.

We can also check on the status of the gateway itself:  each listener should have one attached route:

```shell
kubectl get gtw -n kgateway-system my-gateway -o yaml
```

### Send test requests

Send in some test requests through the new gateway.

First, capture the new gateway IP address:

```shell
export GW_IP=$(kubectl get gtw -n kgateway-system my-gateway -ojsonpath='{.status.addresses[0].value}')
```

Call `httpbin`:

```shell
curl --insecure https://httpbin.example.com/headers --resolve httpbin.example.com:443:$GW_IP | jq
```

Verify that redirects work correctly by making a call over HTTP:

```shell
curl -v http://httpbin.example.com/headers --resolve httpbin.example.com:80:$GW_IP
```

Call `bookinfo`:

```shell
curl --insecure https://bookinfo.example.com/productpage --resolve bookinfo.example.com:443:$GW_IP | grep title
```

Verify that redirects work correctly for `bookinfo` as well:

```shell
curl -v http://bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP
```

Note the HTTP 301 (Moved permanently) response.

## Switching over

We have two gateways, and both are configured with equivalent rules, to route requests to `httpbin` and to `bookinfo`.
One gateway is controlled by the ingress-nginx controller and uses the nginx proxy, while the other is controlled by kgateway and uses the Envoy proxy.
Each has its own distinct public IP address.

Switching from ingress-nginx to kgateway is a matter of updating the DNS configuration for each host name:  alter the A record for `httpbin.example.com` to point to the new gateway IP address.

Monitor traffic through both gateways.
You should notice that all traffic to the hostname now goes through the new gateway, while the old gateway will cease to receive traffic.

Next, we can make the analogous DNS change for the `bookinfo` hostname, and observe that traffic begins to flow through the new gateway for `bookinfo` workloads too.

This points to the fact that it is possible to migrate from ingress-nginx to kgateway in a piecemeal fashion:
Teams can work at different pace and each migrate to the new gateway on their own schedule.

When all teams have completed their migration, we can finally decommission the old gateway.

## Decomission ingress-nginx

Decommissioning the old gateway involves undoing the original ingress setup:

1. Delete Ingress resources from each `bookinfo` and `httpbin` namespaces.
1. Delete the secrets holding the server certificates from these namespaces too.
1. Uninstall ingress-nginx.

## Summary

This migration isn't the end of the road.
kgateway provides capabilities that will take your environment well beyond what the original system could do:

1. Integrate kgateway with your Istio service mesh to gain encrypted communication with mutual TLS from the gateway to your mesh backend workloads.
1. With ambient mesh, you can use kgateway as your waypoint, unlocking kgateway's capabilities for workloads running internally.
1. Use kgateway as an egress gateway to control traffic exiting your environment.
1. Use kgateway's AI capabilities to secure your LLM applications as they make requests to external LLM providers.
