# HTTPS

Real-world scenarios for ingress rarely serve traffic over HTTP.

In this exercise we retrofit our setup to use HTTPS.

This involves:

1. Generating TLS certificates for each host name
1. Storing the certificate in a Kubernetes Secret
1. Reconfiguring the Ingress resources for TLS

## Generate certificates

As this exploration is mostly exercise, we will use self-signed certificates.

Create a local subdirectory where the certificate files will reside:

```shell
mkdir certs && cd certs
```

Below we use the [`step` CLI](https://smallstep.com/docs/step-cli/) to generate the self-signed certificates for each host name:

```shell
step certificate create httpbin.example.com httpbin.crt httpbin.key \
  --profile self-signed --subtle --no-password --insecure
```

And for `bookinfo`:

```shell
step certificate create bookinfo.example.com bookinfo.crt bookinfo.key \
  --profile self-signed --subtle --no-password --insecure
```

## Store the certificates as secrets

```shell
kubectl create secret tls httpbin-cert -n httpbin \
  --cert=httpbin.crt --key=httpbin.key
```

And for `bookinfo`:

```shell
kubectl create secret tls bookinfo-cert -n bookinfo \
  --cert=bookinfo.crt --key=bookinfo.key
```

## Reconfigure Ingress

### `httpbin`

Below, note the addition of a `tls` section that references the secret you just created in association with the hostname `httpbin.example.com`:

```yaml linenums="1" title="httpbin-https-ingress.yaml" hl_lines="9-12"
--8<-- "httpbin-https-ingress.yaml"
```

Apply the Ingress resource:

```shell
kubectl apply -f httpbin-https-ingress.yaml
```

### `bookinfo`

Similarly, for `bookinfo` we add a `tls` configuration section referencing its certificate:

```yaml linenums="1" title="bookinfo-https-ingress.yaml" hl_lines="9-12"
--8<-- "bookinfo-https-ingress.yaml"
```

Apply the resource:

```shell
kubectl apply -f bookinfo-https-ingress.yaml
```

## Test it

Verify that you can now call `httpbin` over HTTPS:

```shell
curl --insecure https://httpbin.example.com/get --resolve httpbin.example.com:443:$GW_IP | jq
```

Note that we have to add the `--insecure` flag because we are using self-signed certificates.

Likewise for `bookinfo`:

```shell
curl -s --insecure https://bookinfo.example.com/productpage --resolve bookinfo.example.com:443:$GW_IP | grep title
```

Try to access `bookinfo` over HTTP:

```shell
curl -v http://bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP
```

This should produce a [`308 Permanent Redirect`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/308) response.

You can instruct `curl` to follow redirects, and the response will be served over HTTPS:

```shell
curl -s -L --insecure http://bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP --resolve bookinfo.example.com:443:$GW_IP | grep title
```

## Where we stand

We now have a setup that uses the nginx-ingress controller, and that uses Kubernetes Ingress resources to serve two applications over HTTPS.

It is time to turn a corner and start discussing how to migrate this setup to the Kubernetes Gateway API.