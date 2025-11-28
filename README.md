# Deploying a replicated MongoDB StatefulSet on Kubernetes (AWS EKS) using a Helm chart

## Architecture Diagram

See [architecture-diagram.md](architecture-diagram.md) for a visual representation of the system architecture and traffic flow.

## Project Purpose

- Install a replicated MongoDB StatefulSet on Kubernetes (AWS EKS) using a Helm chart.
- Enable persistent storage using AWS GP3/EBS storage class.
- Deploy the Mongo Express UI to manage the database.
- Expose the UI via an NGINX ingress controller for browser access.

## Technologies Used

- Kubernetes (EKS)
- Helm
- MongoDB (replica set)
- Mongo Express (UI)
- Nginx Ingress Controller
- AWS EKS / AWS GP3/EBS storage
- Linux / kubectl / eksctl

## Repository Files

- **helm-ingress.yaml**
  - Kubernetes Ingress manifest to expose the Mongo Express service via Nginx ingress. Contains ingress class annotation and a rule with a host placeholder (`YOUR_HOST_DNS_NAME`) — replace with your DNS name or `/etc/hosts` entry.
- **helm-mongo-express.yaml**
  - Deployment + Service manifest for Mongo Express. Deployment reads MongoDB admin password from Kubernetes Secret (`mongodb`, key `mongodb-root-password`). Connection string set in `ME_CONFIG_MONGODB_URL`. Service exposes Mongo-Express on port 8081.
- **helm-mongodb.yaml**
  - Values file for MongoDB Helm chart (replica set, storageClass, root pw, image info, metrics toggle).

---

## Prerequisites

- AWS account with permissions for EKS, VPC, security groups, and EBS volumes.
- `eksctl` installed (or use AWS Console).
- `kubectl` installed/configured.
- Helm v3 installed.
- AWS CLI configured with credentials/region.

---

## Walkthrough Steps

### 1. Create an EKS Cluster (example: eksctl)

```bash
eksctl create cluster \
--name mongo-eks \
--version 1.34 \
--region us-east-1 \
--nodegroup-name mongo-nodes \
--node-type t3.large \
--nodes 2 \
--nodes-min 1 \
--nodes-max 3
```

- eksctl creates a kubeconfig for you. Verify with:
```bash
kubectl get nodes
```

---

### 2. Add the Bitnami Helm repo and install MongoDB as a replica set

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Install MongoDB using your values file:
```bash
helm install mongodb bitnami/mongodb \
    --values helm-mongodb.yaml \
    --set architecture=replicaset
```

If persistence/size is not specified in your file, add:
```bash
--set persistence.enabled=true --set persistence.size=8Gi
```

Confirm pods, services, PVCs:
```bash
kubectl get statefulsets, pods, svc,pvc
```

**Important:** Note the headless service name (likely `mongodb-headless`). Confirm the Mongo Express Deployment URL matches `mongodb-0.mongodb-headless:27017`.

---

### 3. Provide the MongoDB Password to Mongo Express

Create the Secret **before** applying the mongo-express manifest (use the same password as Helm chart):
```bash
kubectl create secret generic mongodb \
    --from-literal=mongodb-root-password='secret-root-pwd'
```

If the Helm chart created its own Secret, inspect and reference it, or create a readable one.

Verify the secret:
```bash
kubectl get secrets
kubectl get secret mongodb -o yaml
```

---

### 4. Deploy Mongo Express

Apply manifest:
```bash
kubectl apply -f helm-mongo-express.yaml
```

Check resources:
```bash
kubectl get deployments, pods, svc -l app=mongo-express
```

If Mongo Express cannot connect:
- Check logs:  
  ```bash
  kubectl logs <mongo-express-pod>
  ```
- Confirm DNS/service name in env `ME_CONFIG_MONGODB_URL`
- Ensure pod index `mongodb-0` exists.

---

### 5. Install Nginx Ingress Controller (Helm)

Add repo and install:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx --create-namespace
```

Verify external LoadBalancer:
```bash
kubectl get svc -n ingress-nginx
```
Note EXTERNAL-IP/hostname. You’ll map this in DNS or via `/etc/hosts` for testing.

___
### 6. Configure and Apply Ingress

Edit `helm-ingress.yaml` and replace `YOUR_HOST_DNS_NAME` with your chosen hostname (ex, `mongo.example.com`). Save and apply:
```bash
kubectl apply -f helm-ingress.yaml
```

Point DNS A record for `mongo.example.com` to the ingress controller’s external IP:
```bash
kubectl get svc -n ingress-nginx
```
In Route53 (or your DNS provider), create a DNS record mapping the hostname.  
For quick local testing, update `/etc/hosts` with mapping.

---

### 7. Access the UI

Visit in browser:
```
http://<YOUR_HOST_DNS_NAME>/
```

If you see a 404 or default backend:
```bash
kubectl describe ingress mongo-express
kubectl logs <ingress-nginx-pod>
```

---

### 8. Useful kubectl Commands for Verification

```bash
kubectl get pods -o wide
kubectl get statefulset
kubectl get pvc
kubectl describe pod <pod-name>
kubectl logs <pod-name> [-c <container>]
kubectl get svc
kubectl describe ingress mongo-express
```

---

### 9. Clean Up

```bash
kubectl delete -f helm-mongo-express.yaml
kubectl delete -f helm-ingress.yaml
helm uninstall mongodb
helm uninstall ingress-nginx -n ingress-nginx
eksctl delete cluster --name mongo-eks
```

---

## Troubleshooting Checklist

- **Pods CrashLoopBackOff?**  
  Check logs and describe the permission/errors.
- **Mongo Express cannot connect?**  
  Confirm secret, service names, namespace, and MongoDB readiness.
- **PVCs Pending?**  
  Check StorageClass `gp3` exists (EKS/AWS EBS CSI). If it is missing, use an available StorageClass or create one.

---

## License

MIT
