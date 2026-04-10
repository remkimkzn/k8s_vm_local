Базовая подготовка (везде)
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg
Отключаем swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
Модули ядра
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
cat /proc/sys/net/ipv4/ip_forward
/etc/hosts
sudo tee -a /etc/hosts <<EOF
192.168.122.101 master
192.168.122.102 node-1
192.168.122.103 node-2
EOF
Установка CRI‑O (v1.35)
export CRIO_VERSION=v1.35
sudo curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o.gpg] \
https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" \
 | sudo tee /etc/apt/sources.list.d/cri-o.list
sudo apt update
sudo apt install -y cri-o
sudo systemctl enable --now crio
systemctl status crio
export KUBERNETES_VERSION=v1.34
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] \
https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" \
 | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
ГОТОВО. Система чистая и правильно настроена.
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=192.168.122.101
Настроить kubectl на master
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
Установить сетевой плагин (Calico под твой CIDR 192.168.0.0/16)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
kubectl get pods -n kube-system
Присоединить node-1 и node-2
sudo kubeadm join 192.168.122.101:6443 --token sp8ijw.4siyoedsdqrvj12v \
  --discovery-token-ca-cert-hash sha256:e7864aba1c4405dc9b280647b9fe601b3d77a5ebc2022a4bbf2031aec1c83ddb
Проверка кластера
На master:
kubectl get nodes



















































