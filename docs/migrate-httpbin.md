# Migrate `httpbin`

Let us assume that we are members of the team managing the `httpbin` app or service.

We will scope the migration of just that application to agentgateway.
The idea is for each team to work independently, at its own pace, to migrate their applications.

## Run the tool

Run the migration tool:

```shell
ingress2gateway print --providers ingress-nginx \
  --emitter agentgateway --namespace httpbin > generated-httpbin.yaml
```

Study the generated resources:

```yaml title="generated-httpbin.yaml" linenums="1"
--8<-- "generated-httpbin.yaml"
```

### Analysis

The tool generated three resources:  The gateway and two routes.

The gateway has two listeners:  a port 80 listener, plus a port 443 listener for HTTPS requests, with the certificate reference `httpbin-cert` carried over from the original configuration.

The original Ingress resource configured TLS redirect through this annotation:

```yaml
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

The tool is aware of the annotation, and translated it into an HTTPRoute with a [RequestRedirect filter](https://gateway-api.sigs.k8s.io/reference/spec/#httprequestredirectfilter), applied to the HTTP listener.

The second HTTPRoute is attached to the port-443 HTTPS listener and bears the rule that routes the request to the `httpbin` backend.

## Apply the generated resources

```shell
kubectl apply -f generated-httpbin.yaml
```

## Verify the migration

List the deployments in the `httpbin` namespace:

```shell
kubectl get deploy -n httpbin
```

_Applying the Gateway resource triggered the provisioning of the AgentGateway proxy deployment `nginx`._

!!! note

    `nginx` is certainly an odd choice for a name for our agentgateway-provisioned gateway.

Also note the accompanying LoadBalancer-type service with external IP address:

```shell
kubectl get svc -n httpbin
```

Check the status of the gateway:

```shell
kubectl get gtw nginx -n httpbin -o yaml
```

Inspect the `status` section in the output and confirm that:

1. The gateway was successfully "Programmed" and "Accepted" (true).
2. The http listener has one attached route, the one with the RequestRedirect filter.
3. The https listener also has one attached route, responsible for routing https requests to the `httpbin` backend.

We can likewise inspect the `status` section for the two HTTPRoute resources, which we leave for you as an exercise.

## Test the configuration

Send some test requests through the new gateway.

First, capture the new gateway IP address:

```shell
export GW_IP=$(kubectl get gtw -n httpbin nginx -ojsonpath='{.status.addresses[0].value}')
```

Call `httpbin`:

```shell
curl -s --insecure https://httpbin.example.com/headers \
  --resolve httpbin.example.com:443:$GW_IP | jq
```

Verify that redirects work correctly by making a call over HTTP:

```shell
curl -v http://httpbin.example.com/headers --resolve httpbin.example.com:80:$GW_IP
```

You should see in the output the 301 "Moved Permanently" HTTP response code.

## Summary

Congratulations!  You have migrated `httpbin` over to the Kubernetes Gateway API with AgentGateway.
We can proceed in this exact same fashion to migrate `bookinfo`.
Once all applications are migrated we are free to decommission ingress-nginx.
At some point we can also consider evaluating the applications and their dedicated gateways, and identifying which might be better suited for a shared gateway configuration.