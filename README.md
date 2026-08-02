This project provides a lightweight Kubernetes GitOps stack for single-node self-hosting. Built with:

- **Ubuntu 24.04 LTS** – Full Linux, SSH accessible
- **K3s** – Lightweight certified Kubernetes distribution
- **Ansible** – Automated node bootstrapping
- **ArgoCD** – GitOps continuous deployment
- **App-of-Apps Pattern** – Standard GitOps architecture

**Core GitOps Principle:** All Kubernetes YAML lives in Git → ArgoCD automatically syncs cluster state → No manual `kubectl apply` after initial setup.

---

## Minimum Hardware

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| VM / Mini PC | 1 | 1 |
| RAM | 6GB | 8GB |
| Disk | 40GB SSD | 40GB SSD |
| Network | Static local IP | Static local IP |

---

## Local Workstation Dependencies

Set up these utilities on your laptop (Ubuntu / WSL2 / macOS):

```bash
mkdir -p ~/gitops
cd ~/gitops
git clone https://github.com/Drsweets/lightkube-gitops-homelab-project.git
cd lightkube-gitops-homelab-project

sudo apt update && sudo apt install -y kustomize yamllint
curl -s https://fluxcd.io/install.sh | sudo bash

> scan.log
echo "==== YAML LINT RESULTS ====" >> scan.log
yamllint . >> scan.log 2>&1

echo -e "\n==== KUSTOMIZE BUILD TESTS ====" >> scan.log
find . -name kustomization.yaml -print0 | while IFS= read -r -d '' file; do
  dir=$(dirname "$file")
  echo "Build test: $dir" >> scan.log
  kustomize build "$dir" >> scan.log 2>&1 || echo "FAILED: $dir" >> scan.log
done

echo -e "\n==== FLUX VALIDATION ====" >> scan.log
flux validate sources >> scan.log 2>&1
flux validate kustomizations >> scan.log 2>&1

cat scan.log
pip install ansible-core

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

kubectl version --client
argocd version
ansible --version
```

**Windows users:** Use WSL2 for all commands.

---

## Repository Structure

```
lightkube-gitops-homelab-project/
├── ansible/
│   ├── inventory.ini
│   ├── playbook-k3s-install.yml
│   └── vars/
│       └── main.yml
├── argocd/
│   ├── argocd-namespace.yaml
│   ├── argocd-install.yaml
│   └── app-of-apps/
│       ├── kustomization.yaml
│       └── root-application.yaml
├── clusters/
│   └── single-node/
│       ├── kustomization.yaml
│       ├── infrastructure/
│       │   ├── traefik/
│       │   ├── kustomization.yaml
│       └── applications/
│           ├── nginx-demo/
│           ├── whoami-demo/
│           └── kustomization.yaml
├── .gitignore
└── README.md
```

**App-of-Apps Pattern Explained:** One root Application deploys all other infrastructure & workload manifests – the standard GitOps architecture.

---

## Deployment Guide

### Prepare Clean Ubuntu 24.04 LTS VM

Deploy fresh Ubuntu Server 24.04 LTS (no desktop) and configure a static IP address on your VM (example: `192.168.1.100`). Enable OpenSSH server during installation and create a regular user with sudo privileges (example: `ubuntu`).

From your local workstation, test SSH connectivity:

```bash
ssh ubuntu@192.168.1.100
```

On the VM, update the base system and configure firewall:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openssh-server ca-certificates curl -y
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### Clone Your GitHub Repository

```bash
git clone git@github.com:YOUR_USERNAME/lightkube-gitops-homelab-project.git
cd lightkube-gitops-homelab-project
```

### Configure Ansible & Bootstrap K3s

**Edit Ansible Inventory (`ansible/inventory.ini`):**
```ini
[cluster_nodes]
k3s-node ansible_host=192.168.1.100 ansible_user=ubuntu
```

**Ansible Variables (`ansible/vars/main.yml`):**
```yaml
k3s_version: v1.30.2+k3s1
k3s_server_args: >-
  --disable servicelb
  --disable traefik
  --tls-san 192.168.1.100
containerd_mirror: true
```

**Ansible Playbook (`ansible/playbook-k3s-install.yml`):**
```yaml
---
- name: Bootstrap single-node K3s Ubuntu host
  hosts: cluster_nodes
  become: true
  vars_files:
    - vars/main.yml
  tasks:
    - name: Install system dependencies
      apt:
        name:
          - curl
          - gnupg
          - lsb-release
        update_cache: yes

    - name: Install K3s Server
      shell: >
        curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh |
        INSTALL_K3S_VERSION={{ k3s_version }}
        INSTALL_K3S_EXEC="server {{ k3s_server_args }}"
        sh -
      args:
        executable: /bin/bash

    - name: Copy kubeconfig to user home
      copy:
        src: /etc/rancher/k3s/k3s.yaml
        dest: /home/{{ ansible_user }}/.kube/config
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0600'

    - name: Fetch kubeconfig to local workstation
      fetch:
        src: /home/{{ ansible_user }}/.kube/config
        dest: ./kubeconfig
        flat: yes
```

**Execute the Ansible Playbook:**
```bash
cd ansible
ansible-playbook -i inventory.ini playbook-k3s-install.yml
```

After successful execution:
```bash
cp ansible/kubeconfig ~/.kube/config
kubectl get nodes
```

Expected result: Your single node appears with `Ready` status.

### Deploy ArgoCD to the K3s Cluster

```bash
kubectl apply -f argocd/argocd-namespace.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd --watch
```

Retrieve the initial ArgoCD admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Access the ArgoCD UI via temporary port-forward:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Visit: `https://localhost:8080` | Username: `admin`

### Authenticate ArgoCD to GitHub (Critical Step)

**Recommended Option: Use a private GitHub repository**

Generate an SSH key without a passphrase:
```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/argocd-git
```

Copy the public key and add it as a Deploy Key inside your GitHub repository:
```bash
cat ~/.ssh/argocd-git.pub
```

Create a Kubernetes Secret containing SSH credentials for ArgoCD:
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argocd-github-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:Drsweets/lightkube-gitops-homelab-project.git
  sshPrivateKey: |
$(sed 's/^/    /' ~/.ssh/argocd-git)
EOF
```

### Deploy ArgoCD Root Application (App-of-Apps)

Edit `argocd/app-of-apps/root-application.yaml` and replace the following values:
- `repoURL`: SSH URL of your GitHub repository
- `targetRevision`: `main`
- `path`: `clusters/single-node`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:YOUR_USERNAME/lightkube-gitops-homelab-project.git
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
```

Apply the root application:
```bash
kubectl apply -f argocd/app-of-apps/root-application.yaml
```

### Verify GitOps Synchronisation

Open the ArgoCD UI. You will see `root-app` automatically provision all infrastructure components and demo workloads.

List all managed applications:
```bash
argocd app list -n argocd
```

### Test the End-to-End GitOps Workflow

This test confirms your GitOps pipeline functions correctly:

1. Modify a manifest locally (e.g., change replica count inside `clusters/single-node/applications/whoami-demo/deployment.yaml` from `1` to `2`)
2. Commit and push changes to GitHub:
```bash
git add .
git commit -m "Scale whoami to 2 replicas for GitOps validation"
git push origin main
```
3. Wait roughly 60 seconds — ArgoCD detects the Git change and rolls out updates automatically
4. Confirm changes on the cluster:
```bash
kubectl get deploy whoami-demo
```

✅ **Success indicator:** No manual `kubectl apply` required. Git acts as the single source of truth for cluster state.

### Configure Ingress & Local DNS

Traefik will be deployed from manifests located at `clusters/single-node/infrastructure/traefik`.

Retrieve Traefik service IP:
```bash
kubectl get svc traefik -n traefik
```

Update your workstation's `/etc/hosts` file for local domain resolution:
```plaintext
192.168.1.100 whoami.local nginx.local
```

---

## Core GitOps Workflow

- Edit Kubernetes YAML locally
- Git commit & push to GitHub
- ArgoCD detects change automatically
- Cluster state updates without manual `kubectl` commands

---

## Learning Objectives

- Kubernetes fundamental resources (Namespace, Deployment, Service, Ingress, PVC)
- Lightweight K3s operation
- GitOps methodology
- Kustomize resource management
- ArgoCD App-of-Apps architecture
- Ingress, DNS, network configuration
- Infrastructure automation with Ansible

---

## Troubleshooting

### Common Issues

**ArgoCD cannot connect to GitHub private repo**
- Verify deploy key added to GitHub
- Check secret `argocd-github-ssh` exists in `argocd` namespace
- Confirm repo URL uses SSH format `git@github.com:...`, not HTTPS

**Pod image pull stuck (China network)**
- The Ansible playbook enables containerd mirror. If still failing, add docker mirror registry inside `/etc/rancher/k3s/registries.yaml`

**ArgoCD OutOfSync errors**
- Ensure `syncPolicy: automated: prune: true selfHeal: true`
- Never manually edit resources inside cluster – always edit Git manifests

**Ingress cannot reach service**
- Confirm static VM IP
- Verify firewall on Ubuntu allows ports 80/443:
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### Useful Debug Commands

```bash
argocd app get root-app
argocd repo list
kubectl logs -n argocd deploy/argocd-application-controller
```

---
