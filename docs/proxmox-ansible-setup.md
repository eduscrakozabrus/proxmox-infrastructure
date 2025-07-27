# Настройка Proxmox VE с Ansible

## Текущая конфигурация сервера

**Хост:** pve-dev-01  
**Версия Proxmox:** 8.4.5  
**Публичный IP:** 65.109.114.18/26  
**Gateway:** 65.109.114.1  
**Сетевой интерфейс:** enp6s0  

### Текущая сетевая конфигурация

```bash
# Основной интерфейс
enp6s0: 65.109.114.18/26 (публичная сеть)
IPv6: 2a01:4f9:3051:4052::2/64

# Маршрутизация
default via 65.109.114.1 dev enp6s0
65.109.114.0/26 via 65.109.114.1 dev enp6s0
```

### Дисковая подсистема

```bash
# RAID массивы
md0: RAID1 256MB (/boot/efi) - nvme0n1p1 + nvme1n1p1
md1: RAID1 1GB (/boot) - nvme0n1p2 + nvme1n1p2 
md2: RAID1 1.75TB - nvme0n1p3 + nvme1n1p3 (resync в процессе: 13.2%)

# LVM конфигурация
VG: vg0 (1.75TB total, 1.72TB свободно)
├── root (15GB) - система Proxmox
└── swap (6GB) - файл подкачки

# Диски
nvme0n1: 1.7TB NVMe SSD
nvme1n1: 1.7TB NVMe SSD (RAID1 mirror)

# Proxmox Storage
local: 14.67GB total (9.93GB доступно, 27% использовано)
Типы: rootdir, vztmpl, images, iso, backup, snippets
```

## План автоматизации с Ansible

### 1. Структура проекта

```
proxmox-infrastructure/
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── proxmox.yml
├── playbooks/
│   ├── site.yml
│   ├── setup-network.yml
│   ├── create-templates.yml
│   └── deploy-vms.yml
├── roles/
│   ├── proxmox-base/
│   ├── proxmox-network/
│   ├── vm-templates/
│   └── vm-deploy/
└── files/
    └── templates/
```

### 2. Сетевая архитектура

#### Планируемые бриджи:
- `vmbr0` - Публичная сеть (bridged с enp6s0)
- `vmbr1` - Приватная сеть (10.0.1.0/24) с NAT
- `vmbr2` - DMZ сеть (10.0.2.0/24)
- `vmbr3` - Management сеть (10.0.3.0/24)

#### NAT конфигурация:
```bash
# iptables правила для NAT
iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -o enp6s0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -o enp6s0 -j MASQUERADE
```

### 3. Ansible роли

#### Role: proxmox-base
- Обновление системы
- Настройка репозиториев
- Базовая конфигурация кластера

#### Role: proxmox-network
- Создание Linux Bridge интерфейсов
- Настройка VLAN
- Конфигурация NAT и firewall

#### Role: vm-templates
- Создание cloud-init шаблонов
- Ubuntu 22.04 LTS template
- CentOS Stream 9 template

#### Role: vm-deploy
- Автоматическое создание VM
- Конфигурация сети
- Управление ресурсами

### 4. Inventory конфигурация

```yaml
# inventory/hosts.yml
all:
  children:
    proxmox:
      hosts:
        pve-dev-01:
          ansible_host: 65.109.114.18
          ansible_user: root
          pve_node: pve-dev-01
```

### 5. Основные переменные

```yaml
# group_vars/proxmox.yml
public_interface: enp6s0
public_ip: 65.109.114.18/26
gateway: 65.109.114.1

# Дисковая подсистема
lvm_vg: vg0
available_space: 1.72TB
storage_pools:
  vm_data:
    size: 800G
    path: /var/lib/vz
  vm_images: 
    size: 500G
    path: /var/lib/vz/images
  vm_backup:
    size: 400G
    path: /var/lib/vz/dump

bridges:
  vmbr0:
    ports: enp6s0
    comment: "Public Bridge"
  vmbr1:
    comment: "Private NAT Network"
    cidr: "10.0.1.1/24"
  vmbr2:
    comment: "DMZ Network"
    cidr: "10.0.2.1/24"

vm_templates:
  ubuntu:
    vmid: 9000
    name: "ubuntu-22.04-template"
    disk_size: 20G
    cores: 2
    memory: 2048
    storage: vm_images
  centos:
    vmid: 9001
    name: "centos-stream9-template"
    disk_size: 20G
    cores: 2
    memory: 2048
    storage: vm_images
```

### 6. Основные плейбуки

#### site.yml - Главный плейбук
```yaml
---
- import_playbook: setup-network.yml
- import_playbook: create-templates.yml
- import_playbook: deploy-vms.yml
```

#### setup-network.yml - Настройка сети
```yaml
---
- hosts: proxmox
  roles:
    - proxmox-base
    - proxmox-network
```

### 7. Команды для запуска

```bash
# Проверка подключения
ansible -i inventory/hosts.yml proxmox -m ping

# Настройка сети
ansible-playbook -i inventory/hosts.yml playbooks/setup-network.yml

# Создание шаблонов
ansible-playbook -i inventory/hosts.yml playbooks/create-templates.yml

# Полная настройка
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### 8. Мониторинг и бэкапы

#### Автоматические бэкапы:
- Ежедневные бэкапы в 23:00
- Ротация: 7 дней локально
- Экспорт в внешнее хранилище

#### Мониторинг:
- Prometheus + Grafana
- Алерты по email/Telegram
- Мониторинг ресурсов VM

### 9. Безопасность

#### Firewall правила:
- Закрытие ненужных портов
- Разрешение только SSH (22) и Web UI (8006)
- Ограничение доступа к Management сети

#### Пользователи:
- Отключение root SSH с паролем
- Создание dedicated пользователей
- Настройка sudo правил

### 10. Управление дисковым пространством

#### Создание дополнительных LVM томов:
```bash
# Создание томов для VM данных
lvcreate -L 800G -n vm-data vg0
lvcreate -L 500G -n vm-images vg0  
lvcreate -L 400G -n vm-backup vg0

# Форматирование
mkfs.ext4 /dev/vg0/vm-data
mkfs.ext4 /dev/vg0/vm-images
mkfs.ext4 /dev/vg0/vm-backup

# Монтирование
echo "/dev/vg0/vm-data /var/lib/vz ext4 defaults 0 2" >> /etc/fstab
echo "/dev/vg0/vm-images /var/lib/vz/images ext4 defaults 0 2" >> /etc/fstab
echo "/dev/vg0/vm-backup /var/lib/vz/dump ext4 defaults 0 2" >> /etc/fstab
```

#### Мониторинг RAID:
- Статус resync: `cat /proc/mdstat`
- Проверка дисков: `smartctl -a /dev/nvme0n1`
- Автоматические уведомления при сбоях

#### Рекомендации:
- **Дождаться завершения RAID resync** перед активным использованием
- Настроить мониторинг SMART атрибутов дисков
- Регулярные scrub операции для проверки целостности

## Следующие шаги

1. **Дождаться завершения RAID resync** (текущий прогресс: 13.2%)
2. Создать дополнительные LVM тома для VM данных
3. Создать структуру Ansible проекта
4. Написать роль для настройки сети и хранилищ
5. Создать шаблоны VM с cloud-init
6. Автоматизировать развертывание
7. Настроить мониторинг и бэкапы