Here is the comprehensive, step-by-step guide to installing Argo CD on EKS and registering your cluster. Since you are using **AWS CloudShell**, these commands are optimized for that environment.

---

### 1. Install Argo CD

First, we create the namespace and apply the official manifests.

```bash
# 1. Create the namespace
kubectl create namespace argocd

# 2. Install Argo CD (Standard Non-HA for dev/test)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for components to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

```

---

### 2. Configure Load Balancer (NLB)

We will patch the `argocd-server` service to change its type to `LoadBalancer`. This triggers EKS to provision an AWS Network Load Balancer.

```bash
# Change service type to LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get the Load Balancer DNS name (wait 1-2 minutes for the 'EXTERNAL-IP' to appear)
kubectl get svc argocd-server -n argocd

```

> **Access Note:** Open your browser to `https://<EXTERNAL-IP>`. You will see a certificate warning; click **Advanced** -> **Proceed** (this is normal for self-signed certificates).

---

### 3. Login Credentials

The initial password for the `admin` user is stored in a secret.

```bash
# Retrieve and decode the password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

```

* **Username:** `admin`
* **Password:** [Output from command]

---

### 4. Register EKS Cluster (Token & CA Data)

If you are managing the **same cluster** Argo CD is installed on, it is already added as `https://kubernetes.default.svc`.

However, if you want to add it manually or as a remote cluster using **Token and CA data**, follow this YAML pattern:

#### A. Generate the Token and CA

```bash
# 1. Get Cluster API Server URL
SERVER_URL=$(aws eks describe-cluster --name <your-cluster-name> --query "cluster.endpoint" --output text)

# 2. Get Certificate Authority Data
CA_DATA=$(aws eks describe-cluster --name <your-cluster-name> --query "cluster.certificateAuthority.data" --output text)

# 3. Create a ServiceAccount and Secret to generate a Token (K8s 1.24+)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argocd-manager-token
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
EOF

# 4. Extract the Token
TOKEN=$(kubectl -n kube-system get secret argocd-manager-token -o jsonpath='{.data.token}' | base64 --decode)

 aws eks get-token --cluster-name costco --region us-east-1 | jq -r '.status.token'

 kubectl config view --raw -o jsonpath='{.clusters[?(@.name=="arn:aws:eks:us-east-1:117687871407:cluster/costco")].cluster.certificate-authority-data}'


 [[[[ get the bearer toekn -->  aws eks get-token --cluster-name costco --region us-east-1 | jq -r '.status.token'

get the CA data --> kubectl config view --raw -o jsonpath='{.clusters[?(@.name=="arn:aws:eks:us-east-1:117687871407:cluster/costco")].cluster.certificate-authority-data}'    ]]]]
```

#### B. Create the Argo CD Cluster Secret

create one argocd-secret.yaml file. Apply this YAML to register the cluster inside Argo CD: 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-eks-cluster-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: my-eks-cluster
  server: ""
  config: |
    {
      "bearerToken": "paste-toekn-here",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "paste-certificate-key-here"
      }
    }

```

*(Replace `${SERVER_URL}`, `${TOKEN}`, and `${CA_DATA}` with the values gathered above).*

---



### 5. Deploy an Application (GitOps) by creating argocd app  - Not Important-- instead use portal ArgoCD UI 

Finally, create an application manifest to deploy a sample project.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```

**Apply it:** `kubectl apply -f demo-app.yaml`

**Would you like me to help you connect a private GitHub repository using an SSH key so you can deploy your own code?**









INSTALL ARGOCD Commmand 

# 1. Download the latest Argo CD CLI binary
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# 2. Make the binary executable
chmod +x argocd-linux-amd64

# 3. Move it to your local bin directory
sudo mv argocd-linux-amd64 /usr/local/bin/argocd


argocd version --client
