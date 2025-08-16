# App Deployment on Kubernetes (Docker Desktop/Minikube) with Jenkins, Vault, and NGINX Ingress

This guide explains how to set up a local Kubernetes environment (Docker Desktop Kubernetes) to run an application with:

- **HashiCorp Vault** for secrets management
- **NGINX Ingress Controller** for HTTP routing
- **Jenkins** for CI/CD automation
- **Helm** for deployments

---

## 1. Prerequisites

- **Docker Desktop** with Kubernetes enabled
- **kubectl** and **helm** installed

---

## 2. Install HashiCorp Vault (Dev Mode) and Add Django Password

#### Add Vault Helm repo
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
```

#### Create Vault namespace
```bash
kubectl create namespace vault`
```
#### Install Vault in dev mode
```bashß
helm install -n vault vault hashicorp/vault --set "server.dev.enabled=true"
```

#### Enable Kubernetes authentication in Vault
```bash
kubectl exec -it vault-0 -n vault -- vault auth enable kubernetes
```

#### Configure Kubernetes auth with the cluster's API server
```bash
kubectl exec -it vault-0 -n vault -- sh -c 'vault write auth/kubernetes/config \`
    `kubernetes_host=https://$KUBERNETES_PORT_443_TCP_ADDR:443'
```

#### Store Django superuser password
```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/infrastore-app DJANGO_SUPERUSER_PASSWORD=secret123
```

---

## 3. Install Nginx Ingress Controller

#### Create namespace for ingress
```bash
kubectl create namespace ingress-nginx
```

#### Install ingress-nginx via Helm
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.publishService.enabled=true \
  --set controller.replicaCount=1 \
  --set controller.service.type=LoadBalancer
```

## 4. Install Jenkins

#### Add Jenkins repo
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

#### Create namespace for Jenkins
```bash
kubectl create namespace jenkins
```

#### Install Jenkins with ingress enabled
```bash
helm upgrade --install jenkins jenkins/jenkins \
  --namespace jenkins \
  --set controller.serviceType=ClusterIP \
  --set controller.ingress.enabled=true \
  --set controller.ingress.hostName=jenkins.local \
  --set controller.ingress.annotations."nginx\.ingress\.kubernetes\.io/rewrite-target"=/ \
  --set controller.ingress.path=/ \
  --set controller.admin.username=admin \
  --set controller.admin.password=admin123 \
  --set persistence.enabled=false \
  --set controller.ingress.ingressClassName=nginx
  ```


#### Give Jenkins Access to the Cluster
```bash
kubectl create serviceaccount jenkins-sa -n jenkins
```

```bash
kubectl create clusterrolebinding jenkins-sa-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=jenkins:jenkins-sa
  ```


## 3. Helm Chart: infrastore

This Helm chart (`infrastore`) is used to deploy the application into a Kubernetes cluster using Jenkins.  
It integrates with **HashiCorp Vault** for secret management and supports PVCs for persistent storage.

---

#### Chart Structure
```tree
testapp
├── jenkinsfile                         # Jenkinsfile for deployment 
├── README.md
└── infrastore
    ├── Chart.yaml                      # Chart metadata
    ├── charts
    ├── templates
    │   ├── _helpers.tpl
    │   ├── deployment.yaml             # Deployment manifest
    │   ├── hpa.yaml                    # Horizontal Pod Autoscaler
    │   ├── ingress.yaml                # App Ingress resource
    │   ├── NOTES.txt
    │   ├── pvc.yaml                    # PersistenceVolumeClaims for media and DB
    │   ├── secret.yaml                 # K8 secret for Django password
    │   ├── service.yaml                # Service to expose app
    │   ├── serviceaccount.yaml         # ServiceAccount for the app
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml             # Default Values for chart configuration
```


#### Following command is used to deploy the application in the namespace appns
```bash
helm upgrade --install testapp infrastore \
  --namespace appns --create-namespace \
  --set-string secret.DJANGO_SUPERUSER_PASSWORD="$DJANGO_PASSWORD"
```
