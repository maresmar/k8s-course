# K8S practice
Sources for a practice lecture about containers used in NSWI026 MFF UK and SEPA4M33SEP, FEL CVUT. This should be a simple introduction for k8s, it's not recommended for a production usage.

## Tasks

1. Install minikube
2. Observe cluster
    - Try k8s dashboard
    - (Optional) Try k9s
3. Deploy resource using kubectl
4. Deploy resource using helm


## 01 Setup
We will use [Minikube](https://minikube.sigs.k8s.io/docs/start).

```bash
# Install minikube
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# Start minikube
minikube start

# Optional steps
minikube config set rootless true
minikube delete
minikube start
```

You will need to have `kubectl`, I successfully installed it using [Brew](https://docs.brew.sh/Homebrew-on-Linux).

```bash
# Install kubectl using brew
brew install kubectl
# Test if it works fine
kubectl get po -A

# Optional, try k9s
brew install k9s
```

## 02 Observe
Let's check a first deployment, we can try it using Kubernetes Dashboard

>  Ingress -> Service -> Pod { Container
> Check namespaces

```bash
minikube addons enable metrics-server
minikube dashboard
```

Check the cluster using dashboard in browser, eg http://127.0.0.1:37737/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/

Optional, check it using `k9s`.

## 03 Deploy a resource using kubectl

Let's try first deployment of an Nginx container,

```bash
kubectl apply -f 01-deployment.yaml
```

Try a port forwarding using `k9s` or `kubectl proxy` 

```bash
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

kubectl proxy
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:80/proxy/
```

> We use deployment for nginx.  Deployment is stateless (name and IP could change), data are stored in ephemeral storage (eg. `/tmp` folder) or shares a single PersistentVolumeClaim (PVC) across all pods. StatefulSet is statuful (name and IP is stable), data are stored in volumeClaimTemplates to create a unique, persistent PVC for each pod (e.g., data-db-0, data-db-1).

## 04 Deploy a resource using Helm

We need to install `helm`, I was successfull using `brew`. See full docs on https://helm.sh/docs/topics/charts/.

```bash
# Install dependencies
brew install helm

# Test the template
helm template hello-helm --values hello-helm/values.yaml
```

Let's deploy with helm
```bash
# Remove old changes
kubectl delete -f 01-deployment.yaml

# Let's use helm install
helm install helm-release hello-helm --values hello-helm/values.yaml
helm list
```