Blast Beacon Proxy + Charon (Holesky)

A lightweight nginx proxy that injects the Blast project prefix and x-api-key for all Beacon API calls (REST + SSE).
Charon (Deployment: obol-dv) talks to the proxy at: http://blast-proxy.obol.svc.cluster.local:8080.

This keeps Blast credentials out of Charon and centralises the auth logic.

What’s committed vs. what you run

Committed in this folder

configmap.yaml — nginx template with ${BLAST_PROJECT_ID} / ${BLAST_API_KEY}

deployment.yaml — nginx Deployment that renders the template; exposes /healthz

service.yaml — ClusterIP Service blast-proxy on port 8080

secret-example.yaml — documentation-only example with placeholders

RUNBOOK.md — this file

Not committed

Any Secret containing real Blast credentials

Prerequisites

Kubernetes cluster reachable by kubectl

Namespace: obol (change NS if different)

Blast credentials: projectId and apiKey

Optional convenience:

export NS=obol

1) Create/Update the Secret (do not commit)

Purpose: store Blast credentials for the proxy. Safe to re-run.

NS=${NS:-obol}
PROJ='<YOUR_BLAST_PROJECT_ID>'
KEY='<YOUR_X_API_KEY>'

kubectl -n "$NS" delete secret blast-api --ignore-not-found
kubectl -n "$NS" create secret generic blast-api \
  --from-literal=projectId="$PROJ" \
  --from-literal=apiKey="$KEY"

2) Apply/Update the proxy manifests (idempotent)

Purpose: render nginx from the template using env vars from the Secret; expose a stable internal service for Charon and testing.

NS=${NS:-obol}

kubectl -n "$NS" apply -f configmap.yaml
kubectl -n "$NS" apply -f deployment.yaml
kubectl -n "$NS" apply -f service.yaml
kubectl -n "$NS" rollout status deploy/blast-proxy


Internal URL: http://blast-proxy.obol.svc.cluster.local:8080

3) Point Charon at the proxy (run once per cluster)

Purpose: make Charon call the proxy (which adds prefix + x-api-key) for REST and SSE. This keeps credentials out of Charon.

NS=${NS:-obol}

kubectl -n "$NS" set env deploy/obol-dv --overwrite \
  CHARON_BEACON_NODE_ENDPOINTS="http://blast-proxy.obol.svc.cluster.local:8080" \
  CHARON_BEACON_NODE_HEADERS= \
  CHARON_BEACON_NODE_TIMEOUT=10s \
  CHARON_BEACON_NODE_SUBMIT_TIMEOUT=10s

kubectl -n "$NS" rollout restart deploy/obol-dv
kubectl -n "$NS" rollout status deploy/obol-dv --timeout=300s


Verify Charon parsed the proxy endpoint:

kubectl -n "$NS" logs deploy/obol-dv --since=5m \
| grep -Ei 'Parsed config|beacon-node-endpoints|beacon-node-headers' || true


Expected: beacon-node-endpoints shows the proxy URL; headers empty.

4) Smoke tests (through the proxy)

4A. REST endpoints

NS=${NS:-obol}
kubectl -n "$NS" run test --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  sh -lc 'b=http://blast-proxy.obol.svc.cluster.local:8080; \
          echo "== /node/version ==";  curl -sS -m 10 "$b/eth/v1/node/version"; echo; \
          echo "== /node/syncing ==";  curl -sS -m 10 "$b/eth/v1/node/syncing"; echo; \
          echo "== /headers/head ==";  curl -sS -m 10 "$b/eth/v1/beacon/headers/head" | head -c 160; echo'


Expect: HTTP 200 JSON; Lighthouse version visible; is_syncing:false on a healthy head.

If you see “pods … already exists”:

kubectl -n "$NS" delete pod test --ignore-not-found


4B. SSE stream

NS=${NS:-obol}
kubectl -n "$NS" run sse --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  sh -lc 'curl -sS -m 30 -N -H "Accept: text/event-stream" \
          "http://blast-proxy.obol.svc.cluster.local:8080/eth/v1/events?topics=head" | sed -n "1,6p"'


Expect lines like:

event:head
data:{"slot":"…","block":"…","state":"…"}


Note: If you pipe SSE to head, nginx may log “client prematurely closed connection”. That’s benign for smoke tests.

5) Health Checks

Proxy health

kubectl -n "$NS" get deploy,svc -l app=blast-proxy
kubectl -n "$NS" logs deploy/blast-proxy --since=5m


Charon liveness/readiness (inside the pod)

POD=$(kubectl -n "$NS" get pod -l app.kubernetes.io/instance=obol-dv -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NS" exec "$POD" -- sh -lc '
  for path in livez readyz; do
    code=$(wget -S -O /dev/null "http://127.0.0.1:3620/$path" 2>&1 | awk "/^  HTTP/{c=\$2} END{print c+0}")
    echo "$path: $code"
  done
'


Interpretation:

livez: 200 ⇒ process is healthy.

readyz may briefly be 500 during boot/sync. See Troubleshooting if it persists.

6) Troubleshooting

6.1 401/404 from Blast
Symptom: 401 on SSE, or 404 on REST like “NOT_FOUND: beacon block …”
Cause: missing /${projectId} prefix and/or missing x-api-key.
Fix: always call via proxy; ensure Secret exists; proxy is running.

6.2 SSE “client prematurely closed connection”
Symptom: message in nginx logs during tests.
Cause: you ended the stream (e.g., using head).
Fix: benign for tests; for long-lived consumers, don’t cut the stream.

6.3 Readiness probe stuck at 0/1
Symptom: Ready shows 0/1; readiness probe returns HTTP 500.
Quick mitigation (switch to /livez while investigating):

kubectl -n "$NS" patch deploy/obol-dv --type='json' -p='[
 {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/livez"},
 {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/timeoutSeconds","value":3},
 {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/periodSeconds","value":5},
 {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/failureThreshold","value":6}
]'
kubectl -n "$NS" rollout restart deploy/obol-dv


Revert to /readyz when stable if your policy prefers it.

6.4 CreateContainerConfigError — runAsNonRoot + named user
Symptom: container has runAsNonRoot and image has non-numeric user (charon)
Fix: set explicit UID/GID and harden:

kubectl -n "$NS" patch deploy/obol-dv --type='json' -p='[
 {"op":"add","path":"/spec/template/spec/containers/0/securityContext","value":{"runAsNonRoot":true}},
 {"op":"add","path":"/spec/template/spec/containers/0/securityContext/runAsUser","value":10001},
 {"op":"add","path":"/spec/template/spec/containers/0/securityContext/runAsGroup","value":10001},
 {"op":"add","path":"/spec/template/spec/containers/0/securityContext/allowPrivilegeEscalation","value":false}
]'
kubectl -n "$NS" patch deploy/obol-dv --type='json' -p='[
 {"op":"add","path":"/spec/template/spec/securityContext","value":{"fsGroup":10001}}
]'
kubectl -n "$NS" rollout restart deploy/obol-dv
kubectl -n "$NS" rollout status deploy/obol-dv --timeout=300s


6.5 Slow / flaky kubectl (TLS handshake, DNS timeouts)
Usually local network/VPN flakiness. Re-run; switch networks; no cluster changes required.

7) Security notes

Never commit real secrets. secret-example.yaml is documentation only.

Proxy centralises Blast credentials; Charon never handles them directly.

Hardened settings: allowPrivilegeEscalation=false; explicit non-root UID/GID.

8) Rollback

Switch Charon back to a direct provider (not recommended for Blast unless you handle prefix + headers yourself):

kubectl -n "$NS" set env deploy/obol-dv --overwrite \
  CHARON_BEACON_NODE_ENDPOINTS="<direct_url>" \
  CHARON_BEACON_NODE_HEADERS="x-api-key=<key>"

kubectl -n "$NS" rollout restart deploy/obol-dv


Disable proxy (optional):

kubectl -n "$NS" scale deploy/blast-proxy --replicas=0

Appendix — one-off validation
NS=${NS:-obol}
kubectl -n "$NS" get deploy blast-proxy obol-dv
kubectl -n "$NS" get pods -l app.kubernetes.io/instance=obol-dv -o wide
kubectl -n "$NS" run test --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  sh -lc 'b=http://blast-proxy.obol.svc.cluster.local:8080; \
          curl -sS -m 10 "$b/eth/v1/node/version"; \
          curl -sS -m 10 "$b/eth/v1/node/syncing"'

Quick commit commands
# Overwrite RUNBOOK.md with the above content, then:
git add RUNBOOK.md
git commit -m "Tidy RUNBOOK: remove stray 'Copy code' artefacts; fix fences"
git push

Optional self-check (should print “OK: none”)
grep -nE 'Copy code|pgsql|lua|perl|cpp|yaml' RUNBOOK.md || echo "OK: none"