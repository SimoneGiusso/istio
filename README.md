# Istio Learning

## Table of Contents

- [Requirements](#requirements)
- [Pre-setup](#pre-setup)
- [Setup](#setup)
  - [Uninstall istio](#uninstall-istio)
  - [Enable automatic sidecar injection](#enable-automatic-sidecar-injection)
- [Observability](#observability)
  - [Tracing](#tracing)
  - [Kiali](#kiali)
- [Traffic Management](#traffic-management)
  - [Ingress](#ingress)
    - [How the YAML maps to the flow](#how-the-yaml-maps-to-the-flow)
  - [Resources to manage traffic in Istio](#resources-to-manage-traffic-in-istio)
    - [Weighted traffic](#weighted-traffic)
    - [Header based traffic](#header-based-traffic)
    - [URI Rewrite](#uri-rewrite)
    - [Traffic Mirroring](#traffic-mirroring)
  - [Resiliency & Failure Injection: timeout, retry & circuit breaking](#resiliency--failure-injection-timeout-retry--circuit-breaking)
    - [Timeout](#timeout)
    - [Retries](#retries)
    - [Circuit breaking with health checks](#circuit-breaking-with-health-checks)
    - [Fault injection](#fault-injection)
    - [Bring external service into the mesh](#bring-external-service-into-the-mesh)
- [Security](#security)
  - [mTLS](#mtls)
  - [Authentication & Authorization](#authentication--authorization)
    - [Authentication](#authentication)
    - [Authorization](#authorization)
- [Advance topics](#advance-topics)
  - [Extend the mesh](#extend-the-mesh)
  - [Improve mesh performances](#improve-mesh-performances)
  - [Onboarding VM in istio cluster](#onboarding-vm-in-istio-cluster)

## Requirements

- Docker
- kind > v0.31.0
- cloud-provider-kind

## Pre-setup

`kind create cluster --name istio-cluster --config kind-config.yaml --wait 5m`

Run `kubectl get nodes -o wide` to verify installation

Run cloud provider kind for LoadBalancer cloud-like feature `sudo /opt/homebrew/bin/cloud-provider-kind`

## Setup

1. Run download script (https://istio.io/latest/docs/releases/supported-releases/#support-status-of-istio-releases):

    `curl -L ht‌tps://istio.io/downloadIstio | ISTIO_VERSION=1.29.0 sh -`

2. Export path:

    ```sh
    export PATH=$PWD/istio-1.29.0/bin:$PATH # Only valid in this terminal session
    istioctl version
    ```
3. Install istio on cluster

     `istioctl install -f demo-profile.yaml` or `istioctl install --set profile=demo`

 All available profiles: `ls istio-1.29.0/manifests/profiles`:

| PROFILE NAME | DESCRIPTION |
|---|---|
| ambient | Used for ambient mode deployment model. Includes the istiod and ztunnel components. |
| default | Includes components (istiod and istio-ingressgateway) and default settings from the IstioOperator API. Recommended profile for production deployments and primary clusters (multi-mesh scenario). |
| demo | Designed for showcasing Istio; includes istiod, istio-ingressgateway, and istio-egressgateway. Enables high levels of tracing and access logging. |
| empty | Same as the default profile, but it does not include any components. Mostly used as a base profile to build custom configurations. |
| minimal | Same as the default profile, without the istio-ingressgateway. |
| openshift | Used for deploying on OpenShift. Similar to the default profile; enables the use of Istio CNI. |
| preview | Includes the same components as the default profile (istiod and istio-ingressgateway) and enables experimental preview features. |
| remote | Previously called "external"; used to configure a remote cluster managed by an external control plane. |

  [IstioOperator resource](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/) [demo-profile.yaml](/demo-profile.yaml) control all the aspects of istio installation and component configuration and can be further customized.

4. Check installation:

    `kubectl get pods -n istio-system` 

### Uninstall istio

`istioctl uninstall -n istio-system`

Or to remove all Istio namespace resources:

`kubectl delete namespace istio-system`

### Enable automatic sidecar injection

```sh
kubectl label namespace default istio-injection=enabled`
kubectl get namespace -L istio-injection

kubectl create deploy my-nginx --image=nginx
```

It should shows a pod containing 2 containers

## Observability

Deploy a sample application and a sleep service:

```sh
kubectl apply -f istio-1.26.0/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f istio-1.26.0/samples/sleep/sleep.yaml 
```

Make an HTTP call
```sh
PRODUCTPAGE_POD=$(kubectl get pod -l app=productpage -ojsonpath='{.items[0].metadata.name}')
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
kubectl exec -it "$SLEEP_POD" -- sh -c 'curl -s productpage:9080/productpage | head'
```

Prometheus scrape endpoint is:

```sh
kubectl exec $PRODUCTPAGE_POD -c istio-proxy curl localhost:15090/stats/prometheus`
```

Deploy Prometheus and Grafana in istio-system namespace already configured to scrape metrics from workloads every 15 seconds:

```sh
kubectl apply -f istio-1.26.0/samples/addons/prometheus.yaml
kubectl apply -f istio-1.26.0/samples/addons/grafana.yaml # it contains pre-made dashboards
```

You can open prometheus/grafana dashboard locally:

```sh
istioctl dashboard prometheus
istioctl dashboard grafana
```

In another terminal make some calls to show something in the dashboards:

```sh
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
while true; do kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage; sleep 0.3; done
```

### Tracing

The Envoy sidecars are configured to send their trace information to a distributed tracing collector. Deploy a collector:

```sh
kubectl apply -f istio-1.26.0/samples/addons/jaeger.yaml
```

Make some calls:

```sh
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
while true; do kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage; sleep 1; done
```

Enable tracint and open jaeger dashboard:

```sh
kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: jaeger
    randomSamplingPercentage: 100.00
EOF

istioctl dashboard jaeger
```

### Kiali

Open source graphical console designed for Istio which uses metrics stored in prometheus:

```sh
kubectl apply -f istio-1.26.0/samples/addons/kiali.yaml
istioctl dashboard kiali
```

![Kiali Graph](resources/kiali-graph.png)

## Traffic Management

### Ingress

Configure istio ingress:

```sh
kubectl apply -f traffic-management/trafficmanagement/ingress-gateway.yaml
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_IP
```

Create a sample app with a service:

```sh
kubectl apply -f traffic-management/trafficmanagement/hello-world.yaml
```

Create a virtual service:

```sh
kubectl apply -f traffic-management/trafficmanagement/virtual-service.yaml
kubectl get vs
```

```sh
curl -v http://$GATEWAY_IP/
```

In the response `server: istio-envoy` shows that the request went through the ingress Envoy proxy.

```sh
kubectl delete deploy hello-world
kubectl delete service hello-world
kubectl delete vs hello-world
kubectl delete gateway gateway
```

#### How the YAML maps to the flow

- `kubectl get svc -n istio-system istio-ingressgateway` gives the external IP used by `curl`.
- `Service istio-ingressgateway` is the Kubernetes entry point exposed as `LoadBalancer`.
- `Gateway.spec.selector.istio: ingressgateway` binds the `Gateway` resource to the ingress gateway pod (`kubectl get pods -n istio-system -l istio=ingressgateway`).
- `Gateway.spec.servers[0].port.number: 80` and `protocol: HTTP` mean the ingress Envoy listens for HTTP traffic on port `80`.
- `Gateway.spec.servers[0].hosts: ["*"]` means any host header is accepted.
- `VirtualService.spec.gateways: ["gateway"]` says: apply these routing rules only for traffic entering through that Gateway.
- `VirtualService.spec.hosts: ["*"]` means the VirtualService matches any host already accepted by the Gateway.
- `VirtualService.spec.http[0].route[0].destination.host` points to the Kubernetes Service `hello-world.default.svc.cluster.local`.
- `Service.spec.selector.app: hello-world` selects the application pod.
- `Service.spec.ports[0].port: 80` and `targetPort: 3000` forward traffic from Service port `80` to the container port `3000`.

![Traffic Graph](resources/traffic-management.svg)

### Resources to manage traffic in Istio

Apart from Gateway, istio offers other resources to manage traffic:

- [Virtual Service](https://istio.io/latest/docs/reference/config/networking/virtual-service/): configure routing rules for services within the istio mesh:
    - split the traffic based on weights (70% to A, 30% to B)
    - route traffic based on some matching rule (e.g. header)
    - redirect traffic
    - mirroring traffic (fire and forget)
- [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/): contains rules applied after routing decision, configuring how to reach the target service (load balancer, connection pool and TLS settings).
- [Service Entry](https://istio.io/latest/docs/reference/config/networking/service-entry/): make an external service or API as part of the mesh (will istio features)

#### Weighted traffic

Route traffic between different service versions.

```sh
kubectl apply -f traffic-management/trafficrouting-weightbased/ingress-gateway.yaml

kubectl apply -f traffic-management/trafficrouting-weightbased/web-frontend.yaml

kubectl apply -f traffic-management/trafficrouting-weightbased/customers-v1.yaml

kubectl apply -f traffic-management/trafficrouting-weightbased/web-frontend-vs.yaml
```

Create a DestinationRule based on a label:

```sh
kubectl apply -f traffic-management/trafficrouting-weightbased/customers-dr.yaml
```

```sh
kubectl apply -f traffic-management/trafficrouting-weightbased/customers-dr.yaml
kubectl apply -f traffic-management/trafficrouting-weightbased/customers-vs.yaml
kubectl apply -f traffic-management/trafficrouting-weightbased/customers-v2.yaml
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Opening `GATEWAY_IP` on the browser should return 50% responses from customers v2 and 50% responses from customers v2.

Clean up:

```sh
kubectl delete deploy web-frontend customers-{v1,v2}
kubectl delete svc customers web-frontend
kubectl delete vs customers web-frontend
kubectl delete dr customers
kubectl delete gw gateway
```

#### Header based traffic

```sh
kubectl apply -f traffic-management/trafficrouting-requestheaderbased/ingress-gateway.yaml
kubectl apply -f traffic-management/trafficrouting-requestheaderbased/web-frontend.yaml
kubectl apply -f traffic-management/trafficrouting-requestheaderbased/customers.yaml
```

Opening `GATEWAY_IP` on the browser you should only see the NAME column on the resulting page since all traffic is routed to `customers-v1` service. However `curl -H "user: debug" http://$GATEWAY_IP/ | grep -E 'CITY|NAME'` will hit the `customer-v2` service.

```sh
kubectl delete deploy web-frontend customers-{v1,v2}
kubectl delete svc customers web-frontend
kubectl delete vs customers web-frontend
kubectl delete dr customers
kubectl delete gateway gateway
```

#### URI Rewrite

When matching on specific URIs, the request URI sometimes must be rewritten before forwarding to the destination. Istio supports URI rewrites using the **`rewrite`** field in a VirtualService.

The demo deploys `customers-v1` (returns names only) and `customers-v2` (returns names + city). The v2 API was internally refactored so its root endpoint is `/`. External callers still use the versioned path `/api/customers`. Rather than updating every caller, the VirtualService matches on the `/api/customers` prefix, rewrites the URI to `/`, and routes the request to the v2 subset. Unmatched traffic falls through to v1.

> **Response content reveals which version served the request:**
> - v1 → `[{"name":"..."}]` — names only
> - v2 → `[{"city":"...","name":"..."}]` — names + city

```sh
kubectl apply -f traffic-management/trafficrouting-rewrite/customers.yaml
kubectl apply -f istio-1.29.0/samples/sleep/sleep.yaml
kubectl rollout status deployment/customers-v1 deployment/customers-v2 deployment/sleep --timeout=60s

SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')

# Without the VirtualService: requests are load-balanced across v1 and v2 at random
kubectl exec $SLEEP_POD -c sleep -- curl -s http://customers.default.svc.cluster.local/api/customers
# could be names-only (v1) or names+city (v2) — non-deterministic

# Apply the VirtualService: /api/customers → rewrite to / → always v2; default → v1
kubectl apply -f traffic-management/trafficrouting-rewrite/customers-vs-rewrite.yaml

sleep 3  # allow Envoy config to propagate

# /api/customers is rewritten to / before reaching v2 → always returns names+city
kubectl exec $SLEEP_POD -c sleep -- curl -s http://customers.default.svc.cluster.local/api/customers
# [{"city":"...","name":"..."},...]

# Default route (no prefix match) always reaches v1 → names only
kubectl exec $SLEEP_POD -c sleep -- curl -s http://customers.default.svc.cluster.local/
# [{"name":"..."},...]
```

```sh
kubectl delete -f traffic-management/trafficrouting-rewrite/customers.yaml
kubectl delete -f traffic-management/trafficrouting-rewrite/customers-vs-rewrite.yaml
kubectl delete -f istio-1.29.0/samples/sleep/sleep.yaml
```

#### Traffic Mirroring

Traffic mirroring duplicates each incoming request and sends a copy to a second destination. The response from the mirror is discarded — callers always receive the primary service's response and are completely unaffected. This makes it safe to test a new service version against live production traffic before promoting it.

The demo routes 100% of traffic to `customers-v1` while simultaneously mirroring 100% of requests to `customers-v2`. Since callers always get v1's response, you verify the mirroring is active by watching the v2 **app container logs** — they show every mirrored request even though no caller ever sent traffic directly to v2.

> **Response content distinguishes the two versions:**
> - v1 → `[{"name":"..."}]` — names only (what callers always see)
> - v2 → `[{"city":"...","name":"..."}]` — names + city (receives silent copies; callers never see this)

> **Note on the `-shadow` hostname suffix:** older Istio/Envoy versions appended `-shadow` to the Host header of mirrored requests (e.g. `customers.default.svc.cluster.local-shadow`) so they were distinguishable in the proxy access logs. Istio 1.18+ disables this by default (`disableShadowHostSuffixAppend: true`), so mirroring is best confirmed via the destination app container logs as shown below.

```sh
kubectl apply -f traffic-management/trafficrouting-mirroring/customers.yaml
kubectl apply -f istio-1.29.0/samples/sleep/sleep.yaml
kubectl rollout status deployment/customers-v1 deployment/customers-v2 deployment/sleep --timeout=60s

SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
V2_POD=$(kubectl get pod -l app=customers,version=v2 -ojsonpath='{.items[0].metadata.name}')

# Apply the VirtualService: all traffic → v1, mirror 100% → v2
kubectl apply -f traffic-management/trafficrouting-mirroring/customers-vs-mirror.yaml
sleep 3  # allow Envoy config to propagate

# Send a few requests — callers always get v1 responses (names only)
for i in $(seq 1 5); do kubectl exec $SLEEP_POD -c sleep -- curl -s http://customers.default.svc.cluster.local/; done
# [{"name":"..."},...] every time — v1 only

# Confirm v2 received the mirrored copies via its app container logs
kubectl logs $V2_POD -c svc | tail -5
# lines like: "127.0.0.6:XXXXX customers.default.svc.cluster.local GET /"
# v2 app received every request even though no caller targeted it directly
```

```sh
kubectl delete -f traffic-management/trafficrouting-mirroring/customers.yaml
kubectl delete -f traffic-management/trafficrouting-mirroring/customers-vs-mirror.yaml
kubectl delete -f istio-1.29.0/samples/sleep/sleep.yaml
```

### Resiliency & Failure Injection: timeout, retry & circuit breaking

These can be specified in the Virtual Service

#### Timeout

A `timeout` on a VirtualService route makes Envoy return `504 Gateway Timeout` if the upstream does not respond in time.

The demo injects a 5-second delay into 100% of calls to `customers`, while the VirtualService enforces a 2-second timeout — every request times out.

```sh
kubectl apply -f resiliency/resiliency-timeout/gateway.yaml
kubectl apply -f resiliency/resiliency-timeout/web-frontend.yaml
kubectl apply -f resiliency/resiliency-timeout/customers.yaml  # normal, no delay yet

export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$GATEWAY_IP/  # 200 OK

kubectl apply -f resiliency/resiliency-timeout/customers-delay.yaml  # 5s delay + 2s timeout
curl http://$GATEWAY_IP/  # 504 after ~2s
```

```sh
kubectl delete -f resiliency/resiliency-timeout/ --ignore-not-found
```

#### Retries

> **Note**: Fault injection aborts bypass retry logic. This demo uses a real "bad" pod (nothing listening on the expected port) to produce genuine connection-level 503s that ARE retried.

The demo deploys two pods: `customers-v1` (healthy) and `customers-v2-bad` (no app listening on port 3000). Without retries ~50% of calls fail; with retries Envoy retries until it hits the healthy pod.

```sh
kubectl apply -f resiliency/resiliency-retries/customers-with-faults.yaml  # v1 healthy + v2 bad, no retries
kubectl apply -f istio-1.29.0/samples/sleep/sleep.yaml
kubectl rollout status deployment/customers-v1 deployment/customers-v2-bad deployment/sleep --timeout=60s # Wait for the above pods to be ready

SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
for i in $(seq 1 10); do kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://customers.default.svc.cluster.local; done  # ~50% are 503

kubectl apply -f resiliency/resiliency-retries/customers-with-retries.yaml  # adds retries: attempts 3, retryOn: 5xx,connect-failure
for i in $(seq 1 10); do kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://customers.default.svc.cluster.local; done  # all 200 (retries route to different pod; if first hits bad pod, retry hits healthy v1)
```

```sh
kubectl delete -f resiliency/resiliency-retries/customers-with-faults.yaml
kubectl delete -f istio-1.29.0/samples/sleep/sleep.yaml
kubectl delete vs customers 2>/dev/null; true
```


#### Circuit breaking with health checks

A `DestinationRule` with `outlierDetection` ejects unhealthy hosts from the load balancing pool after consecutive errors. The ejected endpoint is periodically retried after `baseEjectionTime` to check if it has recovered.

The demo deploys `customers-v1` (healthy) and `customers-v2-bad` (busybox, no listener on port 3000). Without the DestinationRule ~50% of calls hit the bad pod and fail. With `outlierDetection.consecutive5xxErrors: 1` the bad pod is ejected after its first failure for 30 seconds; after 30s Envoy retries it, and if it still fails, it gets re-ejected for another 30 seconds.

```sh
kubectl apply -f resiliency/resiliency-circuit-breaker/customers.yaml  # v1 healthy + v2 bad
kubectl apply -f istio-1.29.0/samples/sleep/sleep.yaml
kubectl rollout status deployment/customers-v1 deployment/customers-v2-bad deployment/sleep --timeout=60s

SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
for i in $(seq 1 10); do kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://customers.default.svc.cluster.local; done # ~50% 503

kubectl apply -f resiliency/resiliency-circuit-breaker/customers-circuit-breaker.yaml  # DestinationRule: eject bad pod after 1 consecutive error

for i in $(seq 1 10); do kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://customers.default.svc.cluster.local; done # bad pod ejected after first hit, then all 200
```

```sh
kubectl delete -f resiliency/resiliency-circuit-breaker/customers.yaml
kubectl delete -f istio-1.29.0/samples/sleep/sleep.yaml
kubectl delete dr customers 2>/dev/null; true
kubectl delete vs customers 2>/dev/null; true
```

#### Fault injection

It's possible to simulate a slow network or abort the Http request. Fault injections don't trigger retry policies!

![Fault Injection](resources/fault-injection.png)

```sh
kubectl apply -f resiliency/delays-and-failure-injection/gateway.yaml
kubectl apply -f resiliency/delays-and-failure-injection/web-frontend.yaml
kubectl apply -f resiliency/delays-and-failure-injection/customers.yaml
kubectl apply -f resiliency/delays-and-failure-injection/customers-delay.yaml

export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$GATEWAY_IP/ # Some requests take 5s

kubectl apply -f resiliency/delays-and-failure-injection/customers-fault.yaml
curl http://$GATEWAY_IP/ # Some requests return 500
```

```sh
kubectl delete deploy web-frontend customers-v1
kubectl delete svc customers web-frontend
kubectl delete vs customers web-frontend
kubectl delete gateway gateway
```

#### Bring external service into the mesh

A `ServiceEntry` registers an external host in Istio's registry so traffic policies (timeout, retry) apply to it. Combined with `meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY`, only explicitly registered external services are reachable.

**Part 1 — apply traffic policies to an external service:**

A `ServiceEntry` for `httpbin.org` plus a `VirtualService` with a 3-second timeout. `httpbin.org/delay/10` would normally take 10s — the timeout cuts it short.

```sh
kubectl apply -f istio-1.29.0/samples/sleep/sleep.yaml
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')

# Works without ServiceEntry (default ALLOW_ANY policy lets all traffic out)
kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://httpbin.org/get

kubectl apply -f resiliency/external-service/httpbin-entry.yaml   # register httpbin.org in the mesh
kubectl apply -f resiliency/external-service/httpbin-timeout.yaml # 3s timeout on all calls to httpbin.org

# This endpoint delays 10s — Envoy cuts it off after 3s with 504
kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://httpbin.org/delay/10
```

**Part 2 — block all unlisted external services (REGISTRY_ONLY):**

```sh
# Clean up Part 1 resources before switching mode
kubectl delete serviceentry httpbin-external 2>/dev/null; true
kubectl delete vs httpbin-external 2>/dev/null; true

istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY -y

# All external calls now blocked
kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://httpbin.org/get  # 502

# Allow only httpbin.org
kubectl apply -f resiliency/external-service/httpbin-entry.yaml
kubectl exec $SLEEP_POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://httpbin.org/get  # 200
```

```sh
# Restore default outbound policy
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY -y
kubectl delete -f istio-1.29.0/samples/sleep/sleep.yaml
kubectl delete serviceentry httpbin-external 2>/dev/null; true
kubectl delete vs httpbin-external 2>/dev/null; true
```

## Security

Some definitions:

**Access Control**: can a *principal* perform an *action* on this *object*?
**Authentication**: validate that credentials are valid and authentic. I k8s the service account is the identity
**mTLS**: both client and service provide certificates to verify identity.

### mTLS

![mTLS Architecture](resources/mTLS.png)

*PeerAuthentication* (accepted traffic) supports:

| Mode | Description |
|---|---|
| UNSET | Setting is inherited from the parent (mesh or namespace). If not set, the effective behavior defaults to PERMISSIVE. |
| DISABLE | mTLS is disabled; connections are plaintext. |
| PERMISSIVE (default) | The workload accepts both plaintext and mTLS connections. Useful for gradual rollout or mixed clients. |
| STRICT | Connections must use mTLS; both client and server must present valid certificates. |

*DestinationRule* (send traffic) supports:

| TLS Mode | Description |
|---|---|
| DISABLE | Does not set up a TLS connection (plaintext). |
| SIMPLE | Standard TLS where the client verifies the server certificate (server-side TLS). |
| MUTUAL | mTLS using user-provided key/certificate for both client and server. |
| ISTIO_MUTUAL (default) | Istio-managed mTLS using automatic Istio-issued certificates (recommended for mesh-wide mTLS). |

Therefore if the destination rule is not specificed, by default all traffic inside the mesh is encrypted.

For inbound connections outside the mesh there is the `tls` field in the `Gateway` resource. Common `tls.mode` values:

| TLS Mode | Description |
|---|---|
| PASSTHROUGH | Do not terminate TLS at the gateway; route matching is performed using SNI and the connection is forwarded to the destination unchanged. |
| SIMPLE | Terminate TLS at the gateway with a standard TLS connection (server-side TLS). |
| MUTUAL | Perform mTLS between the gateway and the destination using provided key/certificate (requires `caCertificates` or `credentialName`). |
| AUTO_PASSTHROUGH | Like `PASSTHROUGH` but the SNI contains destination details (service, subset, port), so an associated `VirtualService` is not required to map SNI to a service. |
| ISTIO_MUTUAL | Use Istio-managed certificates for mTLS on the gateway (Istio issues and rotates the certificates). |

For outbound connection *DestinationRule* is applied.

```sh
kubectl apply -f security/mTLS/gateway.yaml

kubectl label namespace default istio-injection- # Disable automatic injection to simulate a service outside the mesh
kubectl apply -f security/mTLS/web-frontend.yaml

kubectl label namespace default istio-injection=enabled
kubectl get pods # customers should have two containers while the frontend just one
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$GATEWAY_IP/ # it works and won't be encrypted since the frontend send plain text

kubectl apply -f security/mTLS/vs-customers-gateway.yaml # Expose customer
curl -H "Host: customers.default.svc.cluster.local" http://$GATEWAY_IP # This will be encrypted because it goes from ingress to customer

kubectl apply -f security/mTLS/strict-mtls.yaml # From this point on only calls directly to customer will work
```

```sh
# Cleanup

kubectl delete deploy web-frontend customers-v1
kubectl delete svc customers web-frontend
kubectl delete vs customers web-frontend
kubectl delete gateway gateway
```

### Authentication & Authorization

The *RequestAuthentication* resource is used for end-user authentication. The authentication is done per request and verifies the credentials attached to the request in JSON Web Tokens (JWTs).

#### Authentication

The *RequestAuthentication* resource validates JWT tokens on incoming requests. It does **not** enforce access by itself — a missing token is still allowed through (validation only applies when a token is present). Pair it with an *AuthorizationPolicy* to enforce who can access what.

```sh
# Deploy httpbin and curl WITH Envoy sidecars (2 containers each: app + istio-proxy)
kubectl create namespace foo
kubectl label namespace foo istio-injection=enabled
kubectl apply -f <(istioctl kube-inject -f istio-1.29.0/samples/httpbin/httpbin.yaml) -n foo
kubectl apply -f <(istioctl kube-inject -f istio-1.29.0/samples/curl/curl.yaml) -n foo
kubectl rollout status deployment/httpbin deployment/curl -n foo --timeout=90s

# Verify they can communicate before any policy
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -sS -o /dev/null -w "%{http_code}\n"
# 200

kubectl apply -f security/authentication/jwt-auth-httpbin.yaml

# Invalid token → 401 (RequestAuthentication rejects it)
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer invalidToken" -w "%{http_code}\n"
# 401

# No token → 200 (RequestAuthentication only validates; no AuthorizationPolicy yet)
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
# 200

# Valid token → X-Jwt-Payload injected and original Authorization header preserved
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29/security/tools/jwt/samples/demo.jwt -s)
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -H "Authorization: Bearer $TOKEN"
# "Authorization": ["Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c..."]
#   ↑ forwardOriginalToken: true — the full original JWT (header.payload.signature, 3 dot-separated parts) is forwarded as-is
#   Part 1 (header):    echo "eyJhbGciOiJSUzI1NiIs..." | base64 --decode → {"alg":"RS256","kid":"DHFbpoIUqrY8t2zpA2qXfCmr5VO5ZEr4RzHU_-envvQ","typ":"JWT"}
#   Part 2 (payload):   echo "eyJleHAiOjQ2ODU5..." | base64 --decode    → {"exp":4685989700,"foo":"bar","iat":1532389700,"iss":"testing@secure.istio.io","sub":"testing@secure.istio.io"}
#   Part 3 (signature): raw cryptographic bytes — not human-readable JSON
#
# "X-Jwt-Payload": ["eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9"]
#   ↑ outputPayloadToHeader — only the payload/claims part (the middle segment of the JWT), base64url-encoded
#   Unlike Authorization, this contains NO signature and NO JWT header — just the claims JSON
# echo "eyJleHAiOjQ2ODU5ODk3MDAs..." | base64 --decode
# → {"exp":4685989700,"foo":"bar","iat":1532389700,"iss":"testing@secure.istio.io","sub":"testing@secure.istio.io"}
```

`security/authentication/jwt-policy.yaml` — `AuthorizationPolicy` that requires a valid JWT with `requestPrincipal = testing@secure.istio.io/testing@secure.istio.io` (issuer + subject). Without a matching token the request is denied with `403`.

```sh
kubectl apply -f security/authentication/jwt-policy.yaml

# No token → 403 (AuthorizationPolicy denies — no matching principal)
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
# 403

TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 | base64 --decode
# → {"exp":4685989700,"foo":"bar","iat":1532389700,"iss":"testing@secure.istio.io","sub":"testing@secure.istio.io"}
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
# 200 — principal matches
```

`security/authentication/jwt-policy-with-claim.yaml` – updates the `AuthorizationPolicy` (same name `require-jwt`) to also require a `groups: group1` claim. The demo.jwt token no longer satisfies this condition.

```sh
kubectl apply -f security/authentication/jwt-policy-with-claim.yaml

# Valid token but no groups claim → 403
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
# 403

TOKEN_GROUP=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29/security/tools/jwt/samples/groups-scope.jwt -s) && echo "$TOKEN_GROUP" | cut -d '.' -f2 | base64 --decode
# → {"exp":3537391104,"groups":["group1","group2"],"iat":1537391104,"iss":"testing@secure.istio.io","scope":["scope1","scope2"],"sub":"testing@secure.istio.io"}
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN_GROUP" -w "%{http_code}\n"
# 200 — principal matches and groups claim is satisfied
```

```sh
# Clean up
kubectl delete namespace foo
```

#### Authorization

Once authenticated a principal can be subjected to authorized only to perform a specific set of actions: 

- ALLOW
- DENY
- CUSTOM: let the decision to an exteranl authorization system
- AUDIT: only log request if the criterias matches

If there are more policies the get evaluated in the following order: CUSTOM, DENY & ALLOW. A good practice is having a policy to deny all request and create individual allows.

```sh
kubectl apply -f security/authorization/gateway.yaml
kubectl apply -f security/authorization/web-frontend.yaml
kubectl apply -f security/authorization/customers-v1.yaml
kubectl rollout status deployment/web-frontend deployment/customers-v1 --timeout=90s

export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')


curl http://$GATEWAY_IP/ # Works

kubectl apply -f security/authorization/deny-all.yaml
curl http://$GATEWAY_IP/ # Fail

kubectl apply -f security/authorization/allow-ingress-frontend.yaml
curl http://$GATEWAY_IP/ # Fail but this time the failure is not coming from the ingress -> frontend connection but frontend -> customer

# This will be denied because it doesn't go through ingress. Deny all is still applied...
kubectl run curl --image=curlimages/curl --restart=Never --rm -it -- curl http://web-frontend  # RBAC: access denied

kubectl apply -f security/authorization/allow-web-frontend-customers.yaml
curl http://$GATEWAY_IP/ # Works
```

```sh
# Clean up

kubectl delete sa customers-v1 web-frontend
kubectl delete deploy web-frontend customers-v1
kubectl delete svc customers web-frontend
kubectl delete vs customers web-frontend
kubectl delete gateway gateway
kubectl delete authorizationpolicy allow-ingress-frontend allow-web-frontend-customers deny-all
kubectl delete pod curl
```

It is possible to be more granular at endpoint level

```yaml
to:
  - operation:
      host: ["*.hello.com"]
      methods: ["DELETE"]
      notPaths: ["/admin*"]
when:
   - key: request.auth.claims[iss]
     values: ["ht‌tps://accounts.google.com"]
   - key: request.headers[User-Agent]
     notValues: ["curl/*"]
```   

## Advance topics

### Extend the mesh

It's possible via the wasm plugin to add custom functionality to envoy proxies, e.g. it's possible to write a script that add headers to each request.

### Improve mesh performances

By default istio inform each service of each other. If there is a change the work to get the mesh up to date is bigger compare to the case where istio knows which service communicate to which ones. This is worthy for very large clusters though.

```sh

kubectl apply -f istio-1.29.0/samples/bookinfo/platform/kube/bookinfo.yaml
istioctl proxy-config endpoints deploy/ratings-v1.default # shows the list of endpoints the sidecar is configured to but this service has not outbound connections

kubectl apply -f advanced/limit-service-awarness-sidecars.yaml
istioctl proxy-config endpoints deploy/ratings-v1.default # now it has not any reference to ratings service for example

# Clean up
kubectl delete -f istio-1.29.0/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete sidecar {details,productpage,ratings,reviews-v1,reviews-v2,reviews-v3}-sidecar
```

### Onboarding VM in istio cluster

Istio makes its istio-proxy sidecar available as a Debian (.deb) or CentOS (.rpm) package and can simply be installed on a Linux VM and configured as a systemd service.

![VM Mesh Extension](resources/VM-mesh-extension.png)

