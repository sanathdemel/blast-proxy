Blast Beacon Proxy for Charon (Holesky)

A minimal nginx proxy that lets Obol Charon use Blast’s Holesky Beacon API without putting secrets or Blast path prefixes into Charon.

Why a proxy?
Blast requires a project prefix in the URL path and an x-api-key on every call (REST + SSE). The proxy injects both so Charon can just call:
http://blast-proxy.obol.svc.cluster.local:8080

Full deployment + troubleshooting: see RUNBOOK.md.

What’s in this repo

README.md

RUNBOOK.md — Full step-by-step guide

configmap.yaml — nginx template (${BLAST_PROJECT_ID}, ${BLAST_API_KEY}); SSE-friendly

deployment.yaml — nginx Deployment; renders template; /healthz probes

service.yaml — ClusterIP Service on :8080

secret-example.yaml — Example (placeholders only). Do NOT commit real secrets.

Not committed: any real Blast credentials.

How it works (at a glance)

Charon (obol-dv)
|
| http://blast-proxy.obol.svc.cluster.local:8080

v
[blast-proxy Service] -> nginx
- adds x-api-key
- rewrites path to /<PROJECT_ID>/<original>
- stable SSE (HTTP/1.1, no buffering)
|
v
Blast Holesky Beacon API (HTTPS)

60-second smoke test (safe to run)
NS=obol

# Proxy health:
kubectl -n "$NS" run test --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  sh -lc 'curl -sS -m 5 http://blast-proxy.obol.svc.cluster.local:8080/healthz; echo'

# Basic Beacon API through proxy:
kubectl -n "$NS" run test --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  sh -lc 'b=http://blast-proxy.obol.svc.cluster.local:8080;
          echo "== version ==";  curl -sS -m 10 $b/eth/v1/node/version; echo;
          echo "== syncing ==";  curl -sS -m 10 $b/eth/v1/node/syncing; echo;'

# SSE check (prints a couple of head events):
kubectl -n "$NS" run sse --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  sh -lc 'curl -sS -m 30 -N -H "Accept: text/event-stream" \
          "http://blast-proxy.obol.svc.cluster.local:8080/eth/v1/events?topics=head" | sed -n "1,6p"'

Security notes

No secrets in Git. Real Blast creds live only in a Kubernetes Secret.

Least privilege. nginx runs without elevated privileges; /healthz is read-only.

Config drift resistant. Blast values injected via env vars, not hard-coded.

Where to go next

Deploy / operate / troubleshoot: RUNBOOK.md

Review the manifests: configmap.yaml, deployment.yaml, service.yaml

See how secrets fit in: secret-example.yaml (placeholders only)
