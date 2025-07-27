# Proxmox Infrastructure with Ansible

Автоматизация настройки Proxmox VE сервера с помощью Ansible.

## Структура проекта

```
proxmox-infrastructure/
├── inventory/hosts.yml          # Инвентарь серверов
├── group_vars/proxmox.yml       # Переменные для группы proxmox
├── playbooks/
│   ├── site.yml                 # Главный плейбук
│   ├── setup-storage.yml        # Настройка LVM дисков
│   ├── setup-network.yml        # Настройка сетевых бриджей
│   └── setup-firewall.yml       # Настройка файрвола
├── files/
│   └── interfaces.j2            # Шаблон сетевой конфигурации
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
- SSH (22), Proxmox Web UI (8006), VNC (5900-5999)
- NAT для приватных сетей
- Базовые правила безопасности

## Примечания

- RAID resync должен завершиться перед активным использованием
- После настройки сети потребуется перезагрузка
- Все конфигурации создают бэкапы оригинальных файлов