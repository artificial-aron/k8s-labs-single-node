# k8s-labs — Single-Node Kubernetes on Ubuntu 24.04

A single-node kubeadm cluster where the control plane and worker share one VM. Storage comes from a local disk, and ingress uses Gateway API via Envoy Gateway.

Built on **Kubernetes 1.35** on purpose, so the cluster can be upgraded later as a separate exercise.

## Environment

| Item | Value |
|---|---|
| Hostname | `k8s-labs.server.local` |
| IP | `10.10.8.210` |
| OS | Ubuntu 24.04 LTS |
| CPU / RAM | 4 vCPU / 20 GB |
| Disk 1 | 40 GB (OS) |
| Disk 2 | 80 GB (`/dev/sdb`): Persistent Volumes |

## Stack

| Layer | Component | Notes |
|---|---|---|
| Runtime | containerd 2.x (Docker apt repo) | `SystemdCgroup = true` |
| Kubernetes | kubeadm / kubelet / kubectl **v1.35** | `taints: []`, so workloads can run on the control plane |
| CNI | Calico (Tigera operator) | Pod CIDR `10.244.0.0/16` |
| Storage | LVM on `/dev/sdb` → `/mnt/k8s-data` + local-path-provisioner | Default StorageClass `local-path` |
| Ingress | Envoy Gateway (Gateway API) | NodePort `30080` |

### Why Envoy Gateway instead of ingress-nginx?
ingress-nginx was retired in March 2026 and no longer receives fixes or security patches. Upstream Kubernetes recommends Gateway API, which is the successor to Ingress. Envoy Gateway is a fully conformant Gateway API implementation.

## Repository layout

```
.
├── README.md
├── manifests/
│   ├── kubeadm-config.yaml     # Stage 3
│   ├── pvc-test.yaml           # Stage 5 gate
│   ├── envoy-gateway.yaml      # Stage 6: EnvoyProxy + GatewayClass + Gateway
│   └── whoami-test.yaml        # Stage 6 gate
└── .gitignore
```

Clone the repo onto the VM and run the stages from the repo root:

```bash
git clone <your-repo-url> k8s-labs && cd k8s-labs
sudo -i   # Stages 0–5 run as root
```

---

## Stage 0: OS prep

```bash
export NODE_IP=10.10.8.210
export NODE_FQDN=k8s-labs.server.local

hostnamectl set-hostname $NODE_FQDN
echo "$NODE_IP $NODE_FQDN k8s-labs" >> /etc/hosts

# Ubuntu installer gives root LV only ~50% of the disk; extend to full
lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv || true

# Disable swap permanently
swapoff -a
sed -i '/\sswap\s/ s/^/#/' /etc/fstab

ufw disable
timedatectl set-ntp true

cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay && modprobe br_netfilter

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

apt update && apt -y upgrade
# If apt reports "dpkg was interrupted": run `dpkg --configure -a && apt -f install -y`, then re-run this line
apt -y install apt-transport-https ca-certificates curl gpg lvm2 jq
```

**✅ Gate 0**
```bash
hostname -f; free -h | grep -i swap; df -h /; lsblk
sysctl net.ipv4.ip_forward; timedatectl | grep synchronized
```
Expected: swap is 0, root is about 38 GB, `sdb` is 80 GB, forwarding is 1, and the clock is synchronized.

---

## Stage 1: containerd 2.x

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  > /etc/apt/sources.list.d/docker.list
apt update && apt -y install containerd.io

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd && systemctl enable containerd
```

**✅ Gate 1**
```bash
containerd --version
grep -n SystemdCgroup /etc/containerd/config.toml
systemctl is-active containerd
```

---

## Stage 2: kubeadm / kubelet / kubectl 1.35

```bash
K8S_MINOR=v1.35
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" \
  > /etc/apt/sources.list.d/kubernetes.list
apt update

apt-cache madison kubeadm | head      # optional: list available patches
apt -y install kubelet kubeadm kubectl cri-tools   # cri-tools (crictl) is no longer pulled in automatically
apt-mark hold kubelet kubeadm kubectl containerd.io
systemctl enable kubelet

crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock
```

**✅ Gate 2**
```bash
kubeadm version -o short     # v1.35.x
kubectl version --client
apt-mark showhold
```

---

## Stage 3: kubeadm init

Config: [`manifests/kubeadm-config.yaml`](manifests/kubeadm-config.yaml)

```bash
kubeadm config images pull --config manifests/kubeadm-config.yaml
kubeadm init --config manifests/kubeadm-config.yaml | tee /root/kubeadm-init.log

mkdir -p /root/.kube && cp /etc/kubernetes/admin.conf /root/.kube/config
U=$(logname); mkdir -p /home/$U/.kube && cp /etc/kubernetes/admin.conf /home/$U/.kube/config && chown -R $U:$U /home/$U/.kube
echo 'source <(kubectl completion bash); alias k=kubectl; complete -o default -F __start_kubectl k' >> /home/$U/.bashrc
```

**✅ Gate 3**
```bash
kubectl get nodes -o wide            # NotReady (no CNI yet), v1.35.x
kubectl get pods -A                  # coredns Pending
kubectl describe node k8s-labs.server.local | grep -i taint   # <none>
```

---

## Stage 4: Calico CNI

```bash
CALICO_VERSION=$(curl -s https://api.github.com/repos/projectcalico/calico/releases/latest | jq -r .tag_name); echo $CALICO_VERSION
BASE=https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests

kubectl create -f $BASE/operator-crds.yaml || true
kubectl create -f $BASE/tigera-operator.yaml
curl -sLO $BASE/custom-resources.yaml
sed -i 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' custom-resources.yaml
kubectl create -f custom-resources.yaml
```

**✅ Gate 4** (allow 2–3 minutes)
```bash
watch kubectl get tigerastatus       # all AVAILABLE=True
kubectl get nodes                    # Ready
kubectl get pods -A                  # coredns Running
```

---

## Stage 5: Local storage

```bash
DISK=/dev/sdb        # confirm with lsblk first

pvcreate $DISK
vgcreate vg_k8s $DISK
lvcreate -n lv_data -l 100%FREE vg_k8s
mkfs.xfs /dev/vg_k8s/lv_data
mkdir -p /mnt/k8s-data
echo '/dev/vg_k8s/lv_data /mnt/k8s-data xfs defaults 0 0' >> /etc/fstab
systemctl daemon-reload && mount -a

LPP=$(curl -s https://api.github.com/repos/rancher/local-path-provisioner/releases/latest | jq -r .tag_name); echo $LPP
curl -sL https://raw.githubusercontent.com/rancher/local-path-provisioner/${LPP}/deploy/local-path-storage.yaml -o lpp.yaml
sed -i 's#/opt/local-path-provisioner#/mnt/k8s-data#' lpp.yaml
kubectl apply -f lpp.yaml
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**✅ Gate 5**: PVC test ([`manifests/pvc-test.yaml`](manifests/pvc-test.yaml))
```bash
df -h /mnt/k8s-data
kubectl get sc                                   # local-path (default)
kubectl apply -f manifests/pvc-test.yaml
kubectl get pvc test-pvc                         # Bound
ls -R /mnt/k8s-data                              # pvc-*/f exists
kubectl delete -f manifests/pvc-test.yaml
```

> local-path does **not** enforce PVC size limits. That is acceptable for a lab; use OpenEBS LVM LocalPV on `vg_k8s` if you need quotas or snapshots.

---

## Stage 6: Helm + Envoy Gateway

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

EG=$(curl -s https://api.github.com/repos/envoyproxy/gateway/releases/latest | jq -r .tag_name); echo $EG
helm install eg oci://docker.io/envoyproxy/gateway-helm --version $EG \
  -n envoy-gateway-system --create-namespace
kubectl wait -n envoy-gateway-system deploy/envoy-gateway --for=condition=Available --timeout=5m

kubectl apply -f manifests/envoy-gateway.yaml
```

**✅ Gate 6a**
```bash
kubectl get gatewayclass,gateway -A      # ACCEPTED / PROGRAMMED = True
kubectl get svc -n envoy-gateway-system  # NodePort 80:30080
```

**✅ Gate 6b**: end-to-end test ([`manifests/whoami-test.yaml`](manifests/whoami-test.yaml))
```bash
kubectl apply -f manifests/whoami-test.yaml
kubectl -n demo rollout status deploy/whoami
curl -s -H "Host: whoami.k8s-labs.server.local" http://10.10.8.210:30080 | head
```

From your workstation, add this line to the hosts file:
```
10.10.8.210  k8s-labs.server.local whoami.k8s-labs.server.local
```
Then browse to `http://whoami.k8s-labs.server.local:30080`.

---

## Final state

| Component | Endpoint |
|---|---|
| API server | `https://k8s-labs.server.local:6443` |
| Gateway (HTTP) | `http://*.k8s-labs.server.local:30080` |
| Default StorageClass | `local-path` → `/mnt/k8s-data` (80 GB) |

## Verified build (2026-10-15)

| Component | Version |
|---|---|
| Ubuntu | 24.04.5 LTS, kernel 6.8.0-147 |
| containerd | v2.4.2 |
| Kubernetes | v1.35.9 (kubeadm / kubelet / kubectl), crictl v1.35.0 |
| Calico | v3.33.0 (includes goldmane + whisker) |
| local-path-provisioner | latest at build time, path `/mnt/k8s-data` (100G XFS on `vg_k8s/lv_data`) |
| Helm | v3.22.0 |
| Envoy Gateway | v1.9.2 |

All gates 0–6b passed. Actual VM disks: sda 50G (OS), sdb 100G (PV).

## Do not commit
`/etc/kubernetes/admin.conf`, `~/.kube/config`, and `kubeadm-init.log` contain cluster credentials and join tokens. The `.gitignore` in this repo blocks the common ones.
