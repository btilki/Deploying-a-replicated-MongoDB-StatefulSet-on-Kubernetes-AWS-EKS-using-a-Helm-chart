# Deploying a replicated MongoDB StatefulSet on Kubernetes (AWS EKS) using a Helm chart

Project purpose
- Install a replicated MongoDB StatefulSet on Kubernetes (AWS EKS) using a Helm chart.
- Enable persistent storage using AWS gp3/EBS storage class.
- Deploy the Mongo Express UI to manage the database.
- Expose the UI via an nginx ingress controller so you can access it from a browser.

Technologies used
- Kubernetes (EKS)
- Helm
- MongoDB (replica set)
- Mongo Express (UI)
- nginx Ingress Controller
- AWS EKS / EBS (gp3 storage class)
- Linux / kubectl / eksctl

Repository files (brief descriptions)
- helm-ingress.yaml
  - Kubernetes Ingress manifest to expose the Mongo Express service via nginx ingress. It contains the ingress class annotation and a rule with a host placeholder (`YOUR_HOST_DNS_NAME`) that you must replace with your DNS name (or use an /etc/hosts entry for testing).
- helm-mongo-express.yaml
  - A Deployment + Service manifest for Mongo Express (containerized UI). The Deployment reads the MongoDB admin password from a Kubernetes Secret (`mongodb`, key `mongodb-root-password`) and builds a connection string environment variable `ME_CONFIG_MONGODB_URL`. The Service exposes mongo-express on port 8081.
- helm-mongodb.yaml
  - A values-style file intended for a Helm chart (not a direct Kubernetes manifest). It contains configuration options for a MongoDB Helm chart (replica set architecture, storageClass, root password, image info, metrics toggle). You will pass this to Helm when installing the MongoDB chart.

Prerequisites
- AWS account with permissions to create EKS clusters, VPC, security groups and EBS volumes.
- eksctl installed (or use AWS Console).
- kubectl installed and configured.
- Helm v3 installed.
- AWS CLI configured with credentials and region.
- (Optional) curl for convenience.

High-level steps (walkthrough)

1) Create an EKS cluster (example with eksctl)
- Example (adjust region, node type/count):
  - eksctl create cluster \
    --name mongo-eks \
    --region us-east-1 \
    --nodes 3 \
    --node-type t3.medium

- eksctl will create kubeconfig for you. Verify:
  - kubectl get nodes

2) Install nginx Ingress Controller (Helm)
- Create namespace and install:
  - helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  - helm repo update
  - helm install ingress-nginx ingress-nginx/ingress-nginx \
      --namespace ingress-nginx --create-namespace

- Verify the controller has an external LoadBalancer:
  - kubectl get svc -n ingress-nginx
  - Note the EXTERNAL-IP (or hostname). You will create a DNS A record pointing to this LB, or use that IP/hostname with /etc/hosts for testing.

3) Add the Bitnami (or other) Helm repo and install MongoDB as a replica set
- Add repo:
  - helm repo add bitnami https://charts.bitnami.com/bitnami
  - helm repo update

- The provided `helm-mongodb.yaml` is a values file (not a manifest). You can install Bitnami's MongoDB replicaset chart by passing this file. Example command:
  - helm install mongodb bitnami/mongodb \
      --values helm-mongodb.yaml \
      --set architecture=replicaset

  Notes:
  - If the chart you install expects different keys, you can override them via `--set` or adapt `helm-mongodb.yaml`.
  - Ensure persistence is enabled and size is set. If your `helm-mongodb.yaml` doesn't have `persistence.enabled: true` and `persistence.size`, either add them to the file or pass them on the command line:
    - --set persistence.enabled=true --set persistence.size=8Gi

- Confirm pods, services and PVCs:
  - kubectl get statefulsets,pods,svc,pvc

- Important: make note of the headless service name created by the chart. Commonly, for a release named `mongodb` it can be something like `mongodb-headless`. The Mongo Express Deployment in this repo uses `mongodb-0.mongodb-headless:27017` in the URL — verify that the headless service name and pod names match (see `kubectl get svc` and `kubectl get pods`).

4) Provide MongoDB password to Mongo Express
- The `helm-mongo-express.yaml` Deployment references a Secret called `mongodb` and the key `mongodb-root-password`. Create that secret prior to applying the mongo-express manifest, using the same password used by the Helm chart (or adjust accordingly).

  - Example (do not hardcode real passwords in repo):
    - kubectl create secret generic mongodb \
        --from-literal=mongodb-root-password='secret-root-pwd'

  - If the Helm chart you installed created its own secret (Bitnami charts often create secrets for the root user), inspect that secret and either reference it or create a new one that mongo-express can read.

  - Verify:
    - kubectl get secrets
    - kubectl get secret mongodb -o yaml  # check the key is present

5) Deploy Mongo Express
- Apply the provided file:
  - kubectl apply -f helm-mongo-express.yaml

- Check that the Deployment and Service are running:
  - kubectl get deployments,pods,svc -l app=mongo-express

- If mongo-express cannot connect to MongoDB:
  - Check `kubectl logs <mongo-express-pod>`
  - Confirm DNS / service names: the ME_CONFIG_MONGODB_URL points to `mongodb-0.mongodb-headless:27017` — ensure that name resolves in the same namespace and that the pod index `mongodb-0` exists (StatefulSet pods are numbered).

6) Configure and apply Ingress
- Edit `helm-ingress.yaml` and replace `YOUR_HOST_DNS_NAME` with the hostname you want to use (example `mongo.example.com`). Save and apply:
  - kubectl apply -f helm-ingress.yaml

- If your ingress uses the nginx ingress controller service LB, point DNS A record for `mongo.example.com` to the external IP/hostname of the ingress controller:
  - Get the external address:
    - kubectl get svc -n ingress-nginx
  - Create a DNS A (or CNAME) record in Route53 (or your DNS provider) pointing the hostname to the ingress controller's address.
  - For quick local testing, add an /etc/hosts entry mapping the host to the ingress controller external IP.

7) Access the UI
- In a browser, open `http://<YOUR_HOST_DNS_NAME>/` and you should see Mongo Express.
- If you see a 404 or a default backend, check:
  - kubectl describe ingress mongo-express
  - kubectl logs for ingress-nginx controller

8) Verification and useful kubectl commands
- Check pods:
  - kubectl get pods -o wide
- Check StatefulSet:
  - kubectl get statefulset
- Check PVCs:
  - kubectl get pvc
- Describe problem pods:
  - kubectl describe pod <pod-name>
  - kubectl logs <pod-name> [-c <container>]
- Inspect services:
  - kubectl get svc
- Inspect ingress:
  - kubectl describe ingress mongo-express

9) Clean up
- Remove resources:
  - kubectl delete -f helm-mongo-express.yaml
  - kubectl delete -f helm-ingress.yaml
  - helm uninstall mongodb
  - helm uninstall ingress-nginx -n ingress-nginx
  - Optionally delete EKS cluster:
    - eksctl delete cluster --name mongo-eks

Troubleshooting checklist
- Pods CrashLoopBackOff? Check logs and describe. Look for permission or connection errors.
- Mongo Express cannot connect? Verify
  - Secret value and key exist
  - Service/hostnames are correct and in the same namespace
  - MongoDB pods are ready and listening
- PVCs Pending? Check StorageClass `gp3` exists (EKS / AWS EBS CSI driver installed). If gp3 is not available, choose a StorageClass present in your cluster or create one.

License
MIT
