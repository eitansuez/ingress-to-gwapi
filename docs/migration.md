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
k get deploy -n kgateway-system
```

We can also list GatewayClass resources, which are the Gateway API's equivalent to the IngressClass concept:

```shell
k get gatewayclass
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

### Ingress2gateway

The Kubernetes community provides a tool named [ingress2gateway](https://kubernetes.io/blog/2023/10/25/introducing-ingress2gateway/) designed specifically for this purpose.

Follow the instructions to install the tool's CLI on your machine.


Generate gw api artifacts from ingress api resources
Review generated artifacts

### Tooling for migration

Explore ingress2gateway.


## Vet the revised configurations

configure the gateway

## Provision a Gateway

TODO

## Switching over

configure kgateway for httpbin: apply generated artifacts

switching dns
progressive transition
switch dns for httpbin to new gw ip
configure ingress for bookinfo
switch dns for bookinfo to new gw ip

decommission ingress-nginx
no more ingress api

## Summary

this isn't the end of the road.
we haven't even discussed service mesh.
in addition to configuring ingress, the gateway api can be leveraged to configure routing for internal traffic.