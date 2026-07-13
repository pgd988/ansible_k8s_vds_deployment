# Bare-Metal RKE2 & Rancher Management Cluster Deployment with Ansible

These Ansible playbooks automate the deployment of a production-grade, bare-metal Kubernetes cluster using **RKE2 (Rancher Kubernetes Engine 2)** on Ubuntu VDS/VMs. By default, the cluster is configured with **Cilium CNI and WireGuard node-to-node encryption**, deploys the **Rancher Management Server UI** via Helm, and automates ingress routing via **Cloudflare Tunnel (`cloudflared`)**.

## Key Features

- **RKE2 (Rancher Kubernetes Engine 2)**: Rancher's CNCF-certified, FIPS-compliant Kubernetes distribution designed for data center and bare-metal environments (`rke2-server` on master nodes, `rke2-agent` on worker nodes).
- **Cilium CNI + WireGuard Encryption**: Out-of-the-box high performance eBPF networking (`rke2_cni: "cilium"`) configured with transparent node-to-node WireGuard encryption via `HelmChartConfig`.
- **Rancher Management Server UI**: Optional automated deployment of `cert-manager` and the multi-cluster Rancher Web UI dashboard (`rancher/rancher` chart in `cattle-system`). Toggled via `install_rancher_server: true/false`.
- **Cloudflare Tunnel (`cloudflared`)**: Automated Zero Trust ingress traffic tunneling using the `helmforge/cloudflared` Helm chart and native Cloudflare v4 REST API automation (`cloudflare-tunnel.yml`). Automatically configures ingress routing, creates DNS CNAME records, and deploys high-availability `cloudflared` replicas to `kube-system`.

## Prerequisites

1. **Ansible** installed on your control machine (`ansible-core` >= 2.12).
2. **SSH Access** to target Ubuntu VDS/VMs with `sudo` / root privileges.
3. Update `inventory.ini` with target host IPs/domains and usernames:
   ```ini
   [control_plane]
   master ansible_host=10.10.10.253 ansible_user=ubuntu

   [workers]
   worker1 ansible_host=10.10.10.254 ansible_user=ubuntu
   worker2 ansible_host=10.10.10.255 ansible_user=ubuntu
   ```
4. Configure `vars.yml` according to your environment and desired toggles:
   ```yaml
   rke2_channel: "stable"
   rke2_cni: "cilium"
   rke2_cilium_wireguard: true

   apiserver_advertise_address: "10.10.10.253"
   pod_network_cidr: "10.42.0.0/16"
   service_network_cidr: "10.43.0.0/16"
   cluster_domain: "cluster.local"

   # Rancher Management Server UI toggle and configuration
   install_rancher_server: true
   rancher_hostname: "rancher.yourcompany.com"
   rancher_bootstrap_password: "adminPassword123!"

   # Cloudflare Tunnel & Ingress Routing Configuration
   install_cloudflare_tunnel: true
   cloudflare_k8s_namespace: "kube-system"
   cloudflare_domain_name: "yourcompany.com"
   cloudflare_account_id: "your-cf-account-id"
   cloudflare_api_token: "your-cf-api-token"
   cloudflare_tunnel_name: "prod-rke2-tunnel"
   cloudflare_tunnel_token: "" # Optional: populate if tunnel is pre-created outside API
   cloudflare_replica_count: 3
   cloudflare_cname_records:
     - "api"
     - "rancher"
   cloudflare_ingress_routes:
     - hostname: "api.{{ cloudflare_domain_name }}"
       service: "http://my-api-service.default.svc.cluster.local:80"
     - hostname: "rancher.{{ cloudflare_domain_name }}"
       service: "https://rancher.cattle-system.svc.cluster.local:443"
       originRequest:
         noTLSVerify: true
     - service: "http_status:404"
   ```

---

## Usage

### Option 1: Full Automated Deployment (Recommended)

Run the master orchestration playbook (`site.yml`) to initialize the control plane node, join all worker nodes, deploy the Rancher Management Server, and set up Cloudflare Tunnel ingress routing:

```bash
ansible-playbook -i inventory.ini site.yml
```

---

### Option 2: Step-by-Step Deployment

#### 1. Configure Control Plane, RKE2 Server, Rancher & Cloudflare Tunnel

```bash
ansible-playbook -i inventory.ini control-plane-node.yml
```

This tasks:
- Prepares OS (`common-tasks.yml`), loads `wireguard` / `overlay` / `br_netfilter` modules, and sets sysctls.
- Deploys `HelmChartConfig` enabling Cilium WireGuard encryption (`/var/lib/rancher/rke2/server/manifests/rke2-cilium-config.yaml`).
- Configures `/etc/rancher/rke2/config.yaml` and starts `rke2-server.service`.
- Extracts the RKE2 cluster node token to `rke2_node_token.txt` locally.
- Configures `kubectl` and `crictl` symlinks on the master node (`/usr/local/bin/kubectl`).
- Deploys `cert-manager` and the Rancher Management Server (when `install_rancher_server: true`).
- Deploys Cloudflare Tunnel API routes, DNS CNAMEs, and `cloudflared` Helm chart (when `install_cloudflare_tunnel: true`).

#### 2. Join Worker Nodes

```bash
ansible-playbook -i inventory.ini worker-node.yml
```

This tasks:
- Prepares OS (`common-tasks.yml`) and loads required modules (`wireguard`, etc.).
- Reads `rke2_node_token.txt` and configures `/etc/rancher/rke2/config.yaml` with the server URL (`https://<apiserver_advertise_address>:9345`).
- Starts `rke2-agent.service` on all `workers` hosts.

---

## Accessing the Cluster, Rancher UI & Cloudflare Tunnel

### Kubernetes CLI Access (`kubectl`)

On the control plane (`master`) node, `kubectl` is automatically symlinked to `/usr/local/bin/kubectl` and configured for both `root` and `ubuntu` users (`~/.kube/config -> /etc/rancher/rke2/rke2.yaml`).

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

### Rancher Management UI & Cloudflare Ingress Access

When `install_rancher_server: true` and `install_cloudflare_tunnel: true` are enabled, check that all pods in `cattle-system` and `kube-system` are `Running`:

```bash
kubectl get pods -n cattle-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=cloudflared
```

Access the Rancher web interface securely via your configured public Cloudflare hostname (no open firewall ports required):
- **URL**: `https://rancher.<cloudflare_domain_name>`
- **Login Password**: The password defined in `rancher_bootstrap_password` inside `vars.yml`.

---

## Customizing Architecture & Toggles

### Pure RKE2 Mode (No Rancher UI or Cloudflare)
If you need only the underlying RKE2 + Cilium + WireGuard Kubernetes cluster without installing the Rancher UI chart or Cloudflare Tunnel, set the toggles to `false` in `vars.yml`:

```yaml
install_rancher_server: false
install_cloudflare_tunnel: false
```

### Using a Pre-Created Cloudflare Tunnel Token
If you created a tunnel manually via the Cloudflare Zero Trust Dashboard and prefer to skip automated API tunnel/DNS creation:
1. Copy the tunnel token string.
2. Set `cloudflare_tunnel_token` in `vars.yml`:
   ```yaml
   cloudflare_tunnel_token: "eyJh..."
   ```
Ansible will skip API calls and directly deploy the `cloudflared` Helm chart using your token.

### Checking WireGuard Encryption Status
Verify node-to-node WireGuard encryption by checking Cilium status or inspecting the WireGuard interface on any node:

```bash
kubectl -n kube-system exec -it ds/cilium -- cilium status --all | grep WireGuard
ip link show cilium_wg0
```
