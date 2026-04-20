# Select an implementation

The main concerns relating to migrating from Ingress to the Gateway API are:

1. Identifying and selecting a controller that implements the new Kubernetes Gateway API.
1. Translating the Ingress configurations to Gateway API resources.
1. Vetting the revised configurations.
1. Switching over from the old gateway to the new one without downtime.

In addition, it is desirable to have the flexibility to transition one host name at a time.
In other words, in large enterprise environments, we need to accommodate teams transitioning to the new API at their own pace.

## The Gateway API

The [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) improves upon the Ingress model in a number of ways:

- Routing is a separate concern from Gateway configuration; each is configured through distinct resources.  Routes attach to Gateways.
- Personas are taken into account: platform administrators configure gateways while applications teams self-service routing rules for their apps.
- The creation of the Gateway resource also provisions the gateway, on-demand.

This last point affords us the flexibility to use either shared gateways or dedicated ones for specific applications.
Certain applications require a higher degree of isolation.  This can be imposed by regulatory or other requirements, where traffic flowing to the target application cannot pass through a shared component, perhaps for security reasons, or to avoid other types of issues that stem from "noisy neighbors."
On the other hand, for applications that do not require it, it's more cost effective and in some ways simpler to use a single shared gateway.

## Gateway API conformant controller

The Gateway API documentation [lists](https://gateway-api.sigs.k8s.io/implementations/#gateway-controller-implementation-status) implementations that conform to it.

For this migration we will use [agentgateway](https://agentgateway.dev/), an open-source project recently contributed to the Linux Foundation.

### Install agentgateway

Use the instructions from the [docs](https://agentgateway.dev/docs/kubernetes/latest/install/helm/) to install agentgateway on Kubernetes.

1. Apply the Gateway API CRDs:

    ```shell
    kubectl apply --server-side \
      -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
    ```

1. Install agentgateway's own CRDs with Helm:

    ```shell
    helm upgrade --install agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds \
      --namespace agentgateway-system --create-namespace \
      --version v1.0.0 
    ```

1. Install agentgateway:

    ```shell
    helm upgrade --install agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
      --namespace agentgateway-system \
      --version v1.0.0
    ```

Verify the installation by listing deployments in the newly-created `agentgateway-system` namespace:

```shell
kubectl get deploy -n agentgateway-system
```

List GatewayClass-type resources, which are the Gateway API's equivalent to the IngressClass concept:

```shell
kubectl get gatewayclass
```

The output will show a GatewayClass named `agentgateway`.

We now have a running control plane.

The next step is to provision a gateway, and for that we will use the Gateway resource.
