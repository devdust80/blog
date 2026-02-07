+++
date = "2026-02-07T13:51:00Z"
title = "HAProxy Migration Challenges: Navigating the Transition from Ingress-NGINX to HAProxy Ingress Controller"
authors = [ "devdust80" ]
tags = [ "cloud", "migration", "ingress", "OSS", "kubernetes" ]
categories = [ "cloud", "migration", "ingress", "OSS", "kubernetes" ]
slug = "haproxy-migrating-to"
draft = false
+++

The ingress-nginx controller is approaching end-of-life[^1], leaving us with limited time to find a suitable replacement. We need a solution that preserves our existing feature set whilst managing the inevitable changes to configuration and architectural concepts. Several options exist in this space, but as part of this practice I wanted to share my experience with migrating from ingress-nginx to HAProxy Ingress Controller, and the challenges I faced during this process. One of the main challenges was related to mTLS and client certificate authentication, specifically in the scenario where we wanted to allow for mTLS without enforcing strict SSL verification, and also passing the client certificate to the backend services which is a common use case in many applications but is not natively supported in official HAProxy Ingress Controller[^2].

Official HAProxy[^2] is a powerful load balancer and reverse proxy controller that enables Kubernetes workloads to be exposed and accessible to external clients. It is natively allows for enabling mTLS but it lacks the option to allow failures if the verification didn't succeeded due to the CA was not present in the local store, or as pointed earlier the verification option of `optional_no_ca` does not exist here. This means that if we want to enable mTLS with HAProxy, we need to ensure that all client certificates are properly verified against a trusted CA and the certs must be valid and the chain must be present in the local store, otherwise the connection will be rejected. This can be a significant challenge for applications that rely on client certificate authentication but do not have a strict requirement for SSL verification, as it may lead to disruptions in service if the client certificates are not properly managed or if there are issues with the CA trust chain.

## The background: how ingress-nginx handles mTLS and client certificate authentication

As ingress-nginx was widely adopted and used across the industry, so many things have been added to it and by default there are so many annotations that we can add that allows to support multiple configs per each entry, amongst them are:

- `nginx.ingress.kubernetes.io/auth-tls-secret`
- `nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream`
- But the most important one `nginx.ingress.kubernetes.io/auth-tls-verify-client: optional_no_ca`[^3]

with value of <mark>optional_no_ca</mark>. Some part of it HAProxy can do natively out-of-the-box, like enabling mTLS but regarding the verification it is very strict. Ingress-nginx allows configuration per ingress resource with customised annotations present then the controller will generate appropriate nginx configuration for that specific ingress resource, this allows for a lot of flexibility and granularity in terms of how we can configure mTLS and client certificate authentication on a per-service basis. HAProxy, on the other hand, does not have the same level of granularity and flexibility when it comes to mTLS configuration, and it may require additional workarounds or custom configurations to achieve similar functionality.

## HAProxy mTLS and client certificate authentication challenges

HAProxy helm chart supports enabling mTLS but it lacks the option to allow failures if the verification didn't succeeded due to the CA was not present in the local store, or in situations where there is no need to enforce strict verification by sending the client the list of trusted CAs, If that is set then the client would check the list of CAs being presented by the controller and check them against its own CAs and if that was not present then it would not send back anything which is a problematic in this situation as we want to suppress the error and still allow the connection to proceed. But before progressing into how this can be achieved, lets first understand how HAProxy generated needed configuration for mTLS.

### That would be `bind` directive

In HAProxy, the magic happens at the `bind` directive. When using official helm chart[^4], some of the flags would be set by default through helm values, they are usually get populated based on the supported annotations[^5] that are supported by HAProxy ingress controller. Firstly to enable mTLS feature we can set:

- `client-ca`
- `client-crt-optional`

This is the minimum configuration needed to allow for mTLS for both clients that present a valid certificate and those that do not. The **gotcha** here is that if they do present a certificate, it should be a valid one and they should be signed by the same CA that is specified in the former flag. So the values for these flags would be as follows:

```yaml
# values.yaml
controller:
  config:
    client-ca: "namespace/secret-name"
    client-crt-optional: "true"
```

When setting these flags, HAProxy will generate appropriate configuration for the `bind` directive in the config file, which will look something like this:

```bash
# /etc/haproxy/haproxy.cfg
bind :443 ssl crt ... ca-file /etc/haproxy/certs/ca/ca.crt verify optional
```

## What about passing the client certificate to the backend services?

Now that we have enabled mTLS and allowed for clients that do not present a valid certificate, we now need to consider how to also allow for the clients that does not present a valid certificate so still HAProxy would terminate the TLS connection and pass the client certificate to the backend service. Looking at supported annotations[^5] we can see that there is no default annotations that allows us for such scenario, but HAProxy itself does support this feature but it requires adding custom configuration through the use of official HAProxy ingress controller helm chart[^4]. HAProxy engine deals with this scenario through the same `bind` directive and we need to add two more flags to the directive so that it would allow for invalid certificates and also to pass them to the backend services, these flags are:

- `no-ca-names`
- `crt-ignore-err`

<mark>no-ca-names</mark> will prevent HAProxy from sending the list of trusted CAs to the client, which will allow clients that do not have a valid certificate to still connect to the controller. <mark>crt-ignore-err</mark> will allow HAProxy to ignore any errors related to invalid certificates and still pass them to the backend services. With these flags added, the `bind` directive would look something like this:

```bash
bind :443 ssl crt ... ca-file /etc/haproxy/certs/ca/ca.crt verify optional no-ca-names crt-ignore-err 20,21
```

### What are the numbers 20 and 21 in the `crt-ignore-err` flag?

With having the `crt-ignore-err` flag set to 20,21, HAProxy will ignore errors related to invalid certificates and still pass them to the backend services. The numbers 20 and 21 correspond[^6] to specific error codes that HAProxy will ignore when processing client certificates. Error code 20 corresponds to "certificate expired" and error code 21 corresponds to "certificate not yet valid". By ignoring these errors, HAProxy will allow clients with expired or not yet valid certificates to still connect to the controller and have their certificates passed to the backend services for further processing. This can be useful in scenarios where you want to allow for mTLS but do not want to enforce strict certificate validation, such as in development or testing environments.

## Chart

The overview of how the packets flow in this scenario would be as follows:

{{<mermaid>}}
sequenceDiagram
participant Client
participant HAProxy
participant Backend
Client->>HAProxy: Send Request with or without client certificate
HAProxy->>HAProxy: verification
HAProxy->>HAProxy: Verification result (pass/fail)
HAProxy->>Backend: Pass packet to backend (if verified)
Backend->>Client: Response

{{</mermaid>}}

---

#### Footnotes

[^1]: Ingress NGINX Retirement: [What You Need to Know](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/).

[^2]: [Official HAProxy](https://haproxy-ingress.github.io/).

[^3]: Client certificate authentication [nginx-config](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#client-certificate-authentication).

[^4]: [HAProxy ingress-controller Helm Chart](https://github.com/haproxytech/helm-charts/tree/main/kubernetes-ingress)

[^5]: [HAProxy ingress-controller annotations](https://github.com/haproxytech/kubernetes-ingress/blob/master/documentation/annotations.md)

[^6]: [SSL error codes](https://github.com/openssl/openssl/blob/master/include/openssl/x509_vfy.h.in)
