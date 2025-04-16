# Introduction

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

For the new controller, we will explore [kgateway](https://kgateway.dev/), an open-source project recently contributed to the CNCF by Solo.io, and which implements the Kubernetes Gateway API.

## What next?

Follow the instructions on the next page to begin setting up your environment.

Your first goal will be to reach an initial state where:

- You have a Kubernetes cluster hosting two distinct applications, with
- The ingress-nginx controller installed
- Ingress resources configuring the flow of ingress traffic to your APIs and applications

You will next configure ingress with HTTPS to simulate a setup that is more aligned with real-world environment.

From there, we explore migration to the Kubernetes Gateway API.