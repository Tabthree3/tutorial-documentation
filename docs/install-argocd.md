# ArgoCD Installation
Argo CD is a declarative continuous delivery tool for Kubernetes applications. It uses the GitOps style to create and manage Kubernetes clusters. When any changes are made to the application configuration in Git, Argo CD will compare it with the configurations of the running application and notify users to bring the desired and live state into sync.

## Prerequsites 
- Virtual Machine running Ubuntu 22.04 or newer
### Update Package Repository and Upgrade Packages
``` console title="Run from shell prompt" linenums="1"
sudo apt update
sudo apt upgrade
```
## Create Kubernetes Cluster
``` bash title="Run from shell prompt" linenums="1"

#!/bin/bash
set -e

# Install K3s server
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - --disable traefik

# Configure kubeconfig
mkdir -p /home/ubuntu/.kube
sudo cp /etc/rancher/k3s/k3s.yaml /home/ubuntu/.kube/config
sudo chown ubuntu:ubuntu /home/ubuntu/.kube/config
sudo chmod 400 /home/ubuntu/.kube/config

# Export KUBECONFIG
export KUBECONFIG=/home/ubuntu/.kube/config
echo "export KUBECONFIG=/home/ubuntu/.kube/config" >> /home/ubuntu/.bashrc

# Optional: Start any additional services or configurations here


### Install ArgoCD
``` shell title="Run from shell prompt" linenums="1"
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Editting ConfigMap RBAC
``` console title="Run from shell prompt" linenums="1"

kubectl get configmap argocd-rbac-cm -n argocd -o yaml > argocd-rbac-cm.yaml
kubectl apply -f argocd-rbac-cm.yaml

kubectl rollout restart deployment argocd-server -n argocd

apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Existing roles and permissions
    p, role:org-admin, applications, *, */*, allow
    p, role:org-admin, clusters, get, *, allow
    p, role:org-admin, repositories, get, *, allow
    p, role:org-admin, repositories, create, *, allow
    p, role:org-admin, repositories, update, *, allow
    p, role:org-admin, repositories, delete, *, allow
    p, role:org-admin, projects, get, *, allow
    p, role:org-admin, projects, create, *, allow
    p, role:org-admin, projects, update, *, allow
    p, role:org-admin, projects, delete, *, allow
    p, role:org-admin, logs, get, *, allow
    p, role:org-admin, exec, create, */*, allow

    g, your-github-org:your-team, role:org-admin

    # New role for viewing running jobs
    p, role:view-jobs, applications, get, */*, allow
    p, role:view-jobs, applications, sync, */*, allow
    p, role:view-jobs, applications, action/*, */*, allow

    # Grant 'view-jobs' role to 'john_doe'
    g, john_doe, role:view-jobs
```

### Change Service to NodePort
Edit the service can change the service type from `ClusterIP` to `NodePort`
```  shell title="Run from shell prompt" linenums="1"
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}' 
```
### Fetch Password
``` shell title="Run from shell prompt" linenums="1"
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

# Optional (Enable TLS w/Ingress)
If you want to enable access from the internet or private network you can follow the instructions below to install and configure an ingress-controller with lets-encrypt.
``` shell title="Install Cert-Manager" linenums="1" ``
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true
```
Create Cluster Issuser for Lets Encrypt `vi letsencrypt-product.yaml` and paste the below contents adjust your email address
``` shell title="Create a cluster issuer manifest" linenums="1"
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: dinesh@dman.cloud
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```
``` shell title="Apply manifest" linenums="1"
kubectl apply -f letsencrypt-product.yaml
```
Deploy nginx-ingress controller
``` shell title="Apply manifest" linenums="1"
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml
```
Create ingress for ArgoCD `vi ingress.yaml` and paste the below contents adjust the domain name
``` shell title="Apply manifest" linenums="1"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # If you encounter a redirect loop or are getting a 307 response code
    # then you need to force the nginx ingress to connect to the backend using HTTPS.
    #
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.tabthree3.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.tabthree3.net
    secretName: argocd-secret # do not change, this is provided by Argo CD
```
``` shell title="Apply manifest" linenums="1"
kubectl apply -f ingress.yaml
```
# Install ArgoCD command line tool
Download With Curl
``` shell title="Run from shell" linenums="1"
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

## Login to ArgoCD from the CLI and change the password
``` shell title="Remember to swap your domain name below" linenums="1"
argocd login argocd.dev.dman.cloud
```
``` shell title="Update password" linenums="1"
argocd account update-password
```
# Adding cluster with managed k8s
Remember to change cluster name accordingly
``` argocd cluster add do-nyc1-k8s-1-28-2-do-0-nyc1-1701470051145 \
    --name do-nyc1-k8s-1-28-2-do-0-nyc1-1701470051145 \
    --kubeconfig ~/.kube/config
```
# Deploy Demo Application
You can use the below repository to deploy a demo nginx application
``` shell title="This repository has a sample application" linenums="1"
https://github.com/dmancloud/argocd-tutorial
```
### Scale Replicaset 
``` shell title="Run from shell prompt" linenums="1"
kubectl scale --replicas=3 deployment nginx -n default
```

## Clean Up
``` shell title="Run from shell prompt" linenums="1"
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```
