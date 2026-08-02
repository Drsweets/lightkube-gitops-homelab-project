Set up these utilities on your laptop (Ubuntu / WSL2 / macOS):

# Install Ansible
pip install ansible-core

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Install ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Validate installations
kubectl version --client
argocd version
ansible --version


Confirm the VM uses a static IP address

Allow HTTP/HTTPS traffic within Ubuntu firewall:

sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo apt update && sudo apt upgrade -y
sudo apt install openssh-server ca-certificates curl -y

git clone git@github.com:YOUR_USERNAME/lightkube-gitops-homelab-project/

cd lightkube-gitops-homelab-project

Execute the Ansible Playbook

cd ansible
ansible-playbook -i inventory.ini playbook-k3s-install.yml

After successful execution:
cp ansible/kubeconfig ~/.kube/config
# Verify cluster connection
kubectl get nodes

Expected result: Your single node appears with Ready status.

Deploy ArgoCD to the K3s Cluster

# Create namespace and install ArgoCD
kubectl apply -f argocd/argocd-namespace.yaml
kubectl apply -f argocd/argocd-install.yaml

# Watch pod startup progress (approximately 3–5 minutes)
kubectl get pods -n argocd --watch


Retrieve the initial ArgoCD admin password:

kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

Access the ArgoCD UI via temporary port-forward:

kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

Access the ArgoCD UI via temporary port-forward:

kubectl port-forward svc/argocd-server -n argocd 8080:443

Visit: https://localhost:8080 | Username: admin

Step 5: Authenticate ArgoCD to GitHub (Critical Step)

Recommended Option: Use a private GitHub repository

Generate an SSH key without a passphrase:

ssh-keygen -t ed25519 -N "" -f ~/.ssh/argocd-git

Copy the public key and add it as a Deploy Key inside your GitHub repository:

cat ~/.ssh/argocd-git.pub

Create a Kubernetes Secret containing SSH credentials for ArgoCD:

kubectl create secret generic argocd-github-ssh \
  -n argocd \
  --from-file=ssh-privatekey=~/.ssh/argocd-git \
  --from-literal=knownHosts="github.com ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81f6NvPxCeehhlpNujzbDmzaAhoUkniXB045EtjuVpw7MIh8lTLWXcrsWHFdKJPH2l8PNizg2Ux3fmT4nCYuqU5RxcVFA6bAiDYksq24TfrFSe2MjHKC2T4khkNX7AnQj73KuDNCZXAuW2z0muo6lbCuVvAfui3MjMxMk54GftzHq6P4um6awpZhEvz4EZSXuyZfqmqzhIbNsM5YVbVqDV5HkvN1ercw=="

Step 6: Deploy ArgoCD Root Application (App-of-Apps)

Edit argocd/app-of-apps/root-application.yaml and replace the following values:

repoURL: SSH URL of your GitHub repository

targetRevision: main

path: clusters/single-node

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:YOUR_USERNAME/k3s-gitops-starter.git
    targetRevision: main
    path: clusters/single-node
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true


kubectl apply -f argocd/app-of-apps/root-application.yaml

 Verify GitOps Synchronisation
 
Open the ArgoCD UI. You will see root-app automatically provision all infrastructure components and demo workloads.

List all managed applications:

argocd app list -n argocd

Test the End-to-End GitOps Workflow

This test confirms your GitOps pipeline functions correctly:

Modify a manifest locally (e.g., change replica count inside clusters/single-node/applications/whoami-demo/deployment.yaml from 1 to 2)

Commit and push changes to GitHub:

git add .
git commit -m "Scale whoami to 2 replicas for GitOps validation"
git push origin main

Wait roughly 60 seconds — ArgoCD detects the Git change and rolls out updates automatically

Confirm changes on the cluste

kubectl get deploy whoami-demo

✅ Success indicator: No manual kubectl apply required. Git acts as the single source of truth for cluster state.

Step 9: Configure Ingress & Local DNS

Traefik will be deployed from manifests located at clusters/single-node/infrastructure/traefik.

Retrieve Traefik service IP:

kubectl get svc traefik -n traefik

Update your workstation’s /etc/hosts file for local domain resolution:

192.168.1.100 whoami.local nginx.local

Useful Debug Commands

# Check ArgoCD application sync status
argocd app get root-app

# List configured Git repository connections
argocd repo list

# Inspect ArgoCD controller logs
kubectl logs -n argocd deploy/argocd-application-controller
