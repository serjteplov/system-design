
# Задание
Необходимо спроектировать и реализовать (опционально) автоматизированный процесс развертывания веб-приложения с использованием подходов Infrastructure as Code (IaC) и CI/CD. Задание охватывает создание инфраструктуры, настройку конфигураций, контейнеризацию приложения и внедрение CI/CD-пайплайнов для сборки, тестирования и деплоя.

# Архитектура инфраструктуры (описание)
- Кластер разворачивается на собсвенном железе под управлением OpenStack.
- С помощью Terraform, установливаются основные узлы системы и поднимается сеть. Основные узлы системы: VPC, Postgres, Kubernetes/kubeadm/containerd, Kafka, Redis, Minio, Elasticsearch и т.д.
- Далее создаются плэйбуки Ansible для конфигурирования установленных систем.
- Далее создаются Dockerfile, манифесты Deployment, Service, Ingress для запуска приложения в Kubernetes.
- Далее создаются пайплайны в Gitlab. Первый пайплайн - запуск скриптов Terraform, последующий запуск плэйбуков Ansible. 
- Второй пайплайн - сборка приложения, билд докер образа, установка/обновление ресурсов в кубернетес, запуск тестов.

# Архитектура инфраструктуры (драфт)
![Архитектура инфры](https://github.com/serjteplov/system-design/blob/2d4f63e619c78d6357a930fee8fa789a259242d6/dz6%20-%20cicd/arch_infra.jpg)


# 1. Создание инфраструктуры
Пример tf скрипта для поднятия Postgres и Patroni
```
provider "openstack" {
  auth_url    = "https://keystone.example.com:5000/v3"
  user_name   = "admin"
  password    = var.openstack_password
  tenant_name = "admin"
}

resource "openstack_compute_instance_v2" "db_node" {
  count       = 3
  name        = "db-node-${count.index}"
  image_name  = "Ubuntu-22.04"
  flavor_name = "m1.medium"
  key_pair    = "ssh-key"

  network {
    name = "private_network"
  }

  # Установка PostgreSQL и Patroni через cloud-init
  user_data = <<-EOF
    #cloud-config
    packages:
      - postgresql-14
      - postgresql-14-patroni
      - python3-pip
      - etcd
    runcmd:
      - pip3 install patroni[etcd]==2.1.2
      - systemctl enable etcd
      - systemctl start etcd
    EOF
}

resource "openstack_networking_secgroup_v2" "db_secgroup" {
  name        = "db-secgroup"
  description = "PostgreSQL and Patroni security group"
}

resource "openstack_networking_secgroup_rule_v2" "db_rules" {
  for_each = toset(["5432", "8008", "2379", "2380"])
  security_group_id = openstack_networking_secgroup_v2.db_secgroup.id
  direction         = "ingress"
  protocol          = "tcp"
  port_range_min    = each.key
  port_range_max    = each.key
  remote_ip_prefix = "10.0.0.0/24"
}

output "db_nodes_ips" {
  value = openstack_compute_instance_v2.db_node[*].access_ip_v4
}
```
# 2. Настройка конфигураций
Пример плэйбука на Ansible по настройке Postgres и репликации на Patroni
```
---
- name: Configure PostgreSQL and Patroni
  hosts: all
  become: yes
  vars:
    cluster_name: "postgres-cluster"
    etcd_hosts: "db-node-0:2379,db-node-1:2379,db-node-2:2379"
    postgres_version: "14"

  tasks:
    - name: Configure etcd
      template:
        src: etcd.conf.j2
        dest: /etc/default/etcd
      notify: restart etcd

    - name: Create Patroni configuration
      template:
        src: patroni.yml.j2
        dest: /etc/patroni/patroni.yml
        owner: postgres
        group: postgres
        mode: 0600

    - name: Configure Patroni service
      template:
        src: patroni.service.j2
        dest: /etc/systemd/system/patroni.service
      notify: reload systemd

    - name: Ensure services are running
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - etcd
        - patroni

  handlers:
    - name: restart etcd
      systemd:
        name: etcd
        state: restarted

    - name: reload systemd
      systemd:
        daemon_reload: yes
```
# 3. Контейнеризация приложения
Пример Dockerfile для демо-приложения CRUD, написанного на Go
```
FROM golang:1.21-alpine

WORKDIR /app

COPY go.mod ./
RUN go mod init go-crudl-service
RUN go get github.com/gin-gonic/gin

COPY main.go ./

RUN go build -o /go-crudl-service

EXPOSE 8080

CMD ["/go-crudl-service"]
```
Deployment manifest
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-crudl-deployment
  labels:
    app: go-crudl
spec:
  replicas: 2
  selector:
    matchLabels:
      app: go-crudl
  template:
    metadata:
      labels:
        app: go-crudl
    spec:
      containers:
      - name: go-crudl
        image: go-crudl-service:latest
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: "256Mi"
            cpu: "500m"
          requests:
            memory: "128Mi"
            cpu: "250m"
```
Service manifest
```
apiVersion: v1
kind: Service
metadata:
  name: go-crudl-service
spec:
  selector:
    app: go-crudl
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```
Ingress controller
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /go-crudl(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: go-crudl-service
            port:
              number: 80
      - path: /kotlin-worker(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: kotlin-worker-service
            port:
              number: 80
```
# 4. CI/CD-пайплайн
Пайплайн для раскатки инфраструктуры на серверах под управлением OpenStack

![Инфра](https://github.com/serjteplov/system-design/blob/3c5081f1fe7e6761f895bb41a9d8681986278f87/dz6%20-%20cicd/pipeline_infra.png)

Пайплайн для раскатки приложения в Kubernetes

![App](https://github.com/serjteplov/system-design/blob/3c5081f1fe7e6761f895bb41a9d8681986278f87/dz6%20-%20cicd/pipeline_application.png)


# 5. Мониторинг и отчёты
В Gitlab пайплайнах настраивается алёртинг по разным триггерам, например упавший билд, или слишком долгий билд. Также в системе работает сбор и визуализация метрик, что позволяет строить дашборды для мониторинга в Grafana. В Gitlab можно настроить отбрасывание метрик и сбор логов в единое хранилище.





















