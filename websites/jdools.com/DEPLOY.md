# Landing Page Deployment (jdools.com)

## What's served where

Two different things serve `jdools.com`, and it's easy to assume they're the
same. They're not:

| Content | Served by | Live path |
|---------|-----------|-----------|
| `index.html` (landing page) | `landing-page` NGINX pod, from the `landing-page-html` ConfigMap volume | `https://jdools.com/` |
| `privacy-policy.html` | `landing-page` NGINX pod, from the `landing-page-html` ConfigMap volume | `https://jdools.com/privacy-policy.html` |
| `robots.txt` | Edge (Traefik), **not** the ConfigMap | `https://jdools.com/robots.txt` |
| `sitemap.xml` | *Not currently served — returns 404* | `https://jdools.com/sitemap.xml` |

Only the HTML files (index + privacy-policy) come from the ConfigMap. `robots.txt`
is served outside the pod, so editing `robots.txt` in this repo does **not**
change the live file until it's reapplied at the edge. `sitemap.xml` is
referenced in `robots.txt` today but is not actually deployed.

## How the ConfigMap files are served

```
┌──────────────────────────────────────────────────────────┐
│  ConfigMap `landing-page-html` (default namespace)        │
│  ├── index.html                                          │
│  └── privacy-policy.html                                 │
└───────────────────────┬──────────────────────────────────┘
                        │ mounted as a volume at /usr/share/nginx/html
                        ▼
┌──────────────────────────────────────────────────────────┐
│  `landing-page` deployment → NGINX pod (nginx:alpine)     │
│  - Static files read from the ConfigMap volume           │
│  - ConfigMap volume changes sync to the pod asynchronously │
│  - A new pod is required for a newly added file to appear │
└──────────────────────────────┬─────────────────────────────┘
                               │ ClusterIP service (port 80)
                               ▼
┌──────────────────────────────────────────────────────────┐
│  Traefik Ingress → jdools.com (edge)                      │
└──────────────────────────────────────────────────────────┘
```

### Why a rollout restart is required

The ConfigMap is mounted as a pod volume. The kubelet syncs that volume
asynchronously, and a newly added key (e.g. `privacy-policy.html`) is **not
guaranteed** to appear on a running pod — the pod kept serving `index.html`
only, returning `404` for the new file, until a restart. The `nginx:alpine`
image has no `reload` mechanism wired to volume changes, so the reliable fix
is to roll a new pod.

Never assume the volume is live after `kubectl apply`. Always `kubectl rollout
restart deployment/landing-page` (which is the documented step below) and then
verify the file actually serves before trusting it.

## Deploy / update the HTML files

Any HTML change — including adding a new page like `privacy-policy.html` —
follows the same three-step sequence. The deploy is to the live cluster, so do
this after the code is committed and green.

```bash
# 1. Rebuild the ConfigMap from the dist/ source files.
#    --from-file keys become the served paths: index.html and privacy-policy.html.
kubectl create configmap landing-page-html \
  --from-file=index.html=websites/jdools.com/dist/index.html \
  --from-file=privacy-policy.html=websites/jdools.com/dist/privacy-policy.html \
  --dry-run=client -o yaml | kubectl apply -f -

# 2. Force a new pod so the ConfigMap volume picks up the change.
kubectl rollout restart deployment/landing-page

# 3. Verify the file is actually served — before declaring it done.
kubectl rollout status deployment/landing-page
kubectl exec -l app=landing-page -- sh -c 'curl -s -o /dev/null -w "%{http_code}\n" http://localhost/privacy-policy.html'
```

Steps 1–2 alone are not enough — the rollout in step 2 is what makes the new
file appear, which is the step most easily skipped. Step 3 confirms it, because
a `404` means the running pod still doesn't have the file (usually the volume
hasn't synced yet; re-run the exec once more after the rollout finishes).

### Editing an existing file

Same commands, but the ConfigMap rebuild will only include files you list on the
command line. `--from-file` does **not** carry over keys from the existing
ConfigMap — every served HTML file must be listed explicitly, otherwise the
deploy drops the others. Keep `index.html` and `privacy-policy.html` on every
deploy.

## Deploying a brand-new HTML page

Adding a page (e.g. `privacy-policy.html`) is the same flow, just with a new
`--from-file` entry. Nothing else changes:

1. Add the page under `websites/jdools.com/dist/`.
2. (Optional) Link to it from `index.html` and add it to `sitemap.xml`.
3. Deploy per the commands above, listing **every** served HTML file in
   `--from-file` (index + the new page + any others that exist).
4. Verify both the new page and `index.html` serve `200` (step 3 in the deploy
   sequence covers this).

## robots.txt

`robots.txt` is served at the edge, not from the ConfigMap, so editing
`robots.txt` in this repo does not propagate with the ConfigMap deploy. It
currently references `https://jdools.com/sitemap.xml`, which is not deployed yet.
Deploy this separately at the ingress/edge and verify `robots.txt` returns
`200` after the change.

## sitemap.xml

Referenced by `robots.txt` but **not currently served** (`404`). Decide where it
should live — either serve it from the `landing-page` pod by adding it to the
ConfigMap `--from-file` deploy (it'll then follow the same rollout-restart flow
above), or serve it at the edge next to `robots.txt` — and deploy accordingly.
