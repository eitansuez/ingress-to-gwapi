# Rate limiting

In this exercise, we demonstrate the translation of an original ingress-nginx rate limiting configuration to AgentGateway.

Review the following Ingress configuration, and note on line 9 the annotation that specifies a local rate limit of 3 requests per minute:

```yaml linenums="1" title="httpbin-ratelimit-https-ingress.yaml" hl_lines="9"
--8<-- "httpbin-ratelimit-https-ingress.yaml"
```

Apply the configuration:

```shell
kubectl apply -f httpbin-ratelimit-https-ingress.yaml
```

## Run the migration tool

```shell
ingress2gateway print --providers ingress-nginx \
  --emitter agentgateway --namespace httpbin > generated-ratelimited.yaml
```

Study the generated resources:

```yaml title="generated-ratelimited.yaml" linenums="1" hl_lines="76-92"
--8<-- "generated-ratelimited.yaml"
```

The new bits are the last listed resource:  The AgentgatewayPolicy:

- It binds to the HTTPRoute resource so that the policy applies to the requests to `httpbin`.
- It configures agentgateway for local rate limiting: 3 requests per minute.


## Apply the generated resources

```shell
kubectl apply -f generated-ratelimited.yaml
```

Check the status of the applied policy and make sure it was _Accepted_:

```shell
kubectl get agentgatewaypolicies -n httpbin httpbin-ingress -o yaml
```

## Test rate limiting

Capture the gateway IP address:

```shell
export GW_IP=$(kubectl get gtw -n httpbin nginx -ojsonpath='{.status.addresses[0].value}')
```

Call `httpbin` four times:

```shell
for i in {1..4}; do curl --head -s --insecure https://httpbin.example.com/get --resolve httpbin.example.com:443:$GW_IP; done
```

Verify that fourth request was rate-limited (HTTP response code 429).

## Summary

In this scenario, we let the migration tool interpret the rate limiting configuration and produce valid agentgateway resources that capture that configuration, in addition to the other generated Gateway API resources.
