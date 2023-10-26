- [Setup nodes](#setup-nodes)
  - [Install general dependencies](#install-general-dependencies)
  - [Enable iptables bridged traffic on the node](#enable-iptables-bridged-traffic-on-the-node)
  - [Ensure swap is disabled](#ensure-swap-is-disabled)
  - [Install containerd](#install-containerd)
  - [Install runc](#install-runc)
  - [Install CNI (Container Network Interface) network plugins](#install-cni-container-network-interface-network-plugins)
  - [Install kubeadm, kubelet \& kubectl](#install-kubeadm-kubelet--kubectl)
- [Configure Kubernetes Control Plane](#configure-kubernetes-control-plane)
  - [Create cluster using kubeadm](#create-cluster-using-kubeadm)
  - [OPTIONAL - Untaint node to allow master node to accept pods](#optional---untaint-node-to-allow-master-node-to-accept-pods)
  - [Install Helm](#install-helm)
  - [Install Cilium as CNI plugin (Container Network Interface)](#install-cilium-as-cni-plugin-container-network-interface)
  - [OPTIONAL - Install Cilium CLI](#optional---install-cilium-cli)
  - [Install Prometheus CRD](#install-prometheus-crd)
  - [Install Sealed Secrets](#install-sealed-secrets)
- [Configure worker node](#configure-worker-node)
- [Setup Argo CD](#setup-argo-cd)
  - [Install Argo CD](#install-argo-cd)
  - [Install Argo CD CLI](#install-argo-cd-cli)
  - [Setup ArgoCD](#setup-argocd)
- [Kubeseal](#kubeseal)
  - [Restore key in new cluster](#restore-key-in-new-cluster)
  - [Install Kubeseal client tool to encrypt secrets](#install-kubeseal-client-tool-to-encrypt-secrets)
    - [Create secret.yaml](#create-secretyaml)
    - [Convert secret.yaml to sealed-secret.yaml](#convert-secretyaml-to-sealed-secretyaml)
- [Upgrade Kubernetes cluster](#upgrade-kubernetes-cluster)
  - [Upgrade control plane nodes](#upgrade-control-plane-nodes)
  - [Upgrade worker nodes](#upgrade-worker-nodes)


# Setup nodes
## Install general dependencies
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

## Enable iptables bridged traffic on the node
```
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_watches=204800" | sudo tee -a /etc/sysctl.conf

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Ensure swap is disabled
```
sudo swapoff -a
sudo sed -i -e '/swap/d' /etc/fstab
```

## Install containerd
https://github.com/containerd/containerd
```
CONTAINERD_VERSION=1.7.7
PROCESSOR_ARCH=$(dpkg --print-architecture)

sudo mkdir /etc/containerd
cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true
EOF

curl -fsSLo containerd-${CONTAINERD_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz \
  https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz

sudo tar Cxzvf /usr/local containerd-${CONTAINERD_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz
sudo rm containerd-${CONTAINERD_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz

# Start containerd via systemd
sudo curl -fsSLo /etc/systemd/system/containerd.service \
  https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

## Install runc
https://github.com/opencontainers/runc
```
RUNC_VERSION=1.1.9
PROCESSOR_ARCH=$(dpkg --print-architecture)

curl -fsSLo runc.${PROCESSOR_ARCH} \
  https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.${PROCESSOR_ARCH}

sudo install -m 755 runc.${PROCESSOR_ARCH} /usr/local/sbin/runc
rm runc.${PROCESSOR_ARCH}
```

## Install CNI (Container Network Interface) network plugins
https://github.com/containernetworking/plugins
```
CNI_VERSION=1.3.0
PROCESSOR_ARCH=$(dpkg --print-architecture)

curl -fsSLo cni-plugins-linux-${PROCESSOR_ARCH}-v${CNI_VERSION}.tgz \
  https://github.com/containernetworking/plugins/releases/download/v${CNI_VERSION}/cni-plugins-linux-${PROCESSOR_ARCH}-v${CNI_VERSION}.tgz

sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-${PROCESSOR_ARCH}-v${CNI_VERSION}.tgz

sudo rm cni-plugins-linux-${PROCESSOR_ARCH}-v${CNI_VERSION}.tgz
```

## Install kubeadm, kubelet & kubectl
```
KUBERNETES_VERSION=1.27.7

sudo mkdir -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=${KUBERNETES_VERSION}-* kubeadm=${KUBERNETES_VERSION}-* kubectl=${KUBERNETES_VERSION}-*
sudo apt-mark hold kubelet kubeadm kubectl
```

---

# Configure Kubernetes Control Plane
Configure the control plane on the master node only.
## Create cluster using kubeadm
```
sudo kubeadm init --config config.yaml --upload-certs

sudo mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo chmod go-r ~/.kube/config
```

## OPTIONAL - Untaint node to allow master node to accept pods
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Install Helm
https://github.com/helm/helm
```
HELM_VERSION=3.13.0
PROCESSOR_ARCH=$(dpkg --print-architecture)

curl -fsSLo helm-v${HELM_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz \
  https://get.helm.sh/helm-v${HELM_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz
sudo tar xzvf helm-v${HELM_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz linux-${PROCESSOR_ARCH}/helm
sudo mv linux-${PROCESSOR_ARCH}/helm /usr/local/bin/
sudo rm linux-${PROCESSOR_ARCH} -r
sudo rm helm-v${HELM_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz
```

## Install Cilium as CNI plugin (Container Network Interface)
https://github.com/cilium/cilium
```
API_SERVER_IP=cloud.davydehaas.dev
API_SERVER_PORT=6443
CILIUM_HELM_VERSION=1.14.2

helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium \
  --version ${CILIUM_HELM_VERSION} \
  --namespace kube-system \
  --set operator.replicas=1 \
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT}
```

## OPTIONAL - Install Cilium CLI
https://github.com/cilium/cilium-cli
```
CILIUM_CLI_VERSION=0.15.10
CLI_ARCH=$(dpkg --print-architecture)

if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/v${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

## Install Prometheus CRD
https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-operator-crds
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-crds prometheus-community/prometheus-operator-crds
```

## Install Sealed Secrets
https://github.com/bitnami-labs/sealed-secrets/tree/main/helm/sealed-secrets
```
SEALED_SECRETS_VERSION=2.13.0
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --version ${SEALED_SECRETS_VERSION} \
  --namespace kube-system \
  --set-string fullnameOverride=sealed-secrets-controller \
```

---

# Configure worker node
Generate kubeadm join command to use on worker node.
```
kubeadm token create --print-join-command
```

---

# Setup Argo CD

## Install Argo CD
https://github.com/argoproj/argo-helm
```
ARGOCD_HELM_VERSION=5.46.7
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --version ${ARGOCD_HELM_VERSION} \
  -n argocd --create-namespace
```

## Install Argo CD CLI
https://github.com/argoproj/argo-cd
```
ARGOCD_CLI_VERSION=v2.8.4
PROCESSOR_ARCH=$(dpkg --print-architecture)

curl -sSL -o argocd-linux-${PROCESSOR_ARCH} https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_CLI_VERSION}/argocd-linux-${PROCESSOR_ARCH}
sudo install -m 555 argocd-linux-${PROCESSOR_ARCH} /usr/local/bin/argocd
rm argocd-linux-${PROCESSOR_ARCH}
```

```
kubectl config set-context --current --namespace=argocd
argocd login --core
argocd proj create no-sync --dest '*,*' --src '*' --allow-cluster-resource '*/*'
argocd app create argocd \
  --repo https://github.com/davydehaas98/gitops.git --path core \
  --dest-server https://kubernetes.default.svc --dest-namespace argocd
argocd app sync argocd
argocd app sync metallb \
  ingress-nginx \
  sealed-secrets
```

Set cloudflare-api-token sealed secret.

```
argocd app sync external-dns
argocd app sync cert-manager
```

## Setup ArgoCD
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc argocd-server -n argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Log in to ArgoCD.

---

# Kubeseal
> Kubeseal makes it possible to store secrets encrypted on Git that only the cluster itself can decrypt.

## Restore key in new cluster
```
kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > key.yaml
```
Update `tls.key` and `tls.crt` in key.yaml
```
kubectl apply -f key.yaml
sudo rm key.yaml
kubectl -n kube-system rollout restart deployment sealed-secrets-controller
```

## Install Kubeseal client tool to encrypt secrets
https://github.com/bitnami-labs/sealed-secrets
```
KUBESEAL_VERSION=0.24.1
PROCESSOR_ARCH=$(dpkg --print-architecture)

wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz kubeseal
rm kubeseal-${KUBESEAL_VERSION}-linux-${PROCESSOR_ARCH}.tar.gz

sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

### Create secret.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: default
type: Opaque
stringData:
  username: admin
  password: p4ssw0rd
```

### Convert secret.yaml to sealed-secret.yaml
```
cat secret.yaml | kubeseal --format yaml > sealed-secret.yaml
```

---

# Upgrade Kubernetes cluster
Find the Kubernetes version in the apt-cache madison list you want to upgrade to.)
```
sudo apt update
sudo apt-cache madison kubeadm | tac
```

## Upgrade control plane nodes
Install kubeadm:
https://kubernetes.io/releases/
```
KUBERNETES_VERSION=1.27.7
sudo apt update
sudo apt-mark unhold kubeadm kubectl kubelet
sudo apt-get install -y kubeadm=${KUBERNETES_VERSION}-* kubelet=${KUBERNETES_VERSION}-* kubectl=${KUBERNETES_VERSION}-*
sudo apt-mark hold kubeadm kubectl kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## Upgrade worker nodes
```
KUBERNETES_VERSION=1.27.7
NODE_NAME=instance-20230720-1942

kubectl cordon ${NODE_NAME}
kubectl drain ${NODE_NAME} --ignore-daemonsets --delete-emptydir-data
sudo apt update
sudo apt-mark unhold kubeadm kubectl kubelet
sudo apt-get install -y kubeadm=${KUBERNETES_VERSION}-* kubelet=${KUBERNETES_VERSION}-* kubectl=${KUBERNETES_VERSION}-*
sudo apt-mark hold kubeadm kubectl kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon ${NODE_NAME}
```
