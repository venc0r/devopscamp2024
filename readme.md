---
title: Bar-Camp 2025 -- kubernetes ingress vs. gateway-api
author: venc0r (Jörg)
theme:
  name: mytheme
---

Install minikube with metallb
===

# Installing a simple kubernetes cluster based on k3s
```bash
minikube start -d kvm2 --cni=cilium --container-runtime=cri-o --cpu='4'
minikube addons enable metallb
minikube addons configure metallb

-- Enter Load Balancer Start IP: 192.168.39.100
-- Enter Load Balancer End IP: 192.168.39.200
```

# Check the cluster access
```bash
kubectl get nodes

NAME       STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP
minikube   Ready    control-plane   2d22h   v1.32.0   192.168.39.50   <none>
```

<!-- end_slide -->

Install argocd
===

# Download the helm chart
```bash
helm dependency build argocd
Saving 1 charts
Downloading argo-cd from repo https://argoproj.github.io/argo-helm
Deleting outdated charts
...
```
# Create the Namespace
```bash
kubectl create ns argocd
namespace/argocd created
```

# Apply the helm chart
```bash
helm template argocd --namespace argocd --include-crds argocd | kubectl apply -n argocd -f -

serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
...
```
# Install the applications, after the CRDs were created
```bash
kubectl apply -n argocd -f argocd/templates/app_of_apps.yaml
```

<!-- end_slide -->
Access argocd with the created initial admin password and local portforwarding
===

# To access argocd through the browser
```bash
kubectl config set-context --current --namespace argocd
kubectl port-forward  svc/argocd-server 8443:443 &!

Forwarding from 127.0.0.1:8443 -> 8080
Forwarding from [::1]:8443 -> 8080
```

```bash
kubectl get secrets argocd-initial-admin-secret -o json | jq '.data.password' -r | base64 -d | xclip -selection clipboard
```

# With the argocd cli the `--core` flag uses the k8s-api
```bash
argocd app sync argocd --core --retry-limit 3

TIMESTAMP                  GROUP              KIND                            NAMESPACE
2024-10-04T19:56:16+02:00                     ConfigMap                       argocd
2024-10-04T19:56:16+02:00                     ServiceAccount                  argocd
...
GROUP        KIND         NAMESPACE  NAME           STATUS  HEALTH  HOOK  MESSAGE
argoproj.io  Application  argocd     argocd         Synced                application.ar...
argoproj.io  Application  argocd     ingress-nginx  Synced                application.ar...
```

# Cert manager for automatic tls certificate generation (from a self signed ca in the demo)
```bash
argocd app sync cert-manager --core --retry-limit 3
```

<!-- end_slide -->
Installing ingress I
===
# Install the NGINX Ingress Controller
```bash
argocd app sync ingress-nginx --core --retry-limit 3
kubectl get svc -n ingress-nginx

NAME                     TYPE          CLUSTER-IP    EXTERNAL-IP    PORT(S)
ingress-nginx-controller LoadBalancer  10.110.60.161 192.168.39.100 80:32541/TCP,443:32572/TCP
```

<!-- pause -->
Installing ingress II
===
# Contour is an ingress controller for Kubernetes that works by deploying the Envoy proxy
```bash
argocd app sync ingress-contour --core --retry-limit 3
kubectl get svc -n ingress-contour

NAME                  TYPE         CLUSTER-IP    EXTERNAL-IP    PORT(S)
ingress-contour-envoy LoadBalancer 10.104.17.65  192.168.39.101 80:30407/TCP,443:32495/TCP
```

<!-- end_slide -->
Installing ingress III
===
# Ingress controller implementation for HAProxy LoadBalancer
```bash
argocd app sync ingress-haproxy --core --retry-limit 3
kubectl get svc -n ingress-haproxy

NAME                               TYPE         CLUSTER-IP     EXTERNAL-IP    PORT(S)
ingress-haproxy-kubernetes-ingress LoadBalancer 10.103.220.236 192.168.39.102 80:30873/TCP,443:31417/TCP,443:31417/UDP,1024:31518/TCP
```

<!-- pause -->
Installing ingress IV
===
# Traefik Proxy as your Kubernetes Ingress Controller
```bash
argocd app sync ingress-traefik --core
kubectl get svc -n ingress-traefik

NAME             TYPE          CLUSTER-IP     EXTERNAL-IP    PORT(S)
ingress-traefik  LoadBalancer  10.98.186.233  192.168.39.103 80:30607/TCP,443:32543/TCP
```


<!-- end_slide -->
Access through ingress
===

# Check Loadbalancer Services
```bash
kubectl get svc -A | grep LoadBalancer

NAMESPACE         NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)
ingress-nginx     ingress-nginx-controller             LoadBalancer   10.110.60.161    192.168.39.100   80:32541/TCP,443:32572/TCP                                16h
ingress-contour   ingress-contour-envoy                LoadBalancer   10.104.17.65     192.168.39.101   80:30407/TCP,443:32495/TCP                                15h
ingress-haproxy   ingress-haproxy-kubernetes-ingress   LoadBalancer   10.103.220.236   192.168.39.102   80:30873/TCP,443:31417/TCP,443:31417/UDP,1024:31518/TCP   15h
ingress-traefik   ingress-traefik                      LoadBalancer   10.98.186.233    192.168.39.103   80:30607/TCP,443:32543/TCP                                14h
```

# sync pacman argocd application
```bash
argocd app sync pacman --core --retry-limit 3
```

# enable ingress in the argocd app to enable access without port-forwarding
```yaml
# apps/argocd-app.yaml
...
values: |
  argo-cd:
    server:
      ingress:
        enabled: false # <- change this value to true
```

<!-- end_slide -->

Access through ingress
===
# Check ingress
```bash
kubectl get ingress -A

NAMESPACE   NAME             CLASS            HOSTS         ADDRESS         PORTS     AGE
argocd      argocd-server    ingress-traefik   argocd.demo   192.168.39.103   80, 443   3m16s
pacman      pacman-ingress   ingress-traefik   pacman.demo   192.168.39.103   80, 443   21m
```

<!-- pause -->

# Check ingressClasses
```bash

kubectl get ingressclass -o custom-columns=\
NAME:metadata.name,\
CONTROLLER:.spec.controller,\
DEFAULT:".metadata.annotations.ingressclass\.kubernetes\.io/is-default-class"

NAME              CONTROLLER                                                  DEFAULT
contour           projectcontour.io/ingress-contour/ingress-contour-contour   true
haproxy           haproxy.org/ingress-controller/haproxy                      <none>
ingress-traefik   traefik.io/ingress-controller                               true
nginx             k8s.io/ingress-nginx                                        <none>
```

<!-- end_slide -->
Access through ingress
===
# pacman is accessible
```bash
curl https://pacman.demo -v -H 'Host: pacman.demo' --resolve pacman.demo:443:192.168.39.103 -k
```

# argocd ingress is not working
```bash
curl https://argocd.demo -v -H 'Host: argocd.demo' --resolve argocd.demo:443:192.168.39.103 -k
```

# check the docs how to do the ingress
```
https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#kubernetesingress-nginx
```
<!-- end_slide -->


fixing argocd ingress
===
# set new values
```yaml
argo-cd:
  server:
    certificate:
      enabled: true               # <- enabled the backend cert
      domain: argocd.demo
      issuer:
        group: cert-manager.io
        kind: ClusterIssuer
        name: selfsigned-issuer
    ingress:
      enabled: true
      ingressClassName: nginx     # <- select the ingressClass explicit
      hostname: argocd.demo
      annotations:
        nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" # <- talk https to the backend
        cert-manager.io/cluster-issuer: selfsigned-issuer
      extraTls:
        - secretName: argocd-cert
          hosts:
            - argocd.demo
```
<!-- end_slide -->
fixing argocd ingress
===

# check change of the ingressClass
```bash
kubectl get ingress -n argocd

NAME            CLASS   HOSTS         ADDRESS          PORTS     AGE
argocd-server   nginx   argocd.demo   192.168.39.100   80, 443   22m
```

# test connectivity
```bash
curl https://argocd.demo -v -H 'Host: argocd.demo' --resolve argocd.demo:443:192.168.39.100 -k
```

<!-- end_slide -->

Switch from ingress to gateway-api
===

# Compare Ingress with Gateway-API
![image:width:100%](excalidraw.png)
[excalidraw](https://excalidraw.com/#room=773766818138c18b8058,bGocVBhLblqf9xTp6v1zXQ)
<!-- end_slide -->

Switch from ingress to gateway-api
===

```bash
kubectl apply -f gateway-api/standard-install.yaml

customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
```

- GatewayClass    => Implementation to be used with a Gateway
- Gateway         => Creates a LoadBalancer service
- GRPCRoute       => Routing rule for gRPC traffic
- HTTPRoute       => Routing rule for http(s) traffic, terminating tls at the listener
- ReferenceGrant  => Granting the use of resources from different namespaces

<!-- pause -->
```bash
kubectl apply -f gateway-api/experimental-install.yaml

customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io configured
...
customresourcedefinition.apiextensions.k8s.io/backendlbpolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/backendtlspolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/tcproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/tlsroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/udproutes.gateway.networking.k8s.io created
```

- backendLBpolicies   => Configuraiton for session persistence
- backendTLSpolicies  => Backend TLS configuration (reencrypting the traffic)
- TCPRoute            => Routing rule for TCP traffic
- TLSRoute            => Routing rule for TLS traffic using the SNI header, without decrypting the traffic stream to the backend
- UDPRoute            => Routing rule for UDP traffic
<!-- end_slide -->

Enable Gateway Support for Cert-Manager and Install Envoy Gateway
===

# Enabled cert-manager gateway-api support
apps/cert-manager-app.yaml
```yaml
values: |
  cert-manager:
    installCRDs: true
    config:
      apiVersion: "controller.config.cert-manager.io/v1alpha1"
      kind: "ControllerConfiguration"
      enableGatewayAPI: true
```
```bash
argocd app sync cert-manager --core --retry-limit 3
```

# Deploy gateway-envoy application
```bash
argocd app sync gateway-envoy --core --retry-limit 3
```
<!-- end_slide -->

Setup Argocd and Pacman with httproute
===
# Edit argocd application to enabled http route for argocd and pacman
```bash
kubectl edit application pacman
kubectl edit application argocd
```
```yaml
values: |
  enableGateway: true # <- from false to true
```
```bash
argocd app sync pacman --core --retry-limit 3
argocd app sync argocd --core --retry-limit 3
```

<!-- pause -->

# Argocd needs a BackendTLSPolicy to allow https backend traffic
```bash
openssl s_client -showcerts -verify 5 -connect localhost:8443 < /dev/null | tail -n69 | head -n18 > ca.crt
kubectl create cm -n argocd ca --from-file ca.crt
kubectl get pods -n gateway-envoy -o name | xargs kubectl delete -n gateway-envoy
```
argocd/templates/backendtlspolicy.yaml

<!-- end_slide -->

Install Traefik (v3) with gateway-api
===

# cleanup before install, the gateway-traefik is an ingress controller too
```bash
argocd app delete ingress-traefik --core --wait
kubectl delete -f gateway-api/experimental-install.yaml
```

# Deploy traefik with gateway support and resync the apps
```bash
argocd app sync gateway-traefik --core --retry-limit 3
argocd app sync pacman --core --retry-limit 3
argocd app sync argocd --core --retry-limit 3
```

# Change httproutes to gateway-traefik parentRef
```bash
kubectl edit httproute -n pacman pacman.demo
kubectl edit httproute -n argocd argocd.demo
```

<!-- pause -->
## Gateway and GatewayClass through helm values
## LoadBalancer SVC is created from helm chart
## No support for BackendTLSPolicy yet (https://github.com/traefik/traefik/pull/11009 merged >6 month ago), but still not working
<!-- end_slide -->

Install haproxy with gateway-api
===

# Removing traefik and CRDs. Create haproxy application, haproxy requires an old CRD version.
```bash
argocd app delete gateway-traefik --core --wait
kubectl delete -f gateway-api/experimental-install.yaml
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.5.1/experimental-install.yaml

customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencepolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/tcproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/tlsroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/udproutes.gateway.networking.k8s.io created
...
job.batch/gateway-api-admission-patch created

argocd app sync gateway-haproxy --core --retry-limit 3
argocd app sync pacman --core --retry-limit 3
argocd app sync argocd --core --retry-limit 3
```
<!-- pause -->
## Old crds are used
## only TCP Route is supported
## LoadBalancer SVC from values

<!-- end_slide -->
Install nginx gateway fabric
===

# Removing haproxy-gateway and CRDs. Create gateway-traefik application, haproxy requires an old CRD version.
```bash
argocd app delete gateway-haproxy --core -y
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.5.1/experimental-install.yaml
```

# Apply nginx crd distribution
```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.2" | kubectl apply -f -

customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
```

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/experimental?ref=v1.6.2" | kubectl apply -f -

customresourcedefinition.apiextensions.k8s.io/backendtlspolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/tcproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/tlsroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/udproutes.gateway.networking.k8s.io created
```

<!-- end_slide -->
Install nginx gateway fabric
===

# Sync argocd to deploy gateway-nginx
```bash
argocd app sync gateway-nginx --core --retry-limit 3
argocd app sync pacman --core --retry-limit 3
argocd app sync argocd --core --retry-limit 3
```

# Change httproutes to gateway-nginx parentRef
kubectl edit httproute -n pacman pacman.demo
kubectl edit httproute -n argocd argocd.demo

## GatewayClass from helm chart
## LoadBalancer SVC from values

<!-- end_slide -->
Conclution
===

<!-- column_layout: [1, 1] -->

<!-- column: 0-->
# ingress
- Each deployment has its own values schema
- Limited features. The Ingress API only supports TLS termination and simple content-based request routing of HTTP traffic.
- Reliance on annotations for extensibility. The annotations approach to extensibility leads to limited portability as every implementation has its own supported extensions that may not translate to any other implementation.
- Implementation specific CRDs to extend features. (e.g. taefik)
- Insufficient permission model. The Ingress API is not well-suited for multi-team clusters with shared load-balancing infrastructure.

## List of implementations
https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

<!-- column: 1-->
# gateway-api
- Role-oriented - Gateway is composed of API resources which model organizational roles that use and configure Kubernetes service networking.
- Portable - This isn't an improvement but rather something that should stay the same. Just as Ingress is a universal specification with numerous implementations, Gateway API is designed to be a portable specification supported by many implementations.
- Expressive - Gateway API resources support core functionality for things like header-based matching, traffic weighting, and other capabilities that were only possible in Ingress through custom annotations.
- Extensible - Gateway API allows for custom resources to be linked at various layers of the API. This makes granular customization possible at the appropriate places within the API structure.

## List of implementations
https://gateway-api.sigs.k8s.io/implementations/

<!-- reset_layout -->
---
-
Its not perfect
## You have to test each implementation how feature complete it is
## Complex helm values are still a thing (in some cases)
## Only 2/4 of the implementation did work really well

-
No benefits when
## ingress features are enough for you
## your cluster setup is small (not operated by different teams)
