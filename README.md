# 4.md
«Отказоустойчивость в облаке» Васяева Ирина
# Задание 1
```
provider "yandex" {
  token     = var.yandex_token
  cloud_id  = var.yandex_cloud_id
  folder_id = var.yandex_folder_id
}

resource "yandex_compute_instance" "vm" {
  count        = 2
  name         = "vm-${count.index}"
  zone         = "ru-central1-a"
  platform_id  = "standard-v1"
  resources {
    memory = 2
    cores  = 2
  }

  network_interface {
    ipv4_address = "${lookup(module.network.network_interface[0], "primary")}"
    nat         = true
  }

  boot_disk {
    disk_id = yandex_compute_disk.vm_disk.id
  }

  metadata = {
    user-data = <<-EOF
      #!/bin/bash
      apt update
      apt install -y nginx
      systemctl start nginx
      systemctl enable nginx
      echo "<h1>Hello from VM ${count.index}</h1>" > /var/www/html/index.html
    EOF
  }
}

resource "yandex_compute_disk" "vm_disk" {
  count   = 2
  name    = "disk-${count.index}"
  size    = 10
  type    = "network-ssd"
}

# Создание целевой группы
resource "yandex_lb_target_group" "tg" {
  name        = "my-target-group"
  zone        = "ru-central1-a"

  healthcheck {
    interval = 10
    timeout  = 5
    healthy_threshold = 2
    unhealthy_threshold = 2

    http_options {
      path = "/"
      port = 80
      interval = 10
      timeout = 5
    }
  }

  instances {
    for_each = toset(yandex_compute_instance.vm[*].id)
    instance_id = each.value
  }
}

resource "yandex_lb_network_load_balancer" "nlb" {
  name = "my-network-lb"

  listener {
    port = 80
    target_group_id = yandex_lb_target_group.tg.id
  }

  deployment {
    region  = "ru-central1"
    zone    = "ru-central1-a"
  }
}

variable "yandex_token" {}
variable "yandex_cloud_id" {}
variable "yandex_folder_id" {}
```
### Объяснение кода:
1. ***Провайдер***: Используем провайдер Яндекс Облака с необходимыми переменными для токена, ID облака и ID папки.
2. ***Виртуальные машины***: Создаем две идентичные виртуальные машины с помощью аргумента `count`. Используем `user-data` для установки и запуска Nginx.
3. ***Целевая группа***: Создаем целевую группу, в которую будут включены созданные виртуальные машины, с конфигурацией проверки их работоспособности (healthcheck).
4. ***Сетевой балансировщик***: Создаем сетевой балансировщик, который будет слушать на порту 80 и отправлять трафик в целевую группу.
### Поэтапный план:  
1. Проверка установленых Terraform и необходимых плагинов Яндекс Облака.
2. Сохранить файл в директории проекта.
3. Создать файл `variables.tf` для определения переменных и указать значения.
4. Необходимо выполнить команды Terraform:  
   - `terraform init` для инициализации.
   - `terraform plan` для проверки плана.
   - `terraform apply` для создания ресурсов.

