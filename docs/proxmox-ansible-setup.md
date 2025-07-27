# Настройка Proxmox VE с Ansible - Детальная документация

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

# Linux Bridges
vmbr0: Публичный бридж (привязан к enp6s0)
vmbr1: Приватная сеть 10.0.1.0/24 с NAT
vmbr2: DMZ сеть 10.0.2.0/24 с NAT
```

### Дисковая подсистема

```bash
# LVM конфигурация
VG: vg0 (1.75TB total)
├── root (15GB) - система Proxmox
├── swap (6GB) - файл подкачки
├── vm-data (800GB) - /var/lib/vz - основное хранилище VM
├── vm-images (500GB) - /var/lib/vz/images - образы и шаблоны
└── vm-backup (400GB) - /var/lib/vz/dump - бэкапы

# Диски
nvme0n1: 1.7TB NVMe SSD
nvme1n1: 1.7TB NVMe SSD (RAID1 mirror)
```

## Детали реализации

### 1. Storage (LVM)

**Плейбук:** `playbooks/setup-storage.yml`

Создает и настраивает LVM тома:
- **vm-data**: 800GB для хранения дисков VM и контейнеров
- **vm-images**: 500GB для ISO образов и шаблонов VM
- **vm-backup**: 400GB для хранения бэкапов

Особенности:
- Проверка существующих томов перед созданием
- Автоматическое форматирование в ext4
- Монтирование в стандартные директории Proxmox
- Обновление /etc/fstab для постоянного монтирования

### 2. Network (Bridges + NAT)

**Плейбук:** `playbooks/setup-network.yml`

Настраивает сетевую инфраструктуру:
- **vmbr0**: Публичный бридж для VM с публичными IP
- **vmbr1**: Приватная сеть 10.0.1.1/24 (для внутренних сервисов)
- **vmbr2**: DMZ сеть 10.0.2.1/24 (для публичных сервисов)

Особенности:
- Включение IP forwarding (IPv4/IPv6)
- Настройка NAT через iptables для приватных сетей
- Создание бэкапа оригинального /etc/network/interfaces
- Установка iptables-persistent для сохранения правил

### 3. Firewall (pve-firewall)

**Плейбук:** `playbooks/setup-firewall.yml`

Использует встроенный Proxmox firewall:
- **Движок**: pve-firewall с iptables-legacy
- **Политика**: DROP для входящих, ACCEPT для исходящих
- **IPSet management**: Список разрешенных IP для управления

Разрешенные порты:
- SSH (22) - только с IP из management
- Proxmox Web UI (8006) - только с IP из management  
- VNC консоли (5900-5999) - только с IP из management
- Весь трафик из внутренних сетей (vmbr1, vmbr2)

Особенности:
- Автоматическое переключение на iptables-legacy
- Конфигурация через /etc/pve/firewall/cluster.fw
- Настройка pveproxy для прослушивания IPv4

### 4. VM Templates

**Плейбук:** `playbooks/create-templates.yml`  
**Роль:** `roles/vm-templates`

Создает cloud-ready шаблоны:
- **Ubuntu 22.04 LTS** (ID: 9000)
  - Cloud-init образ с jammy-server-cloudimg-amd64.img
  - 2 CPU, 2GB RAM, 20GB диск
- **Debian 12** (ID: 9001)
  - Cloud-init образ с debian-12-generic-amd64.qcow2
  - 2 CPU, 2GB RAM, 20GB диск

Особенности:
- Автоматическая загрузка cloud образов
- Настройка cloud-init (пользователь: admin, пароль: changeme123)
- Поддержка SSH ключей через cloud-init
- Конвертация в шаблоны после настройки

### 5. VM Deployment

**Плейбук:** `playbooks/deploy-vms.yml`  
**Роль:** `roles/vm-deploy`

Развертывает VM из шаблонов:
- **web-server-01** (ID: 101)
  - 4GB RAM, 2 CPU, 50GB диск
  - Сеть: vmbr1 (10.0.1.10/24)
- **db-server-01** (ID: 102)  
  - 8GB RAM, 4 CPU, 100GB диск
  - Сеть: vmbr2 (10.0.2.10/24)

Особенности:
- Клонирование из шаблонов
- Настройка сети через cloud-init
- Установка тегов для группировки
- Опциональный автозапуск после создания

## Важные файлы конфигурации

### inventory/hosts.yml
```yaml
all:
  children:
    proxmox:
      hosts:
        pve-dev-01:
          ansible_host: 65.109.114.18
          ansible_user: root
          pve_node: pve-dev-01
```

### group_vars/proxmox.yml
Содержит основные переменные:
- Сетевая конфигурация (IP, интерфейсы, бриджи)
- LVM настройки (VG, размеры томов)
- NAT сети для маршрутизации
- Правила firewall

### group_vars/all.yml
Глобальные переменные:
- Определения шаблонов VM
- Конфигурации развертываемых VM
- Cloud-init настройки
- Сетевые параметры по умолчанию

## Управление доступом

### Добавление нового администратора
Отредактируйте `playbooks/setup-firewall.yml`, добавив IP в секцию IPSET management:
```
[IPSET management]
65.109.114.0/26
95.217.183.78      # Текущий администратор
YOUR.NEW.IP.HERE   # Новый IP
10.0.1.0/24
10.0.2.0/24
```

### Временное добавление IP
```bash
ansible pve-dev-01 -m shell -a "ipset add PVEFW-0-management-v4 YOUR.IP.HERE"
```

## Решение проблем

### Потеря доступа после настройки firewall
1. Подключитесь через консоль провайдера
2. Добавьте свой IP: `ipset add PVEFW-0-management-v4 YOUR.IP`
3. Или временно отключите: `systemctl stop pve-firewall`

### Proxmox Web UI недоступен
Проверьте, что pveproxy слушает IPv4:
```bash
netstat -tlnp | grep 8006
# Должно показать: 0.0.0.0:8006
```

### NAT не работает для VM
Проверьте правила iptables:
```bash
iptables -t nat -L POSTROUTING -n -v
# Должны быть правила MASQUERADE для 10.0.1.0/24 и 10.0.2.0/24
```

## Следующие шаги

1. **Создание дополнительных шаблонов**
   - Добавить шаблоны в `group_vars/all.yml`
   - Запустить `ansible-playbook playbooks/create-templates.yml`

2. **Развертывание новых VM**
   - Определить VM в `group_vars/all.yml`
   - Запустить `ansible-playbook playbooks/deploy-vms.yml`

3. **Настройка бэкапов**
   - Настроить расписание в Proxmox
   - Добавить внешнее хранилище

4. **Мониторинг**
   - Установить Prometheus node exporter
   - Настроить алерты

## Безопасность

- Firewall настроен в режиме whitelist (DROP по умолчанию)
- SSH и Web UI доступны только с разрешенных IP
- Внутренние сети изолированы с NAT
- Все изменения создают бэкапы оригинальных файлов
- Использует iptables-legacy для совместимости с Proxmox