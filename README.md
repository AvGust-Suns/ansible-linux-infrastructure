# Ansible Linux Infrastructure Automation

Автоматизация настройки и управления парком Linux-серверов (Debian 12) с использованием Ansible. Проект охватывает полный цикл: от базовой конфигурации и харденинга до развертывания веб-сервисов и комплексного мониторинга (Prometheus + Grafana + Alertmanager).

## Инфраструктура

В проекте используется 1 control-узел и 3 управляемых сервера:

| Хост | IP | Роль |
|---|---|---|
| ansible-control | 192.168.56.12 | Control Node (Ansible) |
| ansible-node1 | 192.168.56.13 | Web Server (Nginx, Node Exporter) |
| ansible-node2 | 192.168.56.14 | Docker Host (Node Exporter) |
| ansible-node3 | 192.168.56.15 | Infrastructure (Prometheus, Grafana, Alertmanager, Blackbox) |

## Реализованный функционал

### Безопасность и базовая настройка
* **Роль `common`**: обновление пакетов, установка утилит, настройка часового пояса.
* **Роль `ssh_hardening`**: отключение root-логина и входа по паролю (только ключи).
* **Роль `firewall`**: настройка UFW (закрыто всё, кроме SSH и HTTP).
* **Роль `security_updates`**: автоматическое обновление безопасности через `unattended-upgrades`.
* **Роль `users`**: создание групп (`infra_admins`, `developers`), настройка sudoers и доступ по SSH-ключам для администраторов.
* **Роль `time_sync`**: настройка синхронизации времени (Chrony).

### Сервисы
* **Nginx** (ansible-node1): развертывание веб-сервера с кастомной страницей (Jinja2).
* **Docker** (ansible-node2): установка Docker и Docker Compose.

### Мониторинг и Алертинг
Развернут полноценный стек мониторинга на `ansible-node3`:
* **Prometheus**: сбор метрик, кастомные правила алертов (CPU > 85%, RAM > 90%, Disk < 10%, падение хоста).
* **Grafana**: визуализация (настроен провижининг дашбордов, доступ только с `127.0.0.1`).
* **Node Exporter**: установлен на всех нодах для сбора системных метрик.
* **Blackbox Exporter**: проверка доступности HTTP-эндпоинтов (проверяет живость Nginx).
* **Alertmanager**: маршрутизация алертов (слушает только локальный интерфейс).

### Резервное копирование
* Плейбуки `backup-configs.yml` и `restore-config.yml` для сбора и восстановления конфигураций (`/etc/nginx`, `/etc/ssh`, `/etc/ufw`).
* В бэкапах создается `SHA256SUMS` для контроля целостности.
* Автоматическое расписание бэкапов (`schedule-backups.yml`).

### CI/CD
* Настроен GitHub Actions (`.github/workflows/ansible-ci.yml`).
* При каждом пуше запускается `yamllint`, `ansible-lint` и проверка синтаксиса всех плейбуков.

## Использование

**1. Установка зависимостей:**
```bash
ansible-galaxy collection install -r requirements.yml
