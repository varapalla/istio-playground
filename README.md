# Three-tier Istio circuit-breaker POC

A small Kubernetes POC intended for an iximiuz lab:
- Frontend: Nginx (2 replicas)
- Backend: Flask (2 replicas)
- Database: PostgreSQL (1 replica)
- Istio: sidecar injection, DestinationRule circuit breaker, retries/timeouts, fault injection

The Flask `/api` endpoint performs a real `SELECT 1` against PostgreSQL. `/error` returns HTTP 500 and `/slow` sleeps for 10 seconds.

## Prerequisites

- Kubernetes cluster and `kubectl`
- `istioctl` installed
- Cluster nodes can pull the container images
- Istio control plane installed (or install it below)

Check Istio:

```bash
istioctl version
kubectl get pods -n istio-system
kubectl get crd destinationrules.networking.istio.io
```

If only the `istioctl` binary is installed, install the control plane:

```bash
istioctl install --set profile=default -y
```

The `default` profile is used here to keep the lab footprint lower than the demo profile. Ensure the cluster has enough resources for Istio and the application.

## Deploy

Run from this directory:

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-database.yaml
kubectl apply -f 02-backend.yaml
kubectl apply -f 03-frontend.yaml

kubectl rollout status deployment/postgres -n three-tier
kubectl rollout status deployment/backend -n three-tier
kubectl rollout status deployment/frontend -n three-tier
```

Check that application Pods have sidecars:

```bash
kubectl get pods -n three-tier
```

Expected containers per Pod:
- frontend: `2/2`
- backend: `2/2`
- postgres: `2/2` if injected (the namespace injects sidecars into all Pods)

Apply traffic policies:

```bash
kubectl apply -f 04-destination-rule.yaml
kubectl apply -f 05-virtual-service.yaml
```

## Test the application

In terminal 1:

```bash
kubectl port-forward svc/frontend 8080:80 -n three-tier
```

In terminal 2:

```bash
curl -i http://localhost:8080/
curl -i http://localhost:8080/api
```

The `/api` response should confirm the backend reached PostgreSQL and returned `database_check: 1`.

## Test timeout and backend failure

The backend has:
- `/slow`: sleeps for 10 seconds
- `/error`: returns HTTP 500

The VirtualService sets a 5-second timeout and two retries for eligible failures. Try:

```bash
curl -i http://localhost:8080/slow
curl -i http://localhost:8080/error
```

## Test Envoy circuit-breaker counters

Start a temporary load generator:

```bash
kubectl run fortio -n three-tier \
  --image=fortio/fortio:latest \
  --restart=Never --command -- sleep 3600
```

Generate concurrent traffic:

```bash
kubectl exec fortio -n three-tier -- \
  fortio load -c 20 -qps 0 -n 500 \
  http://backend.three-tier.svc.cluster.local/api
```

Inspect Envoy stats on a frontend Pod:

```bash
kubectl exec deploy/frontend -n three-tier -c istio-proxy -- \
  pilot-agent request GET stats | grep -E \
  'upstream_rq_pending|upstream_cx_overflow|outlier_detection'
```

Circuit-breaker overflow is driven by connection-pool limits under concurrency. Outlier ejection requires consecutive upstream 5xx responses; use `/error` or the fault-injection policy to generate failures. Because the backend has two replicas and `maxEjectionPercent` is 100, both can be ejected if both meet the failure threshold.

## Test Istio fault injection

The fault-injection manifest intentionally replaces the normal VirtualService for the backend host. Apply it only for this test:

```bash
kubectl apply -f 06-fault-injection.yaml
curl -i http://localhost:8080/error
curl -i http://localhost:8080/slow
```

Restore the normal routing policy:

```bash
kubectl delete -f 06-fault-injection.yaml
kubectl apply -f 05-virtual-service.yaml
```

## Troubleshooting

```bash
kubectl get pods -n three-tier
kubectl describe pods -n three-tier
kubectl logs deployment/backend -n three-tier -c backend
kubectl logs deployment/frontend -n three-tier -c nginx
istioctl analyze -n three-tier
```

If image pulls or `pip install` are slow, wait for the backend Pod to become ready and inspect its logs.

## Cleanup

```bash
kubectl delete namespace three-tier
```

## POC limitations

- PostgreSQL uses `emptyDir`; data is lost when its Pod is recreated.
- Credentials are demo-only and must not be used in production.
- The backend installs Python dependencies at container startup to keep the POC self-contained. For production, build and publish an immutable application image.
- This is a sidecar-mode POC. Ambient mode requires a different enrollment and waypoint setup.
