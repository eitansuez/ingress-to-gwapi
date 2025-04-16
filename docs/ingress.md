# Configure ingress

To configure ingress for each application, apply [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) resources, one for each application, as show below.

## `httpbin`

Review the configuration, which matches on a ficticious host name `httpbin.example.com` and routes matching requests to the `httpbin` backend service.

```yaml linenums="1" title="httpbin-ingress.yaml"
--8<-- "httpbin-ingress.yaml"
```

Apply it:

```shell
k apply -f httpbin-ingress.yaml
```

### Test ingress

The ingress-nginx controller has an associated LoadBalancer type service and hence, bearing an external IP address:

```shell
k get svc -n ingress-nginx
```

Capture the external IP address to the environment variable `GW_IP`:

```shell
export GW_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

When making an HTTP request, we can use [curl's `--resolve` flag](https://everything.curl.dev/usingcurl/connections/name.html#provide-a-custom-ip-address-for-a-name) to simulate DNS resolution of the host name to the gateway IP address.

Test the following example call to [`httpbin`'s `/get`](https://httpbin.org/#/HTTP_Methods/get_get) endpoint:

```shell
curl http://httpbin.example.com/get --resolve httpbin.example.com:80:$GW_IP | jq
```

The response should present a json-formatted payload reflecting the properties of the request, including the url, headers, and HTTP method.

## `bookinfo`

The configuration for `bookinfo` is lengthier because it configures multiple specific paths and path prefixes, all routing to the backend service `productpage` listening on port 9080:

```yaml linenums="1" title="bookinfo-ingress.yaml"
--8<-- "bookinfo-ingress.yaml"
```

Apply the configuration:

```shell
k apply -f bookinfo-ingress.yaml
```

### Test ingress

We can test ingress to the `bookinfo` application in a fashion similar to what we did above:

```shell
curl http://bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP
```

The response payload for the `/productpage` endpoint is an HTML page.

To verify the page is indeed from the `bookinfo` application, `grep` the response for the page title:

```shell
curl -s http://bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP | grep title
```

## Summary

Take stock of this "initial state".

- We have a Kubernetes cluster
- The ingress-nginx controller is installed
- Two distinct applications are deployed: `httpbin` and `bookinfo`
- Ingress resources configure ingress to each application through a single gateway

Before proceeding with migration, we need to turn this scenario into a more realistic one, one where the gateway serves traffic over HTTPS, and not as plain-text HTTP traffic.