# Автоматизация инфраструктуры Linux на Ansible

Проект автоматизации настройки и управления небольшим парком Linux-серверов под Debian 12 с помощью Ansible. Демонстрирует конфигурирование инфраструктуры через переиспользуемые роли: аутентификация по SSH-ключам, харденинг SSH, управление фаерволом и развёртывание сервисов.

## Обзор

Инфраструктура состоит из одного control-узла Ansible и трёх управляемых серверов под Debian 12.

### Архитектура

| Хост | Management IP | Internal IP | Назначение |
|---|---|---|---|
| ansible-control | 192.168.56.12 | 10.10.10.11 | Control-узел Ansible |
| ansible-node1 | 192.168.56.13 | 10.10.10.12 | Веб-сервер |
| ansible-node2 | 192.168.56.14 | 10.10.10.13 | Docker-хост |
| ansible-node3 | 192.168.56.15 | 10.10.10.14 | Инфраструктурный узел |

Control-узел подключается к управляемым серверам по SSH с аутентификацией по публичному ключу.

### Стек

- Debian 12
- Ansible Core 2.14
- OpenSSH
- Nginx
- Docker / Docker Compose
- UFW
- YAML и Jinja2

## Структура проекта

```text
ansible-linux-automation/
├── ansible.cfg
├── .gitignore
├── inventory/
│   └── hosts.yml
├── playbooks/
│   ├── bootstrap-user.yml
│   └── site.yml
└── roles/
    ├── common/
    │   └── tasks/main.yml
    ├── nginx/
    │   ├── handlers/main.yml
    │   ├── tasks/main.yml
    │   └── templates/index.html.j2
    ├── docker/
    │   └── tasks/main.yml
    ├── ssh_hardening/
    │   ├── handlers/main.yml
    │   └── tasks/main.yml
    └── firewall/
        └── tasks/main.yml
```

## Требования

- Control-узел под Debian 12.
- Три управляемых сервера под Debian 12.
- Ansible Core на control-узле.
- SSH-доступ между control-узлом и управляемыми серверами.
- Python 3 на управляемых серверах.
- Начальный административный SSH-доступ для bootstrap-этапа.

Установка Ansible и нужной коллекции:

```bash
apt update
apt install ansible sshpass -y
ansible-galaxy collection install community.general
```

## Быстрый старт

### 1. Настройте inventory

Укажите IP-адреса и группы хостов в `inventory/hosts.yml`.

### 2. Сгенерируйте SSH-ключ

На control-узле:

```bash
ssh-keygen -t ed25519 -f /root/.ssh/ansible_ed25519
```

### 3. Создайте пользователя для управления

Используется начальный административный доступ:

```bash
ansible-playbook playbooks/bootstrap-user.yml -u root -k
```

### 4. Проверьте связность

```bash
ansible managed_nodes -m ping
```

### 5. Проверьте синтаксис playbook

```bash
ansible-playbook playbooks/site.yml --syntax-check
```

### 6. Примените конфигурацию

```bash
ansible-playbook playbooks/site.yml
```

### 7. Проверьте результат

```bash
ansible managed_nodes -m ping
ansible managed_nodes -b -m command -a "ufw status verbose"
ansible web_servers -m command -a "curl -I http://127.0.0.1"
```

## Что делают роли и playbook'и

### `bootstrap-user.yml` — пользователь для Ansible

- Создаёт выделенного пользователя `ansible`.
- Настраивает аутентификацию по SSH-ключу.
- Выдаёт права sudo.
- Настраивает sudo без пароля для автоматизации.

### `common` — базовая настройка системы

- Обновление кеша APT.
- Безопасное обновление пакетов.
- Установка базовых утилит администрирования.
- Активация SSH-сервиса.

### `nginx` — веб-сервер (на `ansible-node1`)

- Устанавливает Nginx.
- Разворачивает index-страницу на Jinja2.
- Использует Ansible facts для отображения hostname сервера.
- Гарантирует, что сервис включён и запущен.
- Через handler перезапускает Nginx при изменении контента.

### `docker` — Docker-хост (на `ansible-node2`)

- Устанавливает Docker Engine из репозитория Debian.
- Устанавливает Docker Compose.
- Гарантирует, что сервис Docker включён и запущен.

### `ssh_hardening` — харденинг SSH

- Отключает вход root по SSH.
- Отключает аутентификацию по паролю.
- Включает аутентификацию по публичному ключу.
- Отключает keyboard-interactive аутентификацию.
- Валидирует конфигурацию SSH перед применением рестарта.

### `firewall` — UFW

- Политика входящего трафика по умолчанию: deny.
- Политика исходящего трафика по умолчанию: allow.
- SSH разрешён на всех управляемых серверах.
- HTTP разрешён только на веб-сервере.
- Фаервол включается автоматически.

Используется модуль `community.general.ufw`.

> **Важно:** Docker управляет собственными правилами iptables. Один только UFW не гарантирует фильтрацию опубликованных портов контейнеров.

## Результаты проверки

Реализация протестирована на трёх виртуальных машинах под Debian 12.

| Проверка | Результат |
|---|---|
| Связность Ansible | Успешно на всех управляемых узлах |
| Развёртывание Nginx | Успешно |
| HTTP-ответ | HTTP 200 OK |
| Установка Docker | Успешно |
| Харденинг SSH | Успешно |
| Активация UFW | Успешно на всех узлах |
| SSH после включения фаервола | Успешно |
| Проверка синтаксиса playbook | Успешно |
| Тест идемпотентности | Успешно |

Повторный прогон полного playbook дал:

| Хост | OK | Changed | Failed |
|---|---:|---:|---:|
| ansible-node1 | 17 | 0 | 0 |
| ansible-node2 | 15 | 0 | 0 |
| ansible-node3 | 12 | 0 | 0 |

Нулевой `changed` подтверждает, что playbook не вносит лишних изменений, когда целевое состояние уже достигнуто.

## Планы развития

- Мониторинг: Prometheus + Node Exporter на управляемых узлах.
- Визуализация: Grafana с автоматическим провижинингом дашбордов.
- Алертинг: Alertmanager с базовыми правилами.
- Безопасность: аудит конфигурации, проверка через `ansible-lint`/`yamllint` в CI.
- Расширение автоматизации: резервное копирование и обновление пакетов по расписанию.

## Примечания

Проект разработан и протестирован в лабораторной среде на VirtualBox. Сетевые адреса — пример из лабораторной конфигурации, при развёртывании в другой среде их нужно скорректировать.
