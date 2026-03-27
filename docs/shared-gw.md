# Shared Gateway Migration

Back in [Analysis & Design](analysis.md), we decided to perform a first-pass migration using a dedicated gateway per application.

In this section, we look at the alternative: of instead configuring a single shared gateway for both `httpbin` and `bookinfo` applications.

In this scenario, we eschew the use of the migration tool which, as we saw, is more suited for a one-to-one mapping between the original Ingress resources and Gateway+HttpRoute pairs.

Let us proceed then, to provision a single gateway, with two separate HTTPRoute resources, one per application.

## Provision the Gateway

Study the following Gateway resource configuration:

```yaml title="gateway.yaml" linenums="1"
--8<-- "gateway.yaml"
```

Above, we:

- Provision a single gateway, and place it in the `agentgateway-system` namespace, controlled (and accessible only) by cluster administrators.
- Define a port 80 listener.  We will later define a routing rule to redirect all HTTP requests to HTTPS.
- Specify two listeners, one for each application (`httpbin` and `bookinfo`), matching on the corresponding hostname, each configured to terminate TLS, and each referencing its corresponding TLS certificate.

These decisions have implications:  the secrets we created previously exist alongside each Ingress resource, in their corresponding application namespaces.  We must now place a copy of these secrets in `agentgateway-system` to make it accessible to the new Gateway.

```shell
kubectl create secret tls httpbin-cert -n agentgateway-system \
  --cert=httpbin.crt --key=httpbin.key
```

And for `bookinfo`:

```shell
kubectl create secret tls bookinfo-cert -n agentgateway-system \
  --cert=bookinfo.crt --key=bookinfo.key
```

Apply the Gateway resource:

```shell
kubectl apply -f gateway.yaml
```

List the deployments running in `agentgateway-system`:

```shell
kubectl get deploy -n agentgateway-system
```

_Applying the Gateway resource triggered the provisioning of the Envoy proxy deployment `my-gateway`._

Also note the accompanying LoadBalancer-type service with external IP address:

```shell
kubectl get svc -n agentgateway-system
```

## Configure routing

### `httpbin`

Study the below HTTPRoute for `httpbin`:

```yaml title="httpbin-route.yaml" linenums="1"
--8<-- "httpbin-route.yaml"
```

The above configuration is essentially what `ingress2gateway` produced, but with the following differences:

- Bind to the gateway `my-gateway`, provisioned in the `agentgateway-system` namespace
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

Aside from binding to the `bookinfo-https` listener on the gateway in `agentgateway-system`, the big difference above, compared to what the tool produced, is the elimination of the repeated or duplicate `backendRef` section:  we can specify a single routing rule with multiple path matches, all routing to the same, single `productpage` backend reference.

Apply the route:

```shell
kubectl apply -f bookinfo-route.yaml
```

### HTTP redirect to HTTPS

Study the below routing rule:

```yaml title="http-redirect.yaml" linenums="1"
--8<-- "http-redirect.yaml"
```

!!! note "Notes"

    - We place this HTTPRoute in `agentgateway-system`; it is not a concern of the application development teams.
    - The routing rule binds to (applies to) the `http` listener only.
    - The rule uses the [RequestRedirect](https://gateway-api.sigs.k8s.io/reference/spec/#httprequestredirectfilter) filter to redirect the request to the `https` scheme, otherwise preserving the original URL.

Apply the redirection rule:

```shell
kubectl apply -f http-redirect.yaml
```

## Test the configuration

When we begin testing our setup, the first thing we learn is that the routes did not attach to the gateway:

```shell
kubectl get httproute -n httpbin httpbin-route -o yaml
```

Here is the output of the `status` section:

```yaml linenums="1" title="Route status" hl_lines="4-9"
status:
  parents:
  - conditions:
    - lastTransitionTime: "2026-03-26T14:10:41Z"
      message: Parent listener not usable or not permitted
      observedGeneration: 1
      reason: NotAllowedByListeners
      status: "False"
      type: Accepted
    - lastTransitionTime: "2026-03-26T14:10:41Z"
      message: ""
      observedGeneration: 1
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: agentgateway.dev/agentgateway
    parentRef:
      group: gateway.networking.k8s.io
      kind: Gateway
      name: my-gateway
      namespace: agentgateway-system
      sectionName: httpbin-https
```

The condition _"Accepted=False"_ with the reason _"NotAllowedByListeners"_ is a permission error.
The gateway is not configured to allow the attachment of routes outside its namespace.
It stems from the decision to use a shared gateway and to place it in a separate namespace.
Application namespaces must be designated such that routes defined within them are permitted to attach.

Here is a revised Gateway resource with explicit `allowedRoutes` rules for each listener:

```yaml title="gateway-allows-routes.yaml" linenums="1" hl_lines="13-15 24-29 38-43"
--8<-- "gateway-allows-routes.yaml"
```

Above, we define a convention: a namespace labeled with `self-serve-ingress="true"` is allowed to define routes against the shared gateway.

Apply the label to each `httpbin` and `bookinfo` namespaces:

```shell
kubectl label ns httpbin self-serve-ingress=true
```

And:

```shell
kubectl label ns bookinfo self-serve-ingress=true
```

Finally, apply the revised gateway resource:

```shell
kubectl apply -f gateway-allows-routes.yaml
```

Check the route status once more:

```shell
kubectl get httproute -n httpbin httpbin-route -o yaml
```

Confirm that, this time, the _Accepted_ condition is _True_.

Also, check the status of the gateway itself:  each listener should have one attached route:

```shell
kubectl get gtw -n agentgateway-system my-gateway -o yaml
```

Here is a more targeted command that uses a jsonpath query to confirm that we have one attached route per listener:

```shell
kubectl get gtw -n agentgateway-system my-gateway -ojsonpath='{.status.listeners[*].attachedRoutes}'
```

### Send test requests

Send in some test requests through the new gateway.

First, capture the new gateway IP address:

```shell
export GW_IP=$(kubectl get gtw -n agentgateway-system my-gateway -ojsonpath='{.status.addresses[0].value}')
```

Call `httpbin`:

```shell
curl -s --insecure https://httpbin.example.com/headers --resolve httpbin.example.com:443:$GW_IP | jq
```

Verify that redirects work correctly by making a call over HTTP:

```shell
curl -v http://httpbin.example.com/headers --resolve httpbin.example.com:80:$GW_IP
```

Call `bookinfo`:

```shell
curl -s --insecure https://bookinfo.example.com/productpage --resolve bookinfo.example.com:443:$GW_IP | grep title
```

Verify that redirects work correctly for `bookinfo` as well:

```shell
curl -v http://bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP
```

Note the HTTP 301 (Moved permanently) response.

## Switching over

We have two gateways, and both are configured with equivalent rules, to route requests to `httpbin` and to `bookinfo`.
One gateway is controlled by the ingress-nginx controller and uses the nginx proxy, while the other is controlled by the agentgateway controller and uses the agentgateway proxy.
Each has its own distinct public IP address.

Switching from ingress-nginx to agentgateway is a matter of updating the DNS configuration for each host name:  alter the A record for `httpbin.example.com` to point to the new gateway IP address.

Monitor traffic through both gateways.
You should notice that all traffic to the hostname now goes through the new gateway, while the old gateway will cease to receive traffic.

Next, we can make the analogous DNS change for the `bookinfo` hostname, and observe that traffic begins to flow through the new gateway for `bookinfo` workloads too.

This points to the fact that it is possible to migrate from ingress-nginx to agentgateway in a piecemeal fashion:
teams can work at a different pace, and each migrate to the new gateway on their own schedule.

When all teams have completed their migration, we can finally decommission the old gateway.

## Decommission ingress-nginx

Decommissioning the old gateway involves undoing the original ingress setup:

1. Delete Ingress resources from each `bookinfo` and `httpbin` namespaces.

    ```shell
    kubectl delete ingress -n httpbin httpbin-ingress
    kubectl delete ingress -n bookinfo bookinfo-ingress
    ```

1. Delete the secrets holding the server certificates from these namespaces too.

    ```shell
    kubectl delete secret httpbin-cert -n httpbin
    kubectl delete secret bookinfo-cert -n bookinfo
    ```

1. Uninstall ingress-nginx.

    ```shell
    helm uninstall -n ingress-nginx ingress-nginx
    ```

    And:

    ```shell
    kubectl delete ns ingress-nginx
    ```

## Summary

Some observations about this exercise:

- We end up with a single gateway managed by cluster operators with application teams free to attach their application-specific routes, giving them self-service capabilities.  This aligns with the original persona-based design that the standards authors envisioned.

- Thanks to the Kubernetes Gateway API's ability to provision gateways on demand, we were able to build a separate, shared gateway alongside potentially existing dedicated gateways.  Assuming we prefer this model, we can now delete the original dedicated gateways from their respective namespaces.