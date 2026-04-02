# Домашнее задание к занятию «Организация сети»

## ` Дмитрий Климов `

### Задание 1. Yandex Cloud

1. Создать пустую VPC. Выбрать зону.
2. Публичная подсеть.
   * Создать в VPC subnet с названием public, сетью 192.168.10.0/24.
   * Создать в этой подсети NAT-инстанс, присвоив ему адрес 192.168.10.254. В качестве image_id использовать                    fd80mrhj8fl2oe87o4e1.
   * Создать в этой публичной подсети виртуалку с публичным IP, подключиться к ней и убедиться, что есть доступ к               интернету.
3. Приватная подсеть.
   * Создать в VPC subnet с названием private, сетью 192.168.20.0/24.
   * Создать route table. Добавить статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс.
   * Создать в этой приватной подсети виртуалку с внутренним IP, подключиться к ней через виртуалку, созданную ранее, и         убедиться, что есть доступ к интернету.
  
   ## Ответ:

   # Yandex Cloud Infrastructure with Terraform

Данный код Terraform создает следующую инфраструктуру в Yandex Cloud:
-   Пустую VPC сеть.
-   Публичную подсеть (`192.168.10.0/24`) с NAT-инстансом.
-   Приватную подсеть (`192.168.20.0/24`) с маршрутом через NAT-инстанс.
-   Виртуальную машину в публичной подсети с публичным IP для доступа к приватной сети (Bastion Host).
-   Виртуальную машину в приватной подсети без публичного IP, с выходом в интернет через NAT-инстанс.

---

## Структура файлов проекта

Создайте следующие файлы в корневой директории вашего проекта:

1.  `provider.tf`
2.  `variables.tf`
3.  `main.tf`
4.  `terraform.tfvars`
5.  `.gitignore`

---

### 1. `provider.tf`

```hcl
# provider.tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token     = var.token
  cloud_id  = var.cloud_id
  folder_id = var.folder_id
  zone      = "ru-central1-a"
}
```

---

### 2. ` variables.tf `
```Hcl
variable "token" {
  description = "Yandex Cloud IAM token or OAuth token"
  type        = string
  sensitive   = true
}
variable "cloud_id" {
  description = "Yandex Cloud ID"
  type        = string
}
variable "folder_id" {
  description = "Yandex Cloud Folder ID where resources will be created"
  type        = string
}

variable "public_key_path" {
  description = "Path to public key for SSH access to VMs"
  default     = "~/.ssh/id_rsa.pub"
}
```

---

### 3. ` main.tf `

```Hcl
# 1. VPC
resource "yandex_vpc_network" "develop" {
  name = "develop"
}

# 2. Публичная подсеть
resource "yandex_vpc_subnet" "public" {
  name           = "public"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

# 3. Приватная подсеть
resource "yandex_vpc_subnet" "private" {
  name           = "private"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = ["192.168.20.0/24"]
  route_table_id = yandex_vpc_route_table.nat-route.id
}

# 4. Route Table
resource "yandex_vpc_route_table" "nat-route" {
  name       = "nat-route"
  network_id = yandex_vpc_network.develop.id

  static_route {
    destination_prefix = "0.0.0.0/0"
    next_hop_address   = "192.168.10.254"
  }
}

# 5. NAT-инстанс
resource "yandex_compute_instance" "nat-instance" {
  name        = "nat-instance"
  platform_id = "standard-v1"
  zone        = "ru-central1-a"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd80mrhj8fl2oe87o4e1" # Образ NAT-инстанса
    }
  }

  network_interface {
    subnet_id  = yandex_vpc_subnet.public.id
    ip_address = "192.168.10.254"
    nat        = true # Нужно для доступа в интернет
  }

  metadata = {
    ssh-keys = "ubuntu:${file(var.public_key_path)}"
  }
}

# 6. Публичная ВМ (для теста)
resource "yandex_compute_instance" "public-vm" {
  name        = "public-vm"
  platform_id = "standard-v1"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd83c1pf8uf99qhppnvb" # Ubuntu 22.04
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.public.id
    nat       = true
  }

  metadata = {
    ssh-keys = "ubuntu:${file(var.public_key_path)}"
  }
}

# 7. Приватная ВМ
resource "yandex_compute_instance" "private-vm" {
  name        = "private-vm"
  platform_id = "standard-v1"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd83c1pf8uf99qhppnvb"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.private.id
    nat       = false # У приватной ВМ нет публичного IP
  }

  metadata = {
    ssh-keys = "ubuntu:${file(var.public_key_path)}"
  }
}
```

###  4. ` terraform.tfvars.example `

```Hcl
token           = "YANDEX_CLOUD_IAM_ИЛИ_OAUTH_ТОКЕН"
cloud_id        = "YANDEX_CLOUD_ID"
folder_id       = "YANDEX_CLOUD_FOLDER_ID"
public_key_path = "~/.ssh/id_rsa.pub"
```
### 5. ` .gitignore `
```Hcl
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfplan
*.tfplan.zip
.vscode/
```
---

```bash
terraform init
terraform plan
terraform apply
```
---

<img width="1920" height="1080" alt="Снимок экрана (3221)" src="https://github.com/user-attachments/assets/3050b4d0-c719-4472-a127-baedec7d07cd" />


---

<img width="1920" height="1080" alt="Снимок экрана (3224)" src="https://github.com/user-attachments/assets/d8559dc0-b133-4cfa-9b2d-9cd197d52be6" />

---

```bash
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_rsa
ssh -A ubuntu@<ПУБЛИЧНЫЙ_IP_PUBLIC_VM>
```

---

<img width="1920" height="1080" alt="Снимок экрана (3222)" src="https://github.com/user-attachments/assets/08eb0c6d-8af9-488f-92b2-6203899cb0ee" />

---

```bash
ssh ubuntu@<ВНУТРЕННИЙ_IP_PRIVATE_VM>
ping -c 4 8.8.8.8
```

---

<img width="1920" height="1080" alt="Снимок экрана (3223)" src="https://github.com/user-attachments/assets/7c4cdc3b-58c5-4324-a3d4-a1fa99f31c57" />

---

```bash
terraform destroy
```

---

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/dfd1cc0a-354a-41e2-91c1-95f4dba4c398" />








