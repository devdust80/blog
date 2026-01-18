+++
date = "2026-01-17T21:55:22Z"
title = 'Migrating to HAProxy: Key Challenges'
authors = ["Devdust80"]
tags = ["cloud","migration","ingress","OSS"]
draft=false
categories = ["cloud","challenges","experiences"]
slug = "haproxy-challenges"
+++

The ingress-nginx controller is approaching end-of-life[^1], leaving us with limited time to find a suitable replacement. We need a solution that preserves our existing feature set whilst managing the inevitable changes to configuration and architectural concepts. Several options exist in this space, but I've been tasked with conducting a proof-of-concept using HAProxy.

HAProxy[^2] is a powerful load balancer and reverse proxy controller that enables Kubernetes workloads to be exposed and accessible to external clients. But in our use case we had a scenario where not only we wanted it not to enforce SSL verification but to also pass the certificate to the backend( or upstream which is called in ingress-nginx ).

## How does nginx do it?

As ingress-nginx was widely adopted and used across the industry, so many things have been added to it and by default there are so many annotations that we can add that allows to support multiple configs per each entry, amongst them are: `nginx.ingress.kubernetes.io/auth-tls-secret` and `nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream` but the most important one `nginx.ingress.kubernetes.io/auth-tls-verify-client:` with value of <mark>optional_no_ca</mark>[^3]. Some part of it HAProxy can do natively out-of-the-box, like enabling mTLS but regarding the verification it is very strict that I'll be describing the behaviour in the next section.

## Here we go with HAProxy

HAProxy helm chart supports enabling mTLS but it lacks the option to allow failures if the verification didn't succeeded due to the CA was not present in the local store, or as pointed earlier the verification option of `optiona_no_ca` does not exist here. Before jumping into the solution I'd like to describe how does the HAProxy ingress-controller would generate default config.

### HAProxy configs

configs.

### Solution

solution.

#### Footnotes

[^1]: Ingress NGINX Retirement: [What You Need to Know](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)

[^2]: [Official HAProxy](https://haproxy-ingress.github.io/)

[^3]: Client certificate authentication [nginx-config](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#client-certificate-authentication)
