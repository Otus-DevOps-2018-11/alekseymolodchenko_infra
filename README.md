[![Build Status](https://travis-ci.com/Otus-DevOps-2018-11/alekseymolodchenko_infra.svg?branch=master)](https://travis-ci.com/Otus-DevOps-2018-11/alekseymolodchenko_infra)

## Remotes

```
bastion_IP = 35.206.191.187
someinternalhost_IP = 10.132.0.3
testapp_IP = 35.204.237.177
testapp_port = 9292
```

## Настройка ssh bastion
#### 1. Подключение к someinternalhost одной командой

- ##### C использованием Pseudo-terminal

```
ssh -tt -A -i ~/.ssh/gcp_appuser gcp_appuser@35.206.191.187 ssh 10.132.0.3
```

- ##### C использованием ProxyCommand

```
ssh -o ProxyCommand='ssh -A -i ~/.ssh/gcp_appuser -W %h:%p gcp_appuser@35.206.191.187' 10.132.0.3
```

- ##### C Использованием ProxyJump

```
ssh -i ~/.ssh/gcp_appuser -A -J gcp_appuser@35.206.191.187 10.132.0.3
```

#### 2. Подключение к someinternalhost использованием alias-a

В файл-конфигурации ~/.ssh/config внести следующие настройки

```
Host bastion
User gcp_appuser
Hostname 35.206.191.187
IdentityFile ~/.ssh/gcp_appuser

Host someinternalhost
HostName 10.132.0.3
User gcp_appuser
ProxyCommand ssh bastion -W %h:%p
IdentityFile ~/.ssh/gcp_appuser
```

Команда подключения

```
ssh someinternalhost
```

## Настройка OpenVPN
#### 1. На bastion host создаем файл установки vpn-сервера setupvpn.sh

```
#!/bin/bash
echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.4 multiverse" > /etc/apt/sources.list.d/mongodb-org-3.4.list
echo "deb http://repo.pritunl.com/stable/apt xenial main" > /etc/apt/sources.list.d/pritunl.list
apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv 0C49F3730359A14518585931BC711F9BA15703C6
apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv 7568D9BB55FF9E5287D586017AE645C0CF8E292A
apt-get --assume-yes update
apt-get --assume-yes upgrade
apt-get --assume-yes install pritunl mongodb-org
systemctl start pritunl mongod
systemctl enable pritunl mongod
```

#### 2. Делаем файл исполняемым и запускаем

```
chmod +x ~/setupvpn.sh
./setupvpn.sh
```

#### 3. Настраиваем сервер pritunl
* Добавляем организацию
* Добавляем пользователя
* Добавляем Сервер
* Привязываем сервер к организации

## Работа с CGP
#### 1. Создание истанса с использованием gcloud

```
gcloud compute instances create reddit-app\
  --boot-disk-size=10GB \
  --image-family ubuntu-1604-lts \
  --image-project=ubuntu-os-cloud \
  --machine-type=g1-small \
  --zone europe-west3-a \
  --tags puma-server \
  --restart-on-failure
```

Просмотреть список созданых инстансов можно командой

```
gcloud compute instances list
```

#### 2. Создание истанса с использованием gcloud с ключем startup-script

```
gcloud compute instances create reddit-app \
  --boot-disk-size=10GB \
  --image-family ubuntu-1604-lts \
  --image-project=ubuntu-os-cloud \
  --machine-type=g1-small \
  --zone europe-west3-a \
  --tags puma-server \
  --restart-on-failure \
  --metadata startup-script='#! /bin/bash
    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
    sudo bash -c \"echo \"deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse\" > /etc/apt/sources.list.d/mongodb-org-3.2.list\"
    sudo apt update
    sudo apt install -y mongodb-org ruby-full ruby-bundler build-essential
    sudo systemctl start mongod
    sudo systemctl enable mongod
    git clone -b monolith https://github.com/express42/reddit.git
    cd reddit && bundle install
    puma -d'
```

#### 3. Создание истанса с использованием gcloud с ключем startup-script-url
```
gcloud compute instances create reddit-app \
  --boot-disk-size=10GB \
  --image-family ubuntu-1604-lts \
  --image-project=ubuntu-os-cloud \
  --machine-type=g1-small \
  --zone europe-west3-a \
  --tags puma-server \
  --restart-on-failure \
  --metadata startup-script-url=https://raw.githubusercontent.com/Otus-DevOps-2018-11/alekseymolodchenko_infra/master/startup.sh
```

#### 4. Добавление правила default-puma-server с использование gcloud

```
gcloud compute firewall-rules create puma-default-server --allow tcp:9292 \
    --description "Allow incoming traffic on TCP port 9292 for Reddit App" \
    --direction=INGRESS \
    --target-tags="puma-server"
```

Просмотреть список правил firewall'a можно командой

```
gcloud compute firewall-rules list
```

## Работа с HashiCorp Packer
#### 1. Создание образа reddit-base

ubuntu16.json
```
{
  "variables": {
    "gcp_project_id": null,
    "gcp_source_image_family": null,
    "gcp_machine_type": "f1-micro"
  },

  "sensitive-variables": ["gcp_project_id", "gcp_source_image_family"],

  "builders": [{
    "type": "googlecompute",
    "project_id": "{{user `gcp_project_id`}}",
    "image_name": "reddit-base-{{timestamp}}",
    "image_family": "reddit-base",
    "source_image_family": "{{user `gcp_source_image_family`}}",
    "image_description": "Reddit App Base Image",
    "machine_type": "{{user `gcp_machine_type`}}",
    "disk_size": "10",
    "disk_type": "pd-standard",
    "network": "default",
    "tags": "reddit-base,reddit-image",
    "zone": "europe-west1-b",
    "ssh_username": "appuser"
  }],

  "provisioners": [{
    "type": "shell",
    "scripts": [
      "scripts/install_ruby.sh",
      "scripts/install_mongodb.sh"
    ],
    "execute_command": "sudo {{.Path}}"
  }]
}
```

Собрать образ reddit-base
```
packer build -var-file=variables.json ubuntu16.json
```

#### 2. Создание образа immutable reddit-full на основании reddit-base

immutable.json
```
{
  "variables": {
    "gcp_project_id": null,
    "gcp_source_image_family": null,
    "gcp_machine_type": "f1-micro"
  },

  "sensitive-variables": ["gcp_project_id", "gcp_source_image_family"],

  "builders": [{
    "type": "googlecompute",
    "project_id": "{{user `gcp_project_id`}}",
    "image_name": "reddit-full-{{timestamp}}",
    "image_family": "reddit-full",
    "image_description": "Reddit App Immutable Image",
    "source_image_family": "reddit-base",
    "zone": "europe-west1-b",
    "ssh_username": "appuser",
    "machine_type": "{{user `gcp_machine_type`}}",
    "network": "default",
    "disk_size": "10",
    "disk_type": "pd-standard",
    "tags": "reddit-immutable,reddit-image"
  }],

  "provisioners": [{
      "type": "file",
      "source": "files/redditapp.service",
      "destination": "/tmp/redditapp.service"
    },
    {
      "type": "shell",
      "script": "files/deploy_app.sh",
      "execute_command": "sudo {{.Path}}"
    }
  ]
}
```

Собрать образ reddit-full
```
packer build \
  -var 'gcp_project_id=infra-1234567' \
  -var 'gcp_source_image_family=reddit-full' \
  immutable.json
```

## Работа с HashiCorp Terrarom
#### 1. Добавление пользователя appuser_web

```
После применения terraform apply пользователь будет удален.
Это связано с тем, что terraform использует декларативный подход для описания инфраструктуры
и приводит состояние инфраструкруты к виду описаному в файлах конфигурации.
```

#### 2. Добавление балансировщика

lb.tf
```
resource "google_compute_forwarding_rule" "default" {
  name                  = "reddit-app"
  description           = "reddit app forwarding rule"
  port_range            = "9292"
  target                = "${google_compute_target_pool.default.self_link}"
  load_balancing_scheme = "EXTERNAL"
}

resource "google_compute_target_pool" "default" {
  name          = "reddit-app-instances"
  description   = "reddit app instance pool"
  instances     = ["${google_compute_instance.app.*.self_link}"]
  health_checks = ["${google_compute_http_health_check.default.name}"]
}

resource "google_compute_http_health_check" "default" {
  name                = "health-check-reddit-app"
  port                = 9292
  check_interval_sec  = 10
  timeout_sec         = 5
  unhealthy_threshold = 5
}
```
#### 3. Добавление добавление параметра count в ресурс app

```
resource "google_compute_instance" "app" {
  count = "${var.app_count}"
  name = "${format("reddit-app%02d", count.index+1)}"
  machine_type = "g1-small"
  zone         = "${var.zone}"

  tags = ["reddit-app"]

  # определение загрузочного диска
  boot_disk {
    initialize_params {
      image = "${var.disk_image}"
    }
  }

  metadata {
    ssh-keys = "appuser:${file(var.public_key_path)}"
  }

  # определение сетевого интерфейса
  network_interface {
    # сеть, к которой присоединить данный интерфейс
    network = "default"

    # использовать ephemeral IP для доступа из Интернет
    access_config {}
  }

  connection {
    type        = "ssh"
    user        = "appuser"
    agent       = false
    private_key = "${file(var.private_key_path)}"
  }

  provisioner "file" {
    source      = "files/puma.service"
    destination = "/tmp/puma.service"
  }

  provisioner "remote-exec" {
    script = "files/deploy.sh"
  }
}
```

#### 4. Удаление ресурсов

```
terraform destroy
```

#### 5. Сохранение state во внешнем бекенде для stage и prod окружения

stage/backend.tf
```
terraform {
  backend "gcs" {
    bucket = "terraform-2-otus-storage-state-bucket"
    prefix = "stage"
  }
}
```

prod/backend.tf
```
terraform {
  backend "gcs" {
    bucket = "terraform-2-otus-storage-state-bucket"
    prefix = "prod"
  }
}
```

#### 6. Добавление provisioner для деплоя приложения

modules/app/main.tf
```
connection {
    type        = "ssh"
    user        = "appuser"
    agent       = false
    private_key = "${file(var.private_key_path)}"
  }

  provisioner "file" {
    source      = "../modules/app/files/puma.service"
    destination = "/tmp/puma.service"
  }

  provisioner "file" {
    source      = "../modules/app/files/deploy.sh"
    destination = "/tmp/deploy.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/deploy.sh",
      "/tmp/deploy.sh ${var.db_address}"
    ]
  }
```

modules/db/main.tf
```
connection {
    type        = "ssh"
    user        = "appuser"
    agent       = false
    private_key = "${file(var.private_key_path)}"
  }

	provisioner "file" {
    source      = "../modules/db/files/deploy.sh"
    destination = "/tmp/deploy.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/deploy.sh",
      "sudo /tmp/deploy.sh ${self.network_interface.0.address}"
    ]
  }
```

Скрипты размещаем в директории files модулей

## Управление конфигурациями при помощи Ansible

#### 1. Cоздание playbook для клонирования репозитория

clone.yml
```
---
- name: Clone
  hosts: app
  tasks:
    - name: Clone repo
      git:
        repo: https://github.com/express42/reddit.git
        dest: /home/appuser/reddit

```

#### 2. Использование dynamic inventory

Файл inventory в формате json

inventory.json
```
{
  "app": {
  "hosts": ["34.76.71.112"]
  },
  "db": {
    "hosts": ["35.195.164.228"]
  }
}
```

Скрипт обработки inventory в формате json inventory.py

files/inventory.py
```
#!/usr/bin/env python
import os

__location__ = os.path.realpath(
    os.path.join(os.getcwd(), os.path.dirname(__file__)))

with open(os.path.join(__location__, "../inventory.json")) as f:
    print f.read()
```

В файле конфигурации подключаем нужные плагины и указывам что нудно использовать скрипт для inventory

ansible.cfg
```
[defaults]
inventory = ./files/inventory.py

...

[inventory]
enable_plugins = host_list, script, yaml, ini
```

Проверка

```
$ ansible all -m ping

34.76.71.112 | SUCCESS => {
    "changed": false,
    "failed": false,
    "ping": "pong"
}
35.195.164.228 | SUCCESS => {
    "changed": false,
    "failed": false,
    "ping": "pong"
}
```

#### 3. Добавляем скрипт gce_googleapiclient для dynamic inventory

ansible.cfg
```
[defaults]
...
inventory = inventory = gce_googleapiclient.py
...
```

Проверка
```
ansible all -i gce_googleapiclient.py -m ping
reddit-db-stage-01 | SUCCESS => {
    "changed": false,
    "failed": false,
    "ping": "pong"
}
reddit-app-stage-01 | SUCCESS => {
    "changed": false,
    "failed": false,
    "ping": "pong"
}
```

Для просмотра списка хостов выполняем команду
```
./gce_googleapiclient.py --list
```

#### 4. Использование ansible в качестве provisioner

packer/app.json
```
"provisioners": [{
    "type": "ansible",
    "playbook_file": "ansible/packer_app.yml"
    }
  ]
```

packer/db.json
```
"provisioners": [{
    "type": "ansible",
    "playbook_file": "ansible/packer_db.yml"
    }
  ]
```

## Разработка и тестирование Ansible ролей и плейбуков

### Локальная разработка с Vagrant

- Установлен Vagrant;
  ```bash
  $ vagrant -v
  Vagrant 2.2.3
  ```
- Добавлен ansible/Vagrantfile;
- Созданы VM's appserver и dbserver с помощью Vagrant;
- Добавлен playbook base.yml для установки python;
- Для роли db добалены файлы тасков install_mongo.yml и config_mongo.yml;
- Для роли app добалены файлы тасков puma.yml и ruby.yml. Добавлен параметр для деплоя пользователя;

#### Задание со *

- Дабавлено проксирования запросов с nginx

  <details><summary>Пример</summary><p>

  ```ruby

    ansible.extra_vars = {
      "deploy_user" => "ubuntu",
      "nginx_sites" => {
          "default" => [
           "listen 80 default_server",
            "server_name raddit",
            "location / { proxy_pass http://127.0.0.1:9292; }"
          ]
        }
    }

  ```

  </p></details>

#### Самостоятельное задание

- К роли db добавлен тест проверки доступности БД по порту 27017

  <details><summary>Тест</summary><p>

  ```python

  def test_mongodb_listening_port(host):
    port = host.socket('tcp://27017')
    assert port.is_listening

  ```

  </p></details>

