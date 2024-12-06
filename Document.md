# 4KUBE: Déploiement d'une Application Distribuée sur Kubernetes

## Contexte
Ce projet consiste à déployer une application distribuée permettant de suivre en temps réel une flotte de véhicules effectuant des livraisons

## Pré-requis
- Trois machines (virtuelles ou physiques) avec les adresses IP définies :
  - **Nœud master** : `192.168.1.10`
  - **Nœuds worker** : `192.168.1.11` et `192.168.1.12`
- **Système d'exploitation** : Debian.
- VMware pour gérer les machines virtuelles.

---

## Étapes de configuration

### 1. Mise en place du cluster Kubernetes
#### Désactivation du swap
Désactivez le swap sur toutes les machines :
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### Mise à jour du fichier `/etc/hosts`
Ajoutez les lignes suivantes sur chaque machine :
```plaintext
192.168.1.10 k8s-master
192.168.1.11 k8s-node1
192.168.1.12 k8s-node2
```

#### Installation de Containerd
Configurez les modules du noyau :
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```
Installez Containerd :
```bash
sudo apt update
sudo apt -y install containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```
Configurez Containerd pour Kubernetes :
```plaintext
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
Redémarrez Containerd :
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

#### Installation de Kubernetes (Kubeadm, Kubelet, Kubectl)
```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2. Initialisation du nœud maître
Initialisez le cluster :
```bash
sudo kubeadm init --control-plane-endpoint=k8s-master
```
Configurez `kubectl` :
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Vérifiez l'état du cluster :
```bash
kubectl get nodes
kubectl cluster-info
```

### 3. Ajout des nœuds travailleurs
Sur chaque nœud travailleur, exécutez la commande générée lors de l'initialisation du maître (exemple) :
```bash
sudo kubeadm join k8s-master:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```
Vérifiez que les nœuds sont ajoutés :
```bash
kubectl get nodes
```

### 4. Installation de Calico
Installez Calico pour la gestion des politiques réseau :
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```
Autorisez les ports nécessaires :
```bash
sudo ufw allow 179/tcp
sudo ufw allow 4789/udp
sudo ufw allow 51820/udp
sudo ufw allow 51821/udp
sudo ufw reload
```
Attendez que les nœuds soient en état **Ready** :
```bash
kubectl get nodes
```

---

## Déploiement de l'application distribuée

### 1. Notes importantes
- Changez `SPRING_PROFILES_ACTIVE` en `production-microservice`.
- Changez le tag de l'image `supinfo4kube/web-app` à `1.0.0`.
- Redémarrez `fleetman-queue` si les positions n'apparaissent pas.

### 2. Vérification
Assurez-vous que tous les pods sont en état `Running` :
```bash
kubectl get pods
```
Testez l'application à l'adresse :
```plaintext
http://192.168.249.129:30080
```
