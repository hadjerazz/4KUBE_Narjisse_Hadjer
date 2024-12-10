# 4KUBE: Déploiement d'une Application Distribuée sur Kubernetes

## Contexte
Ce projet consiste à déployer une application  distribuée en local  et sur un Cluster Kubernetes  permettant de suivre en temps réel une flotte de véhicules effectuant des livraisons

## Pré-requis
   **Déployer l'application en local:**
   * Système d'exploitation: **Windows ou autres**
   * Outils nécessaires: Docker

   **Sur un Cluster Kubernetes:**
   * Système d'exploitation: **Debian**
   * Outils nécessaires: Kubeadm,Kubectl,Kubelet,Containerd
   * Trois machines virtuelles avec les adresses IP définies :
        
        Nœud master : `192.168.249.129`

        Nœuds worker : `192.168.249.131` et `192.168.249.133`
   
   * VMware pour gérer les machines virtuelles.

---
## Création des manifestes Kubernetes ##
Création de fichiers YAML pour déployer l'application:
 * fleetman-api-gateway.yaml
 * fleetman-mongodb.yaml
 * fleetman-position-simulator.yaml
 * fleetman-position-tracker.yaml
 * fleetman-queue.yaml
 * fleetman-webapp.yaml

-Tous ces fichiers contiennent des deployments et services.

-Le fichier `fleetman-mongodb.yaml` contient de plus un pv et pvc.

## Accès à l'application en local : ##
Après avoir appliquer les déployments et vérifier les status de pods créés, on peut accéder à l'application via cette adresse:
   http://localhost:30080

---
## Étapes de configuration du Cluster Kubernetes ##

### 1. Mise en place du cluster Kubernetes
#### Désactivation du swap
Désactivation du swap sur toutes les machines,car Kubernetes gère directement la mémoire et ne fonctionne pas correctement avec le swap activé:
```
bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### Mise à jour du fichier `/etc/hosts`
Ajout des lignes suivantes sur chaque machine, pour établir la communication entre les nœuds :
```plaintext
192.168.249.129 k8s-master
192.168.249.131 k8s-node1
192.168.249.133 k8s-node2
```

#### Installation de Containerd
 Containerd est un runtime de conteneurs qui exécute les conteneurs dans Kubernetes.
```bash
$ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

$ sudo modprobe overlay
$ sudo modprobe br_netfilter

sudo apt update
sudo apt -y install containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```
Configuration de Containerd: Une recommandation de Kubernetes pour une gestion optimale des ressources système:
```plaintext
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
Redémarrer Containerd :
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

#### Installation de Kubernetes (Kubeadm, Kubelet, Kubectl):
Kubeadm: Initialise et configure le cluster Kubernetes.

Kubelet : Agent responsable d'exécuter les conteneurs sur chaque nœud.

Kubectl : Outil de commande pour interagir avec le cluster Kubernetes.
```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2. Initialisation du nœud maître
Initialiser le cluster :
```bash
sudo kubeadm init --control-plane-endpoint=k8s-master
```
Configuration de `kubectl` :
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Vérifier l'état du cluster :
```bash
kubectl get nodes
```

### 3. Ajout des nœuds travailleurs
Sur chaque nœud travailleur, on exécute la commande générée lors de l'initialisation du maître:
```bash
sudo kubeadm join k8s-master:6443 --token io1g42.507y49eo4uwqho72 \
        --discovery-token-ca-cert-hash sha256:f3f080c5b55d24b9402c22ded56b1c3787f6538185f00c00d71cf7710893ce79

```
Vérifier que les nœuds sont ajoutés :
```bash
kubectl get nodes
```

### 4. Installation de Calico
 pour la gestion des politiques réseau :
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```
### 5. Connexion à GitHub 
Installer Git, puis cloner le dépôt de notre projet à l'aide de la commande git clone. Cela permettra de déployer notre cluster sur le nœud principal k8s-master.

```bash
git clone https://github.com/hadjerazz/4KUBE_Narjisse_Hadjer.git
```
## Accès à l'application sur le Cluster : ##
Après avoir appliquer les déployments et vérifier les status de pods créés, on peut accéder à l'application via cette adresse:
   http://192.168.249.129:30080/

**Remarque**: 192.168.249.129 est l'adresse du noeud master récupérée par la VM en executant la commande: 
```bash
ip a
```

### Preuve de déployment:
* Le statut des nœuds et les pods déployés sur chacun d'eux :


![alt text](<Capture d'écran 2024-12-10 093410.png>)
* Une capture d'écran de l'application accessible à l'adresse : http://192.168.249.129:30080/:
![alt text](<Capture d'écran 2024-12-10 093226.png>)
* Le statut des nœuds, avec le nœud 1 non disponible, et les pods déployés sur le nœud 2 :
![alt text](<Capture d'écran 2024-12-10 100420.png>)


## Sources ##
* https://gitlab.agglo-lepuyenvelay.fr/-/snippets/1036 : Lien vers le tutoriel utilisé pour la création du cluster Kubernetes, avec des modifications faites pour l'installation de containerd ( changement de package et installation de qemu pour que les images soient compatibles, avec cette commande):
```bash
apt-get install qemu-system qemu-user  qemu-user-static
```
