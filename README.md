# Immich on Kubernetes

This repo is prepared for Argo CD deployment of Immich on the existing Kubernetes cluster.

## Layout

- `k8s/base`: Kubernetes resources for Immich, Valkey, Postgres, storage and Service.
- `k8s/overlays/prod`: production overlay used by Argo CD.
- `argocd/immich-application.yaml`: optional Argo CD `Application`.
- `docker/*`: thin wrapper images over upstream Immich images.
- `.github/workflows/build-and-push.yml`: validates manifests and pushes images to GHCR.

## Cluster choices

- Media/library PVC uses the existing `nfs-csi` StorageClass.
- Machine-learning model cache uses `nfs-csi`.
- Postgres uses a local PV on `worker-1` at `/srv/immich/postgres` because Immich documents that network shares are not supported for database storage.
- The Immich service is a Cilium `LoadBalancer` using `10.10.2.55`.

## Deploy with Argo CD

Apply the Argo CD application from the cluster:

```bash
kubectl apply -f argocd/immich-application.yaml
```

Or create an Argo app that points to:

```text
repoURL: https://github.com/pascariucosmin93/imchi.git
path: k8s/overlays/prod
targetRevision: main
```

## HAProxy

After Argo syncs the app, HAProxy can forward to the Cilium LB IP:

```haproxy
frontend fe_immich_http
    bind 192.168.1.8:8084
    default_backend be_immich_http

backend be_immich_http
    server immich1 10.10.2.55:80 check
```

Reload with:

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
systemctl reload haproxy
```

## Notes

- The initial Immich admin user is created from the web UI on first login.
- Rotate the database password after first deploy if this repo is public.
- Current pinned Immich version: `v2.7.5`.
