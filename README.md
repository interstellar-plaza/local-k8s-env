# Sample DevOps Lab — Full Scripted Walkthrough on macOS

This document walks you through every step of the local main k8s env for sample PodInfo application running inside the cluster behind Traefik proxy and secured by Keyklock authorization layer. This main k8s cluster has a connection with a VM running Kubernetes on Microk8s with PodInfo application as well. Automations is managed by Ansible as a test approach but for prod envrionment it's suggested to use helm with GitOps.

---

## 1. Project Structure

```
devops-assmt/
├── group_vars/
│   └── all.yml 
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── stg/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── releases/
│   ├── release-1.0.0.yml
│   └── release-0.9.0.yml
├── roles/
│   ├── keycloak/
│   │   └── tasks/main.yml
│   ├── podinfo/
│   │   └── tasks/main.yml
│   ├── podinfo_edge/
│   │   └── tasks/main.yml
│   ├── traefik/
│   │   └── tasks/main.yml
├── .gitignore
├── ansible.cfg
├── deploy.yml
└── README.md
```

---

## 2. Install Prerequisites

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install all required tools
- Install docker Desktop: https://docs.docker.com/desktop/setup/install/mac-install/.
- brew install ansible
- brew install multipass
- brew install kubectl
- brew install helm

# Verify versions
> docker --version
> ansible --version
> helm version
> multipass --version
> kubectl version --client
```

Verify that localhost executed the Ansible has the following packages installed:
- python >= 3.9
- kubernetes >= 24.2.0
- PyYAML >= 3.11
- jsonpatch

e.g `pip3 install pyyaml kubernetes jsonpatch --break-system-packages`

---

## 3. Enable the Main Kubernetes Cluster (Docker Desktop)

Docker Desktop ships with a built-in single-node Kubernetes cluster via `Kind` or `Kubeadm`. No separate cluster tool is needed. We deploy cluster via `Kind`.

### 3.1 Enable Kubernetes in Docker Desktop

1. Open **Docker Desktop**
2. Go to **Settings → Kubernetes**
3. Check **Enable Kubernetes**

### 3.2 Switch kubectl context and verify

```bash
# Set kubectl to use the Docker Desktop cluster
kubectl config use-context docker-desktop

# Confirm cluster is up
kubectl cluster-info
kubectl get no
# Expected: one node named "docker-desktop" in Ready state
```

> **Why Docker Desktop instead of lightweight kind?** Docker Desktop's built-in Kubernetes cluster uses `LoadBalancer` services that bind directly to `localhost` via the macOS host networking bridge — no manual port-mapping config required. Traefik's `LoadBalancer` service will be reachable on `localhost:80` and `localhost:443` immediately after deploy, which simplifies the networking setup.

---

## 4. Create the Edge Server VM (Multipass + MicroK8s)

```bash
# Launch the VM and check VM is running
multipass launch --name edge-server --cpus 2 --memory 4G --disk 20G

# Make sure that terminal you are using Terminal.app, iTerm, Warp, etc.) is not blocked from local network access by macOS.
# Open System Settings, Go to Privacy & Security → Local Network and check if your terminal toggle is ON
multipass shell edge-server

# --- Inside the VM ---
sudo snap install microk8s --classic
sudo usermod -aG microk8s ubuntu
mkdir ~/.kube
sudo chown -R ubuntu ~/.kube
newgrp microk8s
sudo apt install python3-pip -y
pip3 install pyyaml kubernetes jsonpatch --break-system-packages

microk8s status --wait-ready
microk8s enable dns storage ingress # It deploys CoreDNS, basic NGINX ingress controller inside the cluster and configures local storageClass to allow PersistentVolumeClaim requests

# Get the VM's IP (you'll need it for Ansible inventory)
hostname -I
# Exit the VM
exit
```

```bash
# From macOS — confirm you can reach it
multipass list
```

---

## 5. Ansible Configuration
- `ansible.cfg` is used for Ansible default configuration while running playbook
- `invenories/<ENV>/group_vars/all.yml` contains variables to be used in ansible playbook
- `invenories/<ENV>/hosts.yml` contains workload resource which should be confitured by Ansible
- `group_vars/all.yml` contains global ansible variables 

### 5.1 SSH key setup for Multipass

```bash
# Generate a key if you don't have one
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Add public ssh key to edge-server autorized keys
multipass shell edge-server
# open ~/.ssh/authorized_keys and paste ~/.ssh/id_rsa.pub, save and exit

# Verify
ssh ubuntu@edge-server.local "echo connected"
```

---

## 6. Ansible Roles

### 6.1 Traefik role
We are using Traefik to deploy via Ansible role under `roles/traefik/tasks/main.yml` as ingress controller forwarding requests to the PodInfo App.

> **Why Traefik?** It integrates natively with Kubernetes via IngressRoute CRDs and has first-class support for ForwardAuth middleware — which is how we hook Keycloak authentication. Also it has embedded LetsEncrypt support for external traffic encryption.

### 6.2 Keycloak role
We are roll out Keycloak via role under `roles/keycloak/tasks/main.yml`. It's used to provide authorization layer for PodInfo sample application.

### 6.3 Podinfo role (main cluster)
We are deploying PodInfo REST API application for test purpuses accessible only inside the cluster. Ansible role is stored under `roles/podinfo/tasks/main.yml`

### 6.4 Edge server Podinfo role
Edge server is deployed via Ansible role stored in `roles/podinfo_edge/tasks/main.yml`

### 6.5 Master Playbook
Ansible entrypoint containing the role execution sequence is store in `deploy.yml`

Ansible role execution instruction is provided in statement 8 and 9.   

---

## 7. Run the Initial Deployment in Dev

```bash
# Install required Ansible collections if not installed. By default they go with Ansible
ansible-galaxy collection install kubernetes.core community.general

# Deploy release 1.0.0 to dev
ansible-playbook deploy.yml -i inventories/dev -e @releases/release-1.0.0.yml

# Confirm Podinfo version on main cluster
kubectl get po -n podinfo-{ENV}
kubectl exec -n podinfo-{ENV} deploy/podinfo -- wget -qO- http://localhost:9898/version

# Confirm Podinfo on edge
curl http://<EDGE-SERVER-IP>:30099/version
...
{
  "commit": "b501abd1f0f2acf49c39f2d9cd47b460578d87f0",
  "version": "6.11.2"
}
```

---

## 8. Keycloak — Manual Post-Deploy Configuration

After Keycloak is running, configure it via the admin UI:

```bash
# Add keycloak.localhost to /etc/hosts
echo "127.0.0.1  keycloak.localhost podinfo.localhost" | sudo tee -a /etc/hosts

# Wait for Keycloak to be ready
kubectl wait --for=condition=ready pod -l app=keycloak -n keycloak --timeout=120s

# Open in browser
open http://keycloak.localhost
```

Then in the Keycloak Admin Console (admin / admin):

1. **Create a realm**: Manage Realms → Create a realm → name it `myrealm`
2. **Create a client**:
   - Clients → Create client
   - Client ID: `podinfo-client`
   - Client authentication: ON
   - Authentication flow: `Standard flow` is enabled
   - Valid redirect URIs: `http://podinfo.localhost/*`
3. **Create a test user**:
   - Users → Add user → username: `testuser`
   - Credentials tab → Set password: `testpass` → Temporary: OFF

> **Why ForwardAuth?** Traefik's ForwardAuth middleware intercepts every request to the protected route and forwards it to Keycloak's userinfo endpoint. If the token is valid, the request proceeds. If not, the user gets a 401. This keeps auth logic entirely out of the application.

---

## 9. Staging Deployment

```bash
# Same playbook, different inventory
ansible-playbook deploy.yml -i inventories/stg -e @releases/release-1.0.0.yml
```

The playbook itself is unchanged. Only the inventory and manifest differ. This demonstrates environment separation without code duplication.

---

## 10. Rollback Demo

```bash
# Deploy the older release (0.9.0)
ansible-playbook deploy.yml -i inventories/dev -e @releases/release-0.9.0.yml

# Verify the new version is running
kubectl rollout status deployment/podinfo -n podinfo-{ENV}
kubectl exec -n podinfo-{ENV} deploy/podinfo -- wget -qO- http://localhost:9898/version
# Should now show 6.12.0 instead of 6.11.2
...
{
  "commit": "a547e00b6cd16bf4017852205169f29f74a86452",
  "version": "6.12.0"
}
```

> **How rollback works:** Kubernetes replaces the running pod with a new one using the updated image tag from the manifest. The same playbook, a different manifest — no special rollback scripts needed. Kubernetes handles the rolling update automatically and rolls back only the changed deployment.

> **Limitation:** This approach assumes the new image is available and healthy. It does not automatically revert if the new pod crashes — you'd need readiness probes and a `kubectl rollout undo` fallback for that in production.

---

## 11. Log Viewing

```bash
# --- Main Cluster ---

# Traefik logs
kubectl logs -n traefik-{ENV} deploy/traefik -f

# Keycloak logs
kubectl logs -n keycloak-{ENV} deploy/keycloak -f

# Podinfo logs
kubectl logs -n podinfo-{ENV} deploy/podinfo -f

# All pods across namespaces at once (requires stern)
brew install stern
stern ".*" --all-namespaces

# --- Edge Server ---
ssh ubuntu@<EDGE-SERVER-IP> \
  "microk8s kubectl logs deploy/podinfo-edge -n {ENV} -f"
```

> **Reasoning:** For this sample exercise, kubectl logs is sufficient — it's built-in, requires no extra tooling, and gives real-time output. In production I would add a Loki + Grafana stack (lightweight) or ship logs to a managed service like Datadog, NewRelic or AWS CloudWatch, since kubectl logs are lost when pods are replaced.

---

## 12. Networking Verification

```bash
# Verify only Traefik has an exposed port (80/443 via LoadBalancer — Docker Desktop binds these to localhost)
kubectl get svc -A

# Expected: 
#   traefik   LoadBalancer/NodePort  80, 443
#   keycloak  ClusterIP              8080    (NOT directly reachable from outside)
#   podinfo   ClusterIP              9898    (NOT directly reachable from outside)

# Confirm Podinfo is NOT directly reachable (should time out or refuse)
curl --max-time 3 http://localhost:9898
# Expected: connection refused or timeout

# Confirm Podinfo is reachable through Traefik
curl http://podinfo.localhost/version

# Confirm main cluster can reach edge Podinfo by spining up a temporary pod and curl the edge from inside the cluster
kubectl run curl-test --image=curlimages/curl --restart=Never --rm -it -- curl http://<EDGE-SERVER-IP>:30099/version
```

> **Production hardening:** Add Kubernetes NetworkPolicies to explicitly deny all ingress to podinfo and keycloak pods except from the Traefik pod. Also use mTLS via a service mesh (Istio or Linkerd) for in-cluster pod-to-pod traffic encryption.

---

## 13. Verify Everything End-to-End

```bash
# 1. Cluster is healthy
kubectl get no
kubectl get po -A

# 2. All services are ClusterIP except Traefik
kubectl get svc -A | grep -v ClusterIP

# 3. Podinfo version matches manifest
kubectl exec -n podinfo-{ENV} deploy/podinfo -- wget -qO- http://localhost:9898/version | python3 -m json.tool

# 4. Edge is reachable from host (proxy for main cluster connectivity)
curl http://<EDGE-SERVER-IP>:30099/version

# 5. Auth is working (expect 401 without token)
curl -v http://podinfo.localhost/version
```

---

## 14. Teardown

```bash
# Uninstall all Helm releases from the main cluster
helm uninstall traefik -n traefik-{ENV}
kubectl delete namespace keycloak-{ENV} podinfo-{ENV}

# To fully reset the Docker Desktop k8s cluster (wipe all workloads):
# Docker Desktop → Settings → Kubernetes → Reset Kubernetes Cluster

# To fully remove the Docker Desktop k8s cluster:
# Docker Desktop → Settings → Kubernetes → Disable Kubernetes Cluster

# Stop and delete the Multipass VM
multipass stop edge-server
multipass delete edge-server
multipass purge
```