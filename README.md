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

sudo apt update && sudo apt install -y yamllint curl

curl -sL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.4.3/kustomize_v5.4.3_linux_amd64.tar.gz" | sudo tar xz -C /usr/local/bin

sudo apt install -y ansible-core or pip install ansible-core --break-system-packages

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

kubectl version --client
argocd version
ansible --version
kustomize version
```

**Windows users:** Use WSL2 for all commands.

### Validate Manifests Locally (Optional)

After installing all dependencies above, run a local validation scan:

```bash
cd ~/gitops/lightkube-gitops-homelab-project
> scan.log
echo "==== YAML LINT RESULTS ====" >> scan.log
yamllint . >> scan.log 2>&1
echo -e "\n==== KUSTOMIZE BUILD TESTS ====" >> scan.log
find . -name kustomization.yaml -not -path './.git/*' -print0 | while IFS= read -r -d '' file; do
  dir=$(dirname "$file")
  echo "Build test: $dir" >> scan.log
  kustomize build "$dir" >> scan.log 2>&1 || echo "FAILED: $dir" >> scan.log
done
cat scan.log
```

---

## Repository Structure

```
lightkube-gitops-homelab-project/
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini
│   ├── playbook-k3s-install.yml
│   └── vars/
│       └── main.yml
├── argocd/
│   ├── argocd-namespace.yaml
│   └── app-of-apps/
│       ├── kustomization.yaml
│       └── root-application.yaml
├── clusters/
│   └── single-node/
│       ├── kustomization.yaml
│       ├── infrastructure/
│       │   ├── kustomization.yaml
│       │   └── traefik/
│       │       ├── kustomization.yaml
│       │       └── traefik-application.yaml
│       └── applications/
│           ├── kustomization.yaml
│           ├── nginx-demo/
│           └── whoami-demo/
├── .gitignore
└── README.md
```

**App-of-Apps Pattern Explained:** One root Application deploys all other infrastructure & workload manifests – the standard GitOps architecture.

---

## Deployment Guide

### Step 1: Prepare Clean Ubuntu 24.04 LTS VM

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
sudo ufw allow 6443/tcp
```

### Step 2: Fork & Clone Your GitHub Repository

Fork this repository to your own GitHub account, then clone it:

```bash
git clone git@github.com:YOUR_USERNAME/lightkube-gitops-homelab-project.git
cd lightkube-gitops-homelab-project
```

### Step 3: Configure Ansible & Bootstrap K3s

**Edit Ansible Inventory (`ansible/inventory.ini`):**

```ini
[cluster_nodes]
k3s-node ansible_host=192.168.1.100 ansible_user=ubuntu
```

**Ansible Variables (`ansible/vars/main.yml`):**

```yaml
k3s_version: v1.30.2+k3s1
k3s_server_args: >-
  --disable traefik
  --tls-san 192.168.1.100
containerd_mirror: true
containerd_mirror_endpoint: "https://docker.m.daocloud.io"
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

    - name: Ensure /etc/rancher/k3s directory exists
      file:
        path: /etc/rancher/k3s
        state: directory
        mode: '0755'

    - name: Configure containerd registry mirror
      copy:
        dest: /etc/rancher/k3s/registries.yaml
        content: |
          mirrors:
            docker.io:
              endpoint:
                - "{{ containerd_mirror_endpoint }}"
        mode: '0644'
      when: containerd_mirror | bool

    - name: Install K3s Server
      shell: |
        curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh |
        INSTALL_K3S_VERSION={{ k3s_version }} \
        INSTALL_K3S_EXEC="server {{ k3s_server_args }}" \
        sh -
      args:
        executable: /bin/bash

    - name: Ensure .kube directory exists
      file:
        path: "/home/{{ ansible_user }}/.kube"
        state: directory
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'

    - name: Copy kubeconfig to regular user home directory
      copy:
        src: /etc/rancher/k3s/k3s.yaml
        dest: "/home/{{ ansible_user }}/.kube/config"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0600'

    - name: Fetch kubeconfig file to local workstation
      fetch:
        src: "/home/{{ ansible_user }}/.kube/config"
        dest: "./kubeconfig"
        flat: yes
```

**Execute the Ansible Playbook:**

```bash
cd ansible
ansible-playbook -i inventory.ini playbook-k3s-install.yml
```

After successful execution, set up kubeconfig on your local workstation. K3s writes `127.0.0.1` as the server address by default — rewrite it to your VM's IP before copying:

```bash
sed -i 's/127.0.0.1/192.168.1.100/' ansible/kubeconfig
cp ansible/kubeconfig ~/.kube/config
kubectl get nodes
```

> Replace `192.168.1.100` with your VM's actual static IP.

Expected result: Your single node appears with `Ready` status.

### Step 4: Deploy ArgoCD to the K3s Cluster

```bash
kubectl apply -f argocd/argocd-namespace.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd --watch
```

Wait until all pods show `Running` status, then retrieve the initial ArgoCD admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Access the ArgoCD UI via temporary port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Visit: `https://localhost:8080` | Username: `admin`

### Step 5: Authenticate ArgoCD to GitHub (Critical Step)

**Recommended Option: Use a private GitHub repository**

Generate an SSH key without a passphrase:

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/argocd-git
```

Copy the public key and add it as a Deploy Key inside your GitHub repository (Settings → Deploy keys → Add deploy key, check "Allow write access" if you want ArgoCD to auto-sync):

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
  url: git@github.com:YOUR_USERNAME/lightkube-gitops-homelab-project.git
  sshPrivateKey: |
$(sed 's/^/    /' ~/.ssh/argocd-git)
EOF
```

### Step 6: Deploy ArgoCD Root Application (App-of-Apps)

Edit `argocd/app-of-apps/root-application.yaml` and replace the `repoURL` with your forked repository's SSH URL:

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

### Step 7: Verify GitOps Synchronisation

Open the ArgoCD UI. You will see `root-app` automatically provision all infrastructure components (Traefik) and demo workloads (nginx-demo, whoami-demo).

List all managed applications:

```bash
argocd app list -n argocd
```

Check that all resources are deployed:

```bash
kubectl get pods -A
kubectl get ingress
```

### Step 8: Test the End-to-End GitOps Workflow

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

**Success indicator:** No manual `kubectl apply` required. Git acts as the single source of truth for cluster state.

### Step 9: Configure Ingress & Local DNS

Traefik will be deployed automatically via the app-of-apps pattern from `clusters/single-node/infrastructure/traefik`.

Retrieve Traefik service IP:

```bash
kubectl get svc traefik -n traefik
```

Update your workstation's `/etc/hosts` file for local domain resolution:

```plaintext
192.168.1.100 whoami.local nginx.local
```

Test the ingress endpoints:

```bash
curl -H "Host: whoami.local" http://192.168.1.100
curl -H "Host: nginx.local" http://192.168.1.100
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
- Verify deploy key added to GitHub (Settings → Deploy keys)
- Check secret `argocd-github-ssh` exists in `argocd` namespace: `kubectl get secret -n argocd`
- Confirm repo URL uses SSH format `git@github.com:...`, not HTTPS
- Test SSH connection: `ssh -T git@github.com`

**Pod image pull stuck (China network)**
- The Ansible playbook configures containerd mirror via `/etc/rancher/k3s/registries.yaml`
- Verify the mirror config exists on the node: `sudo cat /etc/rancher/k3s/registries.yaml`
- If still failing, try a different mirror endpoint in `ansible/vars/main.yml` and re-run the playbook
- Restart K3s after changing registries.yaml: `sudo systemctl restart k3s`

**ArgoCD OutOfSync errors**
- Ensure `syncPolicy: automated: prune: true selfHeal: true` is set in the Application
- Never manually edit resources inside cluster – always edit Git manifests
- Check ArgoCD application controller logs: `kubectl logs -n argocd deploy/argocd-application-controller`

**Ingress cannot reach service**
- Confirm static VM IP and that Traefik pod is running: `kubectl get pods -n traefik`
- Verify firewall on Ubuntu allows ports 80/443:
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```
- Check Ingress resource: `kubectl describe ingress whoami-demo`
- Ensure `ingressClassName: traefik` is set on all Ingress resources

**K3s node not Ready**
- Check K3s service status: `sudo systemctl status k3s`
- Check K3s logs: `sudo journalctl -u k3s -f`
- Verify containerd is running: `sudo crictl ps`

### Useful Debug Commands

```bash
argocd app get root-app
argocd app get traefik
argocd repo list
kubectl logs -n argocd deploy/argocd-application-controller
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```
