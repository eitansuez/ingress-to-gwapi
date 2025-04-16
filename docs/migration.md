# Migration

Provision kgateway
 install ingress2gateway
 generate gw api artifacts from ingress api resources
 review generated artifacts
 configure kgateway for httpbin: apply generated artifacts

switch dns for httpbin to new gw ip
configure ingress for bookinfo
switch dns for bookinfo to new gw ip

decommission ingress-nginx

## Tooling for migration

Explore ingress2gateway.

## The Gateway API

Discuss how gw api is different from the ingress api

## Install kgateway

control plane
provision a gateway dynamically
discuss how now have more flexibility - can deploy multiple gateways, can use either a shared or dedicated model.

configure the gateway
switching dns
progressive transition
decommission ingress-nginx
no more ingress api

## Summary

this isn't the end of the road.
we haven't even discussed service mesh.
in addition to configuring ingress, the gateway api can be leveraged to configure routing for internal traffic.