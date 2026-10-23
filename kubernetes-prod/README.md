# Домашнее задание №14 — Kubernetes Production

## Цель

Развернуть Kubernetes-кластер с помощью kubeadm и обновить его до следующей минорной версии.

## Окружение

- Виртуализация: GNS3 / QEMU
- ОС: Ubuntu 26.04 LTS
- Container Runtime: containerd 2.2.2
- CNI: Flannel
- Способ установки: kubeadm
- Pod CIDR: 10.244.0.0/16

## Состав кластера

| Узел | IP | Роль |
|---|---|---|
| k8s-cp1 | 192.168.122.5 | Control Plane |
| k8s-worker1 | 192.168.122.223 | Worker |
| k8s-worker2 | 192.168.122.158 | Worker |
| k8s-worker3 | 192.168.122.69 | Worker |

## Развертывание Kubernetes

На всех узлах выполнены:

1. Отключение swap.
2. Настройка модулей ядра overlay и br_netfilter.
3. Настройка параметров sysctl.
4. Установка containerd с SystemdCgroup=true.
5. Установка kubeadm, kubelet и kubectl версии 1.34.12.

### Инициализация Control Plane

    sudo kubeadm init \
      --kubernetes-version=v1.34.12 \
      --apiserver-advertise-address=192.168.122.5 \
      --pod-network-cidr=10.244.0.0/16 \
      --cri-socket=unix:///run/containerd/containerd.sock

### Установка Flannel

    kubectl apply -f \
      https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

Worker-ноды подключены через kubeadm join.

## Обновление Kubernetes

- Исходная версия: v1.34.12
- Целевая версия: v1.35.9

### Control Plane

1. Подключен репозиторий Kubernetes v1.35.
2. Обновлен kubeadm до версии 1.35.9.
3. Проверен план обновления.
4. Выполнено обновление Control Plane.
5. Обновлены kubelet и kubectl.
6. Перезапущен kubelet.

Команды:

    sudo kubeadm upgrade plan
    sudo kubeadm upgrade apply v1.35.9

### Worker-ноды

Каждая worker-нода обновлена последовательно.

На Control Plane:

    kubectl drain NODE --ignore-daemonsets --delete-emptydir-data

На worker-ноде:

    sudo apt-mark unhold kubeadm
    sudo apt-get install -y kubeadm=1.35.9-1.1
    sudo apt-mark hold kubeadm

    sudo kubeadm upgrade node

    sudo apt-mark unhold kubelet kubectl
    sudo apt-get install -y \
      kubelet=1.35.9-1.1 \
      kubectl=1.35.9-1.1
    sudo apt-mark hold kubelet kubectl

    sudo systemctl daemon-reload
    sudo systemctl restart kubelet

Возврат ноды в работу:

    kubectl uncordon NODE

## Результаты

После обновления:

- Все 4 узла находятся в статусе Ready.
- Все узлы работают на Kubernetes v1.35.9.
- Все системные Pod'ы находятся в состоянии Running.
- Flannel работает на всех четырех узлах.
- CoreDNS работает.

## Подтверждающие файлы

- [Узлы до обновления](nodes-before-upgrade.txt)
- [Узлы после обновления](nodes-after-upgrade.txt)
- [Pod'ы до обновления](pods-before-upgrade.txt)
- [Pod'ы после обновления](pods-after-upgrade.txt)
