# K8S practice
Sources for a practice lecture about containers used in NSWI026 MFF UK and SEPA4M33SEP, FEL CVUT. This should be a simple introduction for k8s, it's not recommended for a production usage.

## Tasks

1. Install minikube
2. Observe cluster
    - Try k8s dashboard
    - (Optional) Try k9s
3. Deploy resource using kubectl
4. Deploy resource using helm
5. Try GitOps and ArgoCD


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

Let's see what minikube is simulating for us:

![k8s cluster, The Kubernetes Authors, CC BY 4.0](img/kubernetes-cluster-architecture-1.svg)

## 02 Observe
Let's check a first deployment, we can try it using Kubernetes Dashboard


```bash
minikube addons enable metrics-server
minikube dashboard
```

Check the cluster using dashboard in browser, eg. http://127.0.0.1:37737/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/.

Observe existing resoruces in cluster

- See Service -> Pod -> Container
- Check namespaces

![Service, The Kubernetes Authors, CC BY 4.0](img/module_04_labels.svg)

## 03 Deploy a resource using kubectl

Let's try first deployment of an Nginx container,

```bash
kubectl apply -f 01-deployment.yaml
```

You can validate results in dashboard or using `k9s`.

### Try to access new resources

- Pod 
  - This is mostly for debug purposes
  - Pod name change with each restart
    ```bash
    export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
    echo Name of the Pod: $POD_NAME
    ```
  - Let's try `kubectl proxy` (port-forward would work as well)
    ```bash
    kubectl proxy
    curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:80/proxy/
    ```

- Service
    - Service will be stable, even if pod is recreated
    - Let's try `kubectl port-forward` (proxy would work as well)
      ```bash
      kubectl port-forward svc/nginx 9090:80
      curl http://localhost:9090
      ```
- Ingress 
  - We would normally have an ingress that will be pointing to a service.

### Deployment vs StatefulSet

We use deployment for `nginx`.  Deployment is stateless (name and IP could change), data are stored in ephemeral storage (eg. `/tmp` folder) or shares a single PersistentVolumeClaim (PVC) across all pods. StatefulSet is statuful (name and IP is stable), data are stored in volumeClaimTemplates to create a unique, persistent PVC for each pod (e.g., data-db-0, data-db-1).

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

## 05 ArgoCD

Let's try to automate deployment using GitOps principles. 

We can deploy ArgoCD to our cluster.

```bash
# Remove previous deployment
helm uninstall helm-release

# Create a new namespace
kubectl create ns argocd
# Deploy a new resources
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.5.8/manifests/install.yaml
# Forward port to localhost
kubectl port-forward svc/argocd-server -n argocd 9090:443
```

Let's check ArgoCD web GUI at https://localhost:9090, default username is `admin`, password we get using

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Add new project in ArgoCD using following config:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example
spec:
  project: default
  source:
    repoURL: 'https://github.com/maresmar/k8s-course.git' # Or better replace with your repository
    path: 02-helm/hello-helm
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy: {}
```

Use GUI to "SYNC" changes, and observe results. 

You could also commit a new message in `cm.yaml` and try to "REFRESH" (changes from GitHub) and "SYNC" them to cluster.

## Resources
- https://minikube.sigs.k8s.io/docs/handbook/controls/
- https://k9scli.io/
- https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/
- https://argo-cd.readthedocs.io/en/stable/
- https://medium.com/@mehmetodabashi/installing-argocd-on-minikube-and-deploying-a-test-application-caa68ec55fbf