---
title: DEVOPS-Camp 2024 -- kubernetes ingress vs. gateway-api
author: venc0r
theme:
  name: mytheme
---

Install k3s without ingress
===

# Installing a simple kubernetes cluster based on k3s
```bash
❯ export INSTALL_K3S_EXEC="\
        --disable=traefik \
        --tls-san=dvoc24.v3nc.org \
        --node-external-ip=213.95.48.135"

❯ curl -sfL https://get.k3s.io | sh -
```

# The kubeconfig to access the cluster is located at
`/etc/rancher/k3s/k3s.yaml`

# Check the cluster access
```bash
❯ kubectl get nodes

NAME     STATUS   ROLES                  AGE   VERSION
dvoc24   Ready    control-plane,master   19h   v1.30.5+k3s1
```

<!-- end_slide -->

Install argocd
===

```bash
❯ kubectl create ns argocd
namespace/argocd created
```

```bash
❯ helm dependency build argocd
Getting updates for unmanaged Helm repositories...
...Successfully got an update from the "https://argoproj.github.io/argo-helm" chart repository
Hang tight while we grab the latest from your chart repositories...
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading argo-cd from repo https://argoproj.github.io/argo-helm
Deleting outdated charts
```

```bash
❯ helm template argocd --namespace argocd --include-crds argocd | kubectl apply -n argocd -f -

serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
serviceaccount/argocd-notifications-controller created
serviceaccount/argocd-repo-server created
serviceaccount/argocd-server created
serviceaccount/argocd-dex-server created
secret/argocd-notifications-secret created
secret/argocd-secret created
configmap/argocd-cm created
configmap/argocd-cmd-params-cm created
configmap/argocd-gpg-keys-cm created
configmap/argocd-notifications-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
configmap/argocd-redis-health-configmap created
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-notifications-controller created
clusterrole.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-applicationset-controller created
role.rbac.authorization.k8s.io/argocd-notifications-controller created
role.rbac.authorization.k8s.io/argocd-repo-server created
role.rbac.authorization.k8s.io/argocd-server created
role.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
rolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
rolebinding.rbac.authorization.k8s.io/argocd-repo-server created
rolebinding.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
service/argocd-applicationset-controller created
service/argocd-repo-server created
service/argocd-server created
service/argocd-dex-server created
service/argocd-redis created
deployment.apps/argocd-applicationset-controller created
deployment.apps/argocd-notifications-controller created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-redis created
statefulset.apps/argocd-application-controller created
serviceaccount/argocd-redis-secret-init created
role.rbac.authorization.k8s.io/argocd-redis-secret-init created
rolebinding.rbac.authorization.k8s.io/argocd-redis-secret-init created
job.batch/argocd-redis-secret-init created
```

<!-- end_slide -->
Access argocd with the created initial admin password and local portforwarding
===

```bash
❯ argocd app sync argocd --core

TIMESTAMP                  GROUP              KIND                            NAMESPACE
2024-10-04T19:56:16+02:00                     ConfigMap                       argocd
2024-10-04T19:56:16+02:00                     ServiceAccount                  argocd
...
GROUP        KIND         NAMESPACE  NAME           STATUS  HEALTH  HOOK  MESSAGE
argoproj.io  Application  argocd     argocd         Synced                application.ar...
argoproj.io  Application  argocd     ingress-nginx  Synced                application.ar...
```

```bash
❯ kubectl get secrets argocd-initial-admin-secret -o json | jq '.data.password' -r | base64 -d
<redacted>
```

```bash
❯ kubectl port-forward svc/argocd-server 8443:443
Forwarding from 127.0.0.1:8443 -> 8080
Forwarding from [::1]:8443 -> 8080
```

<!-- end_slide -->
Sync the applications
===
# Cert manager for automatic tls certificate generation from Let's Encrypt.
```bash
❯ argocd app sync cert-manager --core
```

# Install the NGINX Ingress Controller
```bash
❯ argocd app sync ingress-nginx --core
❯ kubectl get svc -n ingress-nginx
NAME                     TYPE          CLUSTER-IP    EXTERNAL-IP    PORT(S)
ingress-nginx-controller LoadBalancer  10.43.187.48  213.95.48.135  80:31742/TCP,443:32358/TCP
```

# Ingress controller implementation for HAProxy LoadBalancer
```bash
❯ argocd app sync ingress-haproxy --core
❯ kubectl get svc -n ingress-haproxy
NAME             TYPE          CLUSTER-IP     EXTERNAL-IP    PORT(S)
haproxy-ingress  LoadBalancer  10.43.168.250  213.95.48.135  18000:30512/TCP,18443:32390/TCP
```

<!-- end_slide -->
Sync the applications
===
# Contour is an ingress controller for Kubernetes that works by deploying the Envoy proxy
```bash
❯ argocd app sync ingress-contour --core
❯ kubectl get svc -n ingress-contour
NAME                  TYPE         CLUSTER-IP    EXTERNAL-IP   PORT(S)
ingress-contour-envoy LoadBalancer 10.43.245.106 213.95.48.135 28000:32451/TCP,28443:30234/TCP
```


# Traefik Proxy as your Kubernetes Ingress Controller
```bash
❯ argocd app sync ingress-traefik --core
❯ kubectl get svc -n ingress-traefik
NAME             TYPE          CLUSTER-IP     EXTERNAL-IP    PORT(S)
ingress-traefik  LoadBalancer  10.43.221.155  213.95.48.135  38000:30269/TCP,38443:31270/TCP
```


<!-- end_slide -->
Check Loadbalancer Services
===
```bash
❯ kubectl get svc -A | grep Loadbalancer

NAME                        TYPE CLUSTER-IP     EXTERNAL-IP    PORT(S)
ing-nginx-controller        LB   10.43.187.48   213.95.48.135  80:31742/TCP,443:32358/TCP
ing-haproxy-haproxy-ingress LB   10.43.168.250  213.95.48.135  18000:30512/TCP,18443:32390/TCP
ing-contour-envoy           LB   10.43.245.106  213.95.48.135  28000:32451/TCP,28443:30234/TCP
ing-traefik                 LB   10.43.221.155  213.95.48.135  38000:30269/TCP,38443:31270/TCP
```

# Check the different values.yaml for each ingress-controller
```
ingress-contour/values.yaml
ingress-haproxy/values.yaml
ingress-nginx/values.yaml
ingress-traefik/values.yaml
```
<!-- end_slide -->

Install pacman
===

```bash
❯ argocd app sync pacman --core
```

# pacman is accessible though contour
```bash
❯ curl http://pacman.dvoc24.v3nc.org:28000
```

<!-- end_slide -->
Install pacman
===
# Check ingress
```bash
❯ kubectl get ingress -A

NAMESPACE   NAME             CLASS    HOSTS                    ADDRESS         PORTS     AGE
argocd      argocd-server    nginx    argocd.dvoc24.v3nc.org   213.95.48.135   80, 443   23h
pacman      pacman-ingress   <none>   pacman.dvoc24.v3nc.org   213.95.48.135   80, 443   23h
```

# Check ingressClasses
```bash
❯ kubectl get ingressclass

NAME              CONTROLLER                                                  PARAMETERS   AGE
contour           projectcontour.io/ingress-contour/ingress-contour-contour   <none>       23h
ingress-traefik   traefik.io/ingress-controller                               <none>       23h
nginx             k8s.io/ingress-nginx                                        <none>       23h
```

<!-- end_slide -->

Switch from ingress to gateway-api
===

