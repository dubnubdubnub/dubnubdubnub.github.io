# Deploying the portfolio to the cluster

The site is served from the `tenant-isaac` namespace behind a namespace-scoped
Cloudflare Tunnel. GitHub builds the image; GitHub Pages stays as the
source-of-truth backup.

```
GitHub push → Actions builds image → GHCR → cluster pulls
tenant-isaac: [site (nginx)] ← [cloudflared] → Cloudflare → portfolio.<domain>
```

## One-time setup

### 1. Create the Cloudflare Tunnel (dashboard)
1. Cloudflare **Zero Trust → Networks → Tunnels → Create a tunnel → Cloudflared**.
2. Name it (e.g. `dubcluster`). Copy the **token** — the long string after
   `--token` in the install snippet.
3. **Public Hostname** tab → add a hostname:
   - Subdomain/domain: `portfolio.<yourdomain>` (or whatever you chose)
   - Type: `HTTP`, URL: `site:80`
   - Save. Cloudflare creates the DNS record for you.

### 2. Store the token as a Secret
```bash
export KUBECONFIG=~/.kube/dubcluster-isaac.yaml
kubectl create secret generic cloudflared-token \
  -n tenant-isaac --from-literal=token='<TUNNEL_TOKEN>'
```

### 3. Make the GHCR image pullable
After the first successful Actions run, open
`https://github.com/users/dubnubdubnub/packages/container/portfolio/settings`
and set visibility to **Public** (simplest for a public portfolio).

> Prefer private? Create a pull secret instead and add it to `site.yaml`:
> ```bash
> kubectl create secret docker-registry ghcr -n tenant-isaac \
>   --docker-server=ghcr.io --docker-username=dubnubdubnub \
>   --docker-password='<GH_PAT_with_read:packages>'
> ```
> then add `imagePullSecrets: [{name: ghcr}]` to the pod spec.

### 4. Apply the manifests
```bash
kubectl apply -f deploy/k8s/site.yaml
kubectl apply -f deploy/k8s/cloudflared.yaml
```

## Updating the site
Push to `main` → Actions rebuilds `:latest`. Then roll the deployment:
```bash
kubectl rollout restart deployment/site -n tenant-isaac
```

## Handy checks
```bash
kubectl get pods -n tenant-isaac
kubectl logs -n tenant-isaac deploy/cloudflared   # tunnel connection status
kubectl logs -n tenant-isaac deploy/site
```
