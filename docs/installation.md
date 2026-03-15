# Полный план развертывания кластера Kubernetes 1.35 (1 master + 2 worker)  
CRI-O + Calico (март 2026)

## 0. Общие сведения и адреса

Мастер:   192.168.122.101   hostname: master  
Worker 1: 192.168.122.102   hostname: node-1  
Worker 2: 192.168.122.103   hostname: node-2  

Pod network (Calico VXLAN): 192.168.0.0/16

## 1. Подготовка всех трёх узлов (master + node-1 + node-2)
Выполнить **на всех машинах**:

```bash
# 1. Обновляем систему
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg

# 2. Отключаем swap навсегда
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 3. Настраиваем необходимые модули ядра
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF


# 4. Настраиваем sysctl параметры
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

https://github.com/cri-o/packaging/blob/main/README.md#usage

# 5. Удобные имена хостов (рекомендуется)
cat <<EOF | sudo tee -a /etc/hosts
192.168.122.101 master
192.168.122.102 node-1
192.168.122.103 node-2
EOF

export KUBERNETES_VERSION=v1.35
export CRIO_VERSION=v1.35
# 6. Установка CRI-O (Все узлы)
apt install gnupg
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list

apt-get update
apt-get install -y cri-o
systemctl start crio status

# 7. Установка kubeadm, kubelet, kubectl (Все узлы)
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# 8. Инициализация кластера (Только на мастер-узле)
kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=192.168.122.101
(специально для node-1 и node-2): Присоединение с конкретным мастером
kubeadm join 192.168.122.101:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# Проверить, какие IP использует API Server
kubectl cluster-info
# Посмотреть полную конфигурацию кластера
kubectl config view
# Проверить, что все компоненты мастера работают
kubectl get pods -n kube-system
# Создать новый токен и сразу вывести команду join
kubeadm token create --print-join-command
Настройка конфигурации (на master)
mkdir -p /home/deb/.kube
sudo cp -i /etc/kubernetes/admin.conf /home/deb/.kube/config
sudo chown -R deb:deb /home/deb/.kube
# Проверить, что kubectl работает
kubectl get nodes
дальше сетевой плагин калисто
