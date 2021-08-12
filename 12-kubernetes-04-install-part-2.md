# Домашнее задание к занятию "12.4 Развертывание кластера на собственных серверах, лекция 2"
Новые проекты пошли стабильным потоком. Каждый проект требует себе несколько кластеров: под тесты и продуктив. Делать все руками — не вариант, поэтому стоит автоматизировать подготовку новых кластеров.

## Задание 1: Подготовить инвентарь kubespray
Новые тестовые кластеры требуют типичных простых настроек. Нужно подготовить инвентарь и проверить его работу. Требования к инвентарю:
* подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды;
* в качестве CRI — containerd;
* запуск etcd производить на мастере.
```yml
[all]
node1   ansible_host=169.254.25.10 ip=169.254.25.10
node2   ansible_host=169.254.25.20 ip=169.254.25.20

[kube-master]
node1

[etcd]
node1

[kube-node]
node2

[k8s-cluster:children]
kube-master
kube-node
```

1. git clone https://github.com/kubernetes-sigs/kubespray
2. cd kubespray
3. cp -rfp inventory/sample inventory/mycluster
4. vim inventory/mycluster/inventory.ini
5. sudo vim /etc/hosts
```
node1   ansible_host=169.254.25.10 
node2   ansible_host=169.254.25.20
```
6. pip3 install -r requirements.txt
7. vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```
container_manager: containerd
```

  vim inventory/group_vars/etcd.yml
```
etcd_deployment_type: host
```

  vim inventory/group_vars/all/containerd.yml
```
containerd_registries:
  "docker.io":
    - "https://mirror.gcr.io"
    - "https://registry-1.docker.io"
```
    
8.ansible-playbook -u vagrant -i inventory/mycluster/inventory.ini -b --diff cluster.yml --ask-pass -b --ask-become-pass

## Задание 2 (*): подготовить и проверить инвентарь для кластера в AWS
Часть новых проектов хотят запускать на мощностях AWS. Требования похожи:
* разворачивать 5 нод: 1 мастер и 4 рабочие ноды;
* работать должны на минимально допустимых EC2 — t3.small.
