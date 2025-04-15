# Migrating from ingress-nginx to Gateway API

## Introduction

In the early days of Kubernetes, teams started migrating their workloads from VMs to Kubernetes.
Teams set to work on containerizing and publishing their applications and services to container registries.
They had to develop Kubernetes deployment manifests, and figure out how to templatize and package their manifests, perhaps as a Helm chart.

Another piece of the puzzle had to do with configuring the routing of ingress traffic to their applications and APIs now residing inside a Kubernetes cluster.

One of the early "go-to" solutions for handling ingress traffic was the [ingress-nginx controller](https://kubernetes.github.io/ingress-nginx/).  As its name implies, this controller was designed to use the [nginx proxy](https://nginx.org/) as the runtime gateway that routed traffic to backend workloads.

As far as configuration was concerned, at the time, Kubernetes provided the [Ingress API](https://kubernetes.io/docs/concepts/services-networking/ingress/).

Fast forward to today, many lessons were learned which culminated in the evolved API for configuring ingress for Kubernetes:  the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/).

The Kubernetes Gateway API gives users a standard, and many alternative implementations to choose from, as attested by the long [list of implementations](https://gateway-api.sigs.k8s.io/implementations/).

Some controllers leverage nginx, others HA-Proxy or something else. Those proxies existed long before Kubernetes.
A relatively newer option and open-source CNCF project is [Envoy](https://www.envoyproxy.io/), which has been described as a  "cloud-native" proxy for the Kubernetes age.
Envoy is a "batteries-included" proxy that supports many protocols, and that can manage being hot-reconfigured when routing rules or other configuration details change.

Many vendors opt to use Envoy as their cloud-native gateway of choice, and so even within this microcosm, users have a number of vendors they can choose from.

Many teams today still use the venerable Ingress API together with the ingress-nginx controller.
Using the older implementation is becoming a sort of technical debt.

In this document, we will use a companion sample project that starts out using the ingress-nginx controller with the Ingress API.
We will walk through and explore migrating this project to the Gateway API.

For the new controller, We will explore [kgateway](https://kgateway.dev/), an open-source project recently contributed to the CNCF by Solo.io, and which implements the Kubernetes Gateway API.

## The example project

[tbd]

instructions:
- clone the repo
- provision a cluster
- deploy the workloads / applications
- apply the ingress configurations (https and manifests with tls termination)
- test it

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