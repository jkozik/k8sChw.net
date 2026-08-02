# k8sChw.net

Setup camptonhillsweather.net in a Kubernetes cluster as deployment named chwnet; expose as
service chwnet; connect to the internet as camptonhillsweather.net through an HTTPRoute
pointing to the Envoy Gateway. Reuses the same NFS PV/PVC (`chwcom-persistent-storage`) as
[k8sChw.com](https://github.com/jkozik/k8sChw.com) — different rendering software, same
weather station data.

Source image [InstallCHW.net](https://github.com/jkozik/InstallCHW.net)

## Directory structure

```
k8sChw.net/
├── chwnet-deploy.yml      # Deployment (1 replica, jkozik/chw.net:v1)
├── chwnet-svc.yml         # NodePort service
├── chwnet-httproute.yaml  # HTTPRoute — camptonhillsweather.net via Envoy Gateway port 30458
├── README.md              # This file
└── old/
    └── chwnet-ingress.yml # Retired nginx Ingress (kept for reference)
```

## Prerequisites

- Envoy Gateway running with `weather-gateway` on NodePort 30458
- PV/PVC `chwcom-persistent-storage` already bound (created by k8sChw.com deploy)
- `nfs-common` installed on all cluster nodes

Verify PVC exists and is bound before applying this repo:
```bash
kubectl get pv,pvc | grep chwcom
# Expected: chwcom-persistent-storage   Bound
```

## Deploy

No PV/PVC step — this app reuses `chwcom-persistent-storage` from k8sChw.com.

```bash
cd ~/projects/k8sChw.net

# 1. Application
kubectl apply -f chwnet-svc.yml
kubectl apply -f chwnet-deploy.yml

# 2. Routing
kubectl apply -f chwnet-httproute.yaml
```

## Verify

```bash
kubectl get deployment,service,pod,httproute -l app=chwnet

# Expected:
# deployment.apps/chwnet   1/1
# service/chwnet           NodePort  80:<nodeport>/TCP
# pod/chwnet-<hash>        1/1 Running
# httproute/chwnet-route   camptonhillsweather.net, www.camptonhillsweather.net
```

Test via NodePort directly:
```bash
curl http://<node-ip>:<nodeport>/ | head -5
```

Test via Envoy Gateway (matches production path):
```bash
curl -H "Host: camptonhillsweather.net" http://<node-ip>:30458/ | head -5
# Should return Campton Hills weather HTML
```

## Cloudflare tunnel

Point the `camptonhillsweather.net` and `www.camptonhillsweather.net` public hostnames in
the Cloudflare Zero Trust tunnel to:
```
http://<node-ip>:30458
```

## NFS share (reference)

The NFS export is on 192.168.100.153 (dell3) and is configured via the `chwcom-persistent-storage`
PV in the k8sChw.com repo:
```
/home/nfs/weather-stations/chwcom/public_html  192.168.100.0/24(ro,sync,no_root_squash)
```

Mounted read-only at `/var/www/html/mount` inside the container.

## Build image / push to Docker Hub

```bash
docker login
docker tag jkozik/chw.net jkozik/chw.net:v1
docker push jkozik/chw.net:v1
```

## Ingress → HTTPRoute migration

Ingress (nginx) has been deprecated on this cluster. Traffic is managed via the Kubernetes
Gateway API implemented by Envoy Gateway. The old ingress yaml is preserved in `old/` for
reference only — do not apply it on the new cluster.
