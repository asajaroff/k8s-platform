
Table of Contents
=================

   * [Software tools](#software-tools)
   * [Create Cluster](#create-cluster)
      * [/etc/hosts](#etchosts)
   * [Install ArgoCD](#install-argocd)
      * [Login   Reset password](#login--reset-password)
      * [Bootstrap platform](#bootstrap-platform)
      * [Check UI](#check-ui)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)


# Software tools

* kubectl
* minikube
* argocd

# Create Cluster

Using minikube start a local k8s cluster

```
minikube start --driver=docker --kubernetes-version=v1.16.15
```

Enable Ingress add-on

```
minikube addons enable ingress
```

Patch Ingress controller to enable ssl passthru

```
kubectl patch deploy ingress-nginx-controller -n kube-system -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/8","value":"--enable-ssl-passthrough"}]' --type json
```

## /etc/hosts

Setup IP addresses for ingress end-points

```
echo "$(minikube ip) argocd.test" | sudo tee -a /etc/hosts
```

# Install ArgoCD

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Create an Ingress

```
kubectl apply -f - <<END
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.test
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
        path: /
  tls:
  - hosts:
    - argocd.test
    secretName: argocd-secret # do not change, this is provided by Argo CD
END
```

## Login + Reset password

Retrieve initial password

```
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

Login

```
$ argocd login argocd.test --insecure
Username: admin
Password: 
'admin' logged in successfully
Context 'argocd.test' updated
```

Change the password

```
argocd account update-password
```

## Bootstrap platform

```
argocd app create apps \
    --dest-namespace argocd \
    --dest-server https://kubernetes.default.svc \
    --repo https://github.com/myspotontheweb/k8s-platform.git \
    --path bootstrap/chart  

argocd app sync apps 
```

## Check UI

* https://argocd.test

