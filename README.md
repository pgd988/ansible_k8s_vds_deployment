# Bare-Metal RKE2 & Rancher Management Cluster Deployment with Ansible

These Ansible playbooks automate the deployment of a production-grade, bare-metal Kubernetes cluster using **RKE2 (Rancher Kubernetes Engine 2)** on Ubuntu VDS/VMs. By default, the cluster is configured with **Cilium CNI and WireGuard node-to-node encryption**, deploys the **Rancher Management Server UI** via Helm, and offers flexible external ingress options via **MetalLB (`IPAddressPool` + `L2Advertisement`)** and/or **Cloudflare Tunnel (`cloudflared`)**.

## Key Features

- **RKE2 (Rancher Kubernetes Engine 2)**: Rancher's CNCF-certified, FIPS-compliant Kubernetes distribution designed for data center and bare-metal environments (`rke2-server` on master nodes, `rke2-agent` on worker nodes).
- **Cilium CNI + WireGuard Encryption**: Out-of-the-box high performance eBPF networking (`rke2_cni: "cilium"`) configured with transparent node-to-node WireGuard encryption via `HelmChartConfig`.
- **Rancher Management Server UI**: Optional automated deployment of `cert-manager` and the multi-cluster Rancher Web UI dashboard (`rancher/rancher` chart in `cattle-system`). Toggled via `install_rancher_server: true/false`.
- **MetalLB Bare-Metal LoadBalancer**: Out-of-the-box Layer 2 LoadBalancer IP assignment (`metallb.yml`). Automatically deploys MetalLB via Helm, creates an `IPAddressPool` (`metallb_ip_range`), and advertises VIPs across your VDS/bare-metal subnet for any `LoadBalancer` service. Toggled via `install_metallb: true/false`.
- **Cloudflare Tunnel (`cloudflared`)**: Alternative or complementary Zero Trust ingress traffic tunneling using the `helmforge/cloudflared` Helm chart and native Cloudflare v4 REST API automation (`cloudflare-tunnel.yml`). Automatically configures ingress routing, creates DNS CNAME records, and deploys high-availability `cloudflared` replicas to `kube-system`. Toggled via `install_cloudflare_tunnel: true/false`.

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
   rancher_hostname: "rancher.10.10.10.253.nip.io"
   rancher_bootstrap_password: "adminPassword123!"

   # Ingress & External Traffic Controller Selection
   # Toggle between MetalLB (Bare-Metal L2 LoadBalancer) and Cloudflare Tunnel (Zero-Trust Edge)
   install_metallb: true
   install_cloudflare_tunnel: false

   # MetalLB LoadBalancer Configuration
   metallb_chart_version: "0.14.8"
   metallb_ip_range: "10.10.10.240-10.10.10.250"

   # Cloudflare Tunnel & Ingress Routing Configuration (Used when install_cloudflare_tunnel: true)
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

Run the master orchestration playbook (`site.yml`) to initialize the control plane node, join all worker nodes, deploy the Rancher Management Server, and set up your selected ingress controller (`MetalLB` or `Cloudflare Tunnel`):

```bash
ansible-playbook -i inventory.ini site.yml
```

---

### Option 2: Step-by-Step Deployment

#### 1. Configure Control Plane, RKE2 Server, Rancher & Ingress Controllers

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
- Deploys MetalLB L2 `IPAddressPool` (`metallb_ip_range`) and `L2Advertisement` via `metallb.yml` (when `install_metallb: true`).
- Deploys Cloudflare Tunnel API routes, DNS CNAMEs, and `cloudflared` Helm chart via `cloudflare-tunnel.yml` (when `install_cloudflare_tunnel: true`).

#### 2. Join Worker Nodes

```bash
ansible-playbook -i inventory.ini worker-node.yml
```

This tasks:
- Prepares OS (`common-tasks.yml`) and loads required modules (`wireguard`, etc.).
- Reads `rke2_node_token.txt` and configures `/etc/rancher/rke2/config.yaml` with the server URL (`https://<apiserver_advertise_address>:9345`).
- Starts `rke2-agent.service` on all `workers` hosts.

---

## Toggling External Traffic Controllers (MetalLB vs Cloudflare Tunnel)

You can easily switch how external traffic enters your Kubernetes cluster by adjusting two variables inside `vars.yml`:

### 1. Bare-Metal Local/VDS LoadBalancer (MetalLB)
Best suited when your VDS boxes or bare-metal servers are on a private/public network with a pool of available IP addresses that can be assigned directly to services.
```yaml
install_metallb: true
install_cloudflare_tunnel: false
metallb_ip_range: "10.10.10.240-10.10.10.250" # Set to your subnet's available IP pool
```
Check assigned VIPs using `kubectl`:
```bash
kubectl get svc -A -o wide | grep LoadBalancer
```

### 2. Zero-Trust Edge Tunnel (Cloudflare Tunnel)
Best suited when your servers are behind NAT or firewalls, or when you want Cloudflare to handle SSL termination, DDoS protection, and routing without exposing public inbound ports.
```yaml
install_metallb: false
install_cloudflare_tunnel: true
```
Check `cloudflared` pod health using `kubectl`:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=cloudflared
```

### 3. Both Simultaneously
If you require internal local LoadBalancer VIPs inside your VDS subnet while simultaneously routing public traffic through Cloudflare:
```yaml
install_metallb: true
install_cloudflare_tunnel: true
```

---

## Accessing the Cluster & Rancher UI

### Kubernetes CLI Access (`kubectl`)

On the control plane (`master`) node, `kubectl` is automatically symlinked to `/usr/local/bin/kubectl` and configured for both `root` and `ubuntu` users (`~/.kube/config -> /etc/rancher/rke2/rke2.yaml`).

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

### Rancher Management UI Access

When `install_rancher_server: true` is enabled, check that all pods in `cattle-system` are `Running`:

```bash
kubectl get pods -n cattle-system
```

Access the Rancher web interface securely via your configured hostname:
- **URL**: `https://<rancher_hostname>`
- **Login Password**: The password defined in `rancher_bootstrap_password` inside `vars.yml`.

---

## Customizing Architecture & Toggles

### Pure RKE2 Mode (No Rancher UI, MetalLB, or Cloudflare)
If you need only the underlying RKE2 + Cilium + WireGuard Kubernetes cluster without any add-ons, set all toggles to `false` in `vars.yml`:

```yaml
install_rancher_server: false
install_metallb: false
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
