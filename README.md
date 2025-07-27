# Proxmox Infrastructure with Ansible

Автоматизация настройки Proxmox VE сервера с помощью Ansible.

## Структура проекта

```
proxmox-infrastructure/
├── inventory/hosts.yml          # Инвентарь серверов
├── group_vars/
│   ├── all.yml                  # Глобальные переменные
│   └── proxmox.yml              # Переменные для группы proxmox
├── playbooks/
│   ├── site.yml                 # Главный плейбук
│   ├── setup-storage.yml        # Настройка LVM дисков
│   ├── setup-network.yml        # Настройка сетевых бриджей
│   ├── setup-firewall.yml       # Настройка файрвола
│   ├── create-templates.yml     # Создание шаблонов VM
│   └── deploy-vms.yml           # Развертывание VM
├── roles/
│   ├── vm-templates/            # Роль создания шаблонов
│   └── vm-deploy/               # Роль развертывания VM
├── files/
│   └── interfaces.j2            # Шаблон сетевой конфигурации
├── docs/
│   └── proxmox-ansible-setup.md # Детальная документация
└── ansible.cfg                  # Конфигурация Ansible
```

## Требования

- Ansible >= 2.9
- Доступ по SSH к Proxmox серверу
- Proxmox VE уже установлен

## Использование

### Проверка подключения
```bash
cd proxmox-infrastructure
ansible proxmox -m ping
```

### Полная настройка
```bash
ansible-playbook playbooks/site.yml
```

### Отдельные компоненты
```bash
# Только диски
ansible-playbook playbooks/setup-storage.yml

# Только сеть
ansible-playbook playbooks/setup-network.yml  

# Только файрвол
ansible-playbook playbooks/setup-firewall.yml

# Создание шаблонов VM
ansible-playbook playbooks/create-templates.yml

# Развертывание VM
ansible-playbook playbooks/deploy-vms.yml
```

## Что настраивается

### Диски (LVM)
- vm-data: 800GB для данных VM
- vm-images: 500GB для образов и шаблонов  
- vm-backup: 400GB для бэкапов

### Сеть
- vmbr0: Публичный бридж (привязан к enp6s0)
- vmbr1: Приватная сеть 10.0.1.0/24 с NAT
- vmbr2: DMZ сеть 10.0.2.0/24 с NAT

### Файрвол
- Использует встроенный pve-firewall (iptables-legacy)
- SSH (22), Proxmox Web UI (8006), VNC (5900-5999)
- IPSet management для управления доступом
- NAT для приватных сетей
- Политика по умолчанию: DROP для входящих, ACCEPT для исходящих

### Шаблоны VM
- Ubuntu 22.04 LTS (cloud-init ready)
- Debian 12 (cloud-init ready)
- Автоматическая загрузка cloud образов
- Настройка cloud-init (пользователи, SSH ключи)

### VM для развертывания
- web-server-01: 4GB RAM, 2 CPU, 50GB диск (vmbr1)
- db-server-01: 8GB RAM, 4 CPU, 100GB диск (vmbr2)

## Важные настройки

### Управление доступом
Для добавления нового IP в management доступ, отредактируйте `group_vars/proxmox.yml`:
```yaml
# Добавьте IP в раздел [IPSET management] в firewall правилах
```

### pveproxy настройки
Proxmox Web UI настроен на прослушивание IPv4 (`0.0.0.0:8006`) через `/etc/default/pveproxy`.

## Примечания

- RAID resync должен завершиться перед активным использованием
- После настройки сети может потребоваться перезагрузка
- Все конфигурации создают бэкапы оригинальных файлов
- Firewall использует pve-firewall с iptables-legacy (не nftables)