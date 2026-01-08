# 🚀 УСТАНОВКА KUBERNETES С НУЛЯ 

**🎯 Цель:** поднять полностью готовый кластер Kubernetes (Ready) на Ubuntu 22.04 / 24.04 с Flannel.

Ниже — рекомендуемые параметры для Kubernetes-кластера, особенно если worker-ноды на HDD.

## 1️⃣ Control Plane (Master)

**❗ Только SSD / NVMe
**
Минимум (для учебы):
```bash
CPU: 2 vCPU

RAM: 4 GB

Disk: 40–50 GB SSD

IOPS: желательно 3000+

Рекомендуемо:

CPU: 4 vCPU

RAM: 8 GB

Disk: 80–100 GB SSD

Причина: etcd очень чувствителен к задержкам диска
```
## 2️⃣ Worker Node (на HDD)

Можно, но аккуратно 👇

Минимум:
```bash
CPU: 2 vCPU

RAM: 4 GB

Disk: 80 GB HDD

Рекомендуемо:

CPU: 4 vCPU

RAM: 8–16 GB

Disk: 120–200 GB HDD

💡 Чем больше RAM — тем меньше боли от HDD, потому что:

кэшируется fs

меньше обращений к диску
```
> 💡 IP мастер-ноды: `172.16.18.196`  
> Все команды можно выполнять по шагам или автоматизировать через скрипты.

---

## 📦 СТРУКТУРА РЕПОЗИТОРИЯ


```bash
├── README.md                        # Основная инструкция
├── manifests/
│   ├── activar-deployment.yml       # Пример деплоя приложения
│   ├── dashboard-admin.yaml         # Конфиг для Dashboard admin-user
│   └── kube-flannel.yml             # Сетевой плагин Flannel (ссылка)
└── scripts/
    ├── prepare-system.sh            # Подготовка системы
    ├── install-containerd.sh        # Установка containerd
    └── install-k8s-packages.sh      # Установка kubeadm/kubelet/kubectl
```
## Топология:

🧠 MASTER — 172.16.18.196

⚙️ WORKER-1 — 172.16.18.164

⚙️ WORKER-2 — 172.16.18.165
---

## 🧩 1. Подготовка системы
📍 Где: MASTER + WORKER-1 + WORKER-2 (ВСЕ НОДЫ)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg

# 🔧 Отключаем swap (Kubernetes его не допускает)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
# 🧱 Настраиваем модули ядра и сетевые параметры
📍 Где: MASTER + WORKER-1 + WORKER-2
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

---

## 🧩 2. Установка и настройка containerd
📍 Где: MASTER + WORKER-1 + WORKER-2
```bash
sudo apt install -y containerd

# Создаём конфигурацию по умолчанию
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Включаем поддержку systemd cgroups
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Перезапуск и автозагрузка
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager
```

---

## 🧩 3. Установка Kubernetes-компонентов

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```
📍 Где: MASTER + WORKER-1 + WORKER-2
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 🧩 4. Инициализация мастер-ноды

```bash
sudo kubeadm init \
  --apiserver-advertise-address=172.16.18.196 \
  --pod-network-cidr=10.244.0.0/16
```

После успешной инициализации:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

---

## 🧩 5. Установка Flannel (CNI)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

> При необходимости установите CNI-плагины вручную:
```bash
sudo mkdir -p /opt/cni/bin
curl -L -o cni-plugins.tgz https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
sudo tar -C /opt/cni/bin -xzvf cni-plugins.tgz
sudo systemctl restart kubelet
```

Проверка:
```bash
kubectl get pods -n kube-system
kubectl get nodes
```
Через 1–2 минуты нода должна стать **Ready** ✅

---

## 🧩 6. Установка Worker-ноды

```bash
sudo kubeadm join 172.16.18.196:6443 --token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

> 💡 Команду возьмите из вывода `kubeadm init`.

Если нужно создать вручную CNI-конфиг:
```bash
sudo tee /etc/cni/net.d/10-flannel.conflist > /dev/null <<'EOF'
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
EOF

sudo systemctl restart containerd
sudo systemctl restart kubelet
```

---

## 🧩 7. Установка Kubernetes Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Создаём пользователя администратора:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Применяем:
```bash
kubectl apply -f dashboard-admin.yaml
kubectl -n kubernetes-dashboard create token admin-user
```

Делаем Dashboard доступным извне:
```bash
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
# Меняем: type: ClusterIP → type: NodePort
kubectl -n kubernetes-dashboard get svc
```

🔗 Теперь открой в браузере:  
`https://<IP_МАСТЕРА>:<NodePort>`  
и вставь токен.

---

## 🧩 8. Пример Deployment + Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: activar-deployment
  labels:
    app: activar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: activar
  template:
    metadata:
      labels:
        app: activar
    spec:
      containers:
      - name: activar
        image: jacobvell/activar:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: activar-service
spec:
  selector:
    app: activar
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: NodePort
```

---

## 🧹 Удаление воркер-ноды

На мастере:
```bash
kubectl drain worker-node1 --delete-emptydir-data --force --ignore-daemonsets
kubectl delete node worker-node1
```

На воркере:
```bash
sudo kubeadm reset -f
sudo ip link delete cni0
sudo ip link delete flannel.1
sudo rm -rf /etc/cni/net.d /var/lib/cni/ /var/lib/kubelet/* ~/.kube
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -X
```

---

## 🧠 Полезные команды

```bash
kubectl get pods -A
kubectl get nodes -o wide
kubectl logs -n kube-system <pod>
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f
```

---

## 🔐 Безопасность

- Не используйте `admin-user` с полным доступом в production.
- Ограничьте доступ к Dashboard (через `kubectl proxy` или VPN).
- Храните токены и kubeconfig в безопасном месте.

---

## 📜 Лицензия

**MIT License** — свободно используйте и модифицируйте.

