---
title: K3S - Lightweight Kubernetes
date: 2026-10-17 11:00:00 +0300
categories: k8s
tags: kubernetes                    # Tag names should always be lowercase
image:
  path: /assets/img/headers/k3s-tutorial.webp
  lqip: data:image/webp;base64,UklGRqgAAABXRUJQVlA4IJwAAAAwBACdASoUAAsAPpE4l0eloyIhMAgAsBIJZACw7YvW327yVtV8OJWkRYAA/vPO0hS6pJqbKdacBIShckH3kkm95sLI2GVdf3Udz6ryJzIZtSd0EH1lItaFlA1xiOHUEev9xOh//nYb5SnVq637eAZ6/r/6bKWGLumpbBvxP/cSdMOYVAJJB9Wm/E/YeN2z/ZGoFa9/cdVTGo1AAAA=
---

# What is K3S ?

**K3S** is a lightweight, fully compliant, CNCF-certified Kubernetes distribution originally developed by Rancher Labs (now part of SUSE). It packages Kubernetes into a single binary (under ~100MB) by stripping out legacy, alpha, and in-tree cloud provider code, and replacing heavy defaults with lightweight alternatives.

### Pros & Cons

| Category           | Pros              | Cons              |
| :----------------  | :---------------- | :---------------- |
| **Resource Usage**        |   Extremely low footprint (~512 MB RAM, minimal CPU); runs seamlessly on ARM, Raspberry Pi, and edge devices.  | Not designed or optimized for massive, hyperscale enterprise data centers (thousands of large nodes). |
| **Installation & Setup** | Packaged as a single lightweight binary (<100 MB); can be installed in under a minute with one command. | Pre-bundled components (Traefik, Flannel) must be explicitly disabled via flags if you prefer alternatives (e.g., Cilium, NGINX). |
| **K8s Compatibility** | 100% API-compliant with standard Kubernetes manifests, Helm charts, and tools (kubectl, ArgoCD, etc.) | In-tree cloud provider code is removed; native integrations with cloud load balancers/disks require manual CCM/CSI setup. |
| **Datastore** | Highly flexible: supports lightweight **SQLite** for single-node, external SQL (Postgres/MySQL), or **etcd**. | Production High Availability (HA) requires manually provisioning and maintaining external databases or etcd clusters. |
| **Networking & Ingress** | "Batteries-included": comes pre-configured with Flannel CNI, Traefik Ingress, and Klipper Load Balancer. | Flannel provides basic networking; advanced features (like eBPF or complex service meshes) require replacing the default CNI. |
| **Storage** | Includes `local-path-provisioner` out-of-the-box for instant local persistent volume claims. | Storage is bound to the local host (`hostPath`); multi-node shared/replicated storage requires extra tooling (e.g., Longhorn, Rook-Ceph). |
| **CI/CD & Development** | Fast boot times make it ideal for CI/CD test runners and local multi-node development (via `k3d`). | Lacks built-in enterprise multi-tenancy, auditing, or compliance tools found in heavyweight platforms (like OpenShift). |


### Prerequisites
- 3 Linux VMs (Ubuntu/Debian recommended)
- Local network connectivity through SSH (TCP port 22) and Kubernetes API server (TCP port 6443)

### Step 1: Setting up the Control Plane Node
- SSH into one of your VMs that you will decide to make it controller node.
- Install k3s server:
```bash
curl -sfL https://get.k3s.io | sh -
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

>Note: If you prefer using your own Ingress Controller (e.g., NGINX Ingress or Cilium) instead of default components, pass flags during installation:
>```bash
>curl -sfL https://get.k3s.io | sh -s - --disable traefik --disable servicelb
>```
{: .prompt-info }
<!-- markdownlint-restore -->

- After the installation is complete, verify the control node status:
```bash
sudo k3s kubectl get nodes
```
- Now that the control node is up and running you need to join the rest of the VMs to the cluster. But to do that you will need to retrive the control node token:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> Please note: this will display the full content of the token
>```bash
>ABCDef...::server:abcdef...
>```
> Please copy the full token
{: .prompt-warning }

<!-- markdownlint-restore -->

### Step 2: Joining the Worker Nodes
- Run the installer on the other workers using the control node IP and token:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<CONTROL_NODE_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```

### Step 3: Verify the Cluster
- Back on the control node, check the cluster nodes:
```bash
sudo k3s kubectl get nodes
```

### Step 4: Accessing the Cluster Locally

To manage the cluster from your local machine using standard `kubectl`:

- Copy `/etc/rancher/k3s/k3s.yaml` from the control plane node to your local `~/.kube/config`.
```bash
ssh -t <username>@<control-plane-ip> "sudo cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/config
```
- Edit the file on your local machine and replace `127.0.0.1` with your control plane IP:
```bash
sed -i 's/127.0.0.1/<CONTROL_NODE_IP>/g' ~/.kube/config
```
- Verify local connectivity:
```bash
kubectl get nodes
```

#### Optional: Apply labels to worker nodes
- Apply the `worker` label to the worker node:
```bash
kubectl label node <YOUR_WORKER_NODE> node-role.kubernetes.io/worker=worker
```
- Verify the label applied:
```bash
kubectl describe nodes <YOUR_WORKER_NODE>
```

### Step 5: Deploying a Demo Workload

Verify cluster functionality by deploying a simple NGINX application:

- Create the new `demo` namespace:
```bash
kubectl create namespace demo
```

#### Option A: Expose the service under LoadBalancer (Klipper)

- Create a deployment and expose it via a LoadBalancer service (using Klipper):
```bash
kubectl create deployment nginx-demo --image=nginx -n demo
kubectl scale deployment nginx-demo --replicas=3 -n demo
kubectl expose deployment nginx-demo --port=8080 --target-port=80 --type=LoadBalancer -n demo
```
- Check pod distribution across nodes and get the external IP:
```bash
kubectl get pods,svc -n demo
```
- Test connectivity to the NGINX application:
```bash
curl http://<EXTERNAL_IP_OR_NODE_IP>:8080
```

#### Option B: Expose the service under Traefik
- Keep the service under ClusterIP:
```bash
kubectl expose deployment nginx-demo --port=80 -n demo
```
- Create the Ingress for nginx:
```bash
kubectl create ingress nginx-ingress --rule="/=nginx-demo:80" -n demo
```
- Check external IP form ingress:
```bash
kubectl get ingress -n demo
```
- Test connectivity to the NGINX application
```bash
curl http://<EXTERNAL_IP_FROM_INGRESS>
```

### Cleanup the deployment:
```bash
kubectl delete namespace demo
```

## Conclusion

In this tutorial, you set up a multi-node Kubernetes cluster using **k3s** with a single control plane and two worker nodes, configured standard `kubectl` access from your local workstation, and deployed a test workload. 

Thanks to its lightweight footprint, fast installation time, and full Kubernetes API compatibility, k3s is an exceptional choice for edge computing, IoT, CI/CD pipelines, and local multi-node labs. From here, you can explore adding persistent storage with Longhorn, installing custom Ingress controllers, or managing your cluster declaratively with GitOps tools like ArgoCD or Flux.
