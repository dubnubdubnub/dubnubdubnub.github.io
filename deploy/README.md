# Deploying the portfolio to the cluster

Live at **https://dubnubdubnub.com**, served from the `web` namespace on the
k3s cluster, behind a Cloudflare Tunnel. GitHub Actions builds the image;
GitHub Pages stays as the source-of-truth backup.

```
GitHub push → Actions builds image → GHCR → cluster pulls
web ns: [site (nginx)] ← [cloudflared] → Cloudflare tunnel → dubnubdubnub.com
```

## Current wiring (already provisioned)

- **Tunnel** `dubcluster` (`b4d8fd1d-481d-4efa-b76f-3b07015c31e5`), remotely
  managed; ingress routes `dubnubdubnub.com → http://site:80`.
- **Apex DNS** `dubnubdubnub.com` is a proxied CNAME to the tunnel.
- **cloudflared** runs in `web` (managed by the cluster sessions) using the
  connector token in Secret `cloudflared-token`. It must share a namespace
  with `site` so `http://site:80` resolves.
- **Image** `ghcr.io/dubnubdubnub/portfolio:latest`, built by
  `.github/workflows/docker-image.yml`. The GHCR package is public.

## Updating the site
Push to `main` → Actions rebuilds `:latest`, then:
```bash
export KUBECONFIG=~/.kube/dubcluster-isaac.yaml
kubectl rollout restart deployment/site -n web
```

## From scratch (if ever redeploying)
```bash
export KUBECONFIG=~/.kube/dubcluster-isaac.yaml
# token comes from the dubcluster tunnel (Zero Trust → Tunnels → Configure)
kubectl create secret generic cloudflared-token -n web --from-literal=token='<TUNNEL_TOKEN>'
kubectl apply -f deploy/k8s/site.yaml         # Deployment + Service in web
kubectl apply -f deploy/k8s/cloudflared.yaml  # only if not already running in web
```

## Handy checks
```bash
kubectl get pods -n web
kubectl logs -n web deploy/cloudflared   # tunnel connection / origin errors
kubectl logs -n web deploy/site
curl -sI https://dubnubdubnub.com
```

## Gotcha: cross-node pod networking (Flannel/ufw)
If the site 502s with cloudflared logging `i/o timeout` or `no such host` for
`site`, check whether the node running `site` allows Flannel's VXLAN overlay.
Nodes default-deny inbound ufw; the overlay needs **UDP 8472** open from the
cluster LAN. On the affected node:
```bash
sudo ufw allow from 10.1.1.0/24 to any port 8472 proto udp comment 'flannel vxlan'
```
(This is what was missing on `ux430` when it joined — node-exporter/kubelet
ports were opened but not the VXLAN port, so all cross-node pod traffic to it
was dropped.)
