
# Задание
Необходимо спроектировать и реализовать (опционально) автоматизированный процесс развертывания веб-приложения с использованием подходов Infrastructure as Code (IaC) и CI/CD. Задание охватывает создание инфраструктуры, настройку конфигураций, контейнеризацию приложения и внедрение CI/CD-пайплайнов для сборки, тестирования и деплоя.

# Архитектура системы
- Кластер разворачиваем на собсвенном железе под управлением OpenStack.
- Для начала, с помощью Terraform, установим основные узлы системы и поднимем сеть. Основные узлы системы: VPC, Postgres, Kubernetes/kubeadm/containerd, Kafka, Redis, Minio, Elasticsearch и т.д.
- Затем создадим плэйбуки Ansible для конфигурирования установленных систем.
- Далее создадим Dockerfile, Deployment, Service, Ingress для запуска приложения в Kubernetes.
- И последний шаг, создадим пайплайны в Gitlab. Первый пайплайн - запуск скриптов Terraform, последующий запуск плэйбуков Ansible. Второй пайплайн - сборка приложения, билд докер образа, установка/обновление ресурсов в кубернетес, запуск тестов.

# 1. Создание инфраструктуры
Пример tf скрипта для поднятия Postgres
```
# Настройка провайдера OpenStack
provider "openstack" {
  auth_url    = "https://keystone.example.com:5000/v3"
  user_name   = "admin"
  password    = var.openstack_password
  tenant_name = "admin"
}

# Создаем 3 ВМ для кластера Patroni (кворум из 3 нод)
resource "openstack_compute_instance_v2" "patroni" {
  count       = 3
  name        = "patroni-${count.index}"
  image_name  = "Ubuntu-22.04"  # Базовый образ без PostgreSQL
  flavor_name = "m1.medium"
  key_pair    = "ssh-key"

  network {
    name = "private_network"
  }

  # Отдельный диск для данных PostgreSQL
  block_device {
    boot_index            = 0
    delete_on_termination = true
    destination_type      = "volume"
    source_type           = "image"
    volume_size           = 20  # GB
  }

  # Теги для Ansible
  metadata = {
    role = "patroni"
  }
}

# Security Group для Patroni и PostgreSQL
resource "openstack_networking_secgroup_v2" "patroni_sg" {
  name        = "patroni-sg"
  description = "Allow Patroni and PostgreSQL traffic"
}

# Правила доступа
resource "openstack_networking_secgroup_rule_v2" "patroni_rules" {
  for_each = {
    ssh     = { port = 22 }
    postgres = { port = 5432 }
    patroni = { port = 8008 }  # REST API Patroni
    etcd    = { port = 2379 }  # Если etcd на тех же нодах
  }

  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = each.value.port
  port_range_max    = each.value.port
  remote_ip_prefix = "10.0.0.0/24"
  security_group_id = openstack_networking_secgroup_v2.patroni_sg.id
}

# Выводим IP-адреса нод
output "patroni_ips" {
  value = openstack_compute_instance_v2.patroni[*].access_ip_v4
}
```
# 2. Настройка конфигураций
```
---
- name: Configure Patroni cluster
  hosts: patroni_servers
  become: yes
  vars:
    patroni_version: "3.0.1"
    postgres_version: "14"
    cluster_name: "my-postgres-cluster"
    etcd_hosts: "patroni-0:2379,patroni-1:2379,patroni-2:2379"  # etcd на тех же нодах

  tasks:
    # Установка зависимостей
    - name: Install required packages
      ansible.builtin.apt:
        name:
          - python3-pip
          - postgresql-{{ postgres_version }}
          - postgresql-client-{{ postgres_version }}
          - postgresql-server-dev-{{ postgres_version }}
          - etcd  # Для координации кластера
        update_cache: yes

    # Установка Patroni
    - name: Install Patroni via pip
      ansible.builtin.pip:
        name:
          - patroni[etcd]=={{ patroni_version }}

    # Конфигурация etcd
    - name: Configure etcd
      ansible.builtin.template:
        src: templates/etcd.conf.j2
        dest: /etc/default/etcd
      notify: Restart etcd

    # Конфигурация Patroni
    - name: Create Patroni config
      ansible.builtin.template:
        src: templates/patroni.yml.j2
        dest: /etc/patroni/patroni.yml
        owner: postgres
        group: postgres
        mode: 0600

    # Systemd unit для Patroni
    - name: Configure Patroni service
      ansible.builtin.template:
        src: templates/patroni.service.j2
        dest: /etc/systemd/system/patroni.service
      notify: Reload systemd

    # Запуск сервисов
    - name: Ensure etcd is running
      ansible.builtin.systemd:
        name: etcd
        state: started
        enabled: yes

    - name: Ensure Patroni is running
      ansible.builtin.systemd:
        name: patroni
        state: started
        enabled: yes

  handlers:
    - name: Restart etcd
      ansible.builtin.systemd:
        name: etcd
        state: restarted

    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: yes
```




















