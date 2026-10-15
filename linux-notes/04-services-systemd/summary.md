# Module 04 — Services & systemd
## Quick Reference

### 1. Основы systemd

`systemd` — система инициализации и менеджер служб Linux.

Обычно работает как процесс с PID 1.

Основные понятия:

| Термин | Значение |
|---|---|
| Unit | Объект, которым управляет systemd |
| Service | Unit для управления сервисом |
| Target | Unit для группировки других units |
| Socket | Unit для работы с сокетами и активации сервисов |
| Timer | Unit для запуска задач по расписанию |
| PID 1 | Первый процесс пользовательского пространства, обычно systemd |

---

### 2. Управление сервисами

```bash
systemctl status nginx.service
# Проверить состояние сервиса.

sudo systemctl start nginx.service
# Запустить сервис.

sudo systemctl stop nginx.service
# Остановить сервис.

sudo systemctl restart nginx.service
# Перезапустить сервис.

sudo systemctl reload nginx.service
# Перечитать конфигурацию приложения, если поддерживается.
```

### 3. Active vs Enabled

**Active** — текущее состояние сервиса.

**Enabled** — настройка автозапуска.

```bash
systemctl is-active nginx.service
# Проверить текущее состояние.

systemctl is-enabled nginx.service
# Проверить автозапуск.

sudo systemctl enable nginx.service
# Включить автозапуск.

sudo systemctl disable nginx.service
# Отключить автозапуск.

sudo systemctl enable --now nginx.service
# Включить автозапуск и сразу запустить сервис.
```

Важно:

- `enable` ≠ `start`
- `disable` ≠ `stop`
- `disabled` не запрещает ручной запуск.
- `masked` блокирует запуск unit через systemd.

---

### 4. Структура Unit File

Пример:

```ini
[Unit]
Description=Example Application
After=network.target

[Service]
Type=simple
User=app
Group=app
WorkingDirectory=/opt/app
ExecStart=/opt/app/bin/server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Основные параметры:

| Параметр | Назначение |
|---|---|
| `Description` | Описание |
| `After` | Порядок запуска |
| `User` | Пользователь процесса |
| `Group` | Группа процесса |
| `WorkingDirectory` | Рабочая директория |
| `ExecStart` | Команда запуска |
| `Restart` | Политика перезапуска |
| `RestartSec` | Задержка перед перезапуском |
| `WantedBy` | Подключение к target при enable |

---

### 5. Типы сервисов

| Type | Поведение |
|---|---|
| `simple` | Запускает основной процесс без отдельного подтверждения готовности |
| `forking` | Приложение создаёт дочерний процесс, исходный завершается |
| `oneshot` | Выполняет действие и завершается |

Для `oneshot`:

```ini
RemainAfterExit=yes
```

После успешного завершения сервис может оставаться в состоянии `active (exited)`.

---

### 6. Restart vs Reload vs Daemon-reload

| Команда | Назначение |
|---|---|
| `systemctl restart` | Перезапуск приложения |
| `systemctl reload` | Перечитывание конфигурации приложения |
| `systemctl daemon-reload` | Перечитывание unit-файлов systemd |

После изменения unit-файла:

```bash
sudo systemctl daemon-reload
# Перечитать unit-файлы.

sudo systemctl restart app.service
# Перезапустить приложение с новыми параметрами.
```

---

### 7. Автоматический перезапуск

```ini
Restart=on-failure
RestartSec=5
```

`Restart=on-failure` — автоматический перезапуск при ошибке.

`RestartSec=5` — задержка 5 секунд.

Важно:

`Restart=always` не отменяет намеренную остановку через `systemctl stop`.

---

### 8. Зависимости

| Параметр | Значение |
|---|---|
| `After=` | Порядок запуска |
| `Before=` | Обратный порядок запуска |
| `Wants=` | Слабая зависимость |
| `Requires=` | Более сильная зависимость |

Важно:

`After=network.target` не гарантирует готовность сети.

`network-online.target` используется для ожидания готовности сети при соответствующей настройке.

---

### 9. Journalctl

```bash
journalctl -u nginx.service
# Журнал сервиса.

journalctl -u nginx.service -n 50
# Последние 50 записей.

journalctl -u nginx.service -f
# Новые сообщения в реальном времени.

journalctl -u nginx.service -b
# Журнал сервиса за текущую загрузку.

journalctl -u nginx.service --since "1 hour ago"
# Журнал за последний час.

journalctl --list-boots
# Список доступных загрузок.

journalctl -b -1
# Журнал предыдущей загрузки, если сохранился.
```

---

### 10. Troubleshooting Checklist

**Если сервис не работает:**

1. Проверить состояние.
2. Изучить журнал.
3. Проверить unit-файл.
4. Проверить пользователя процесса.
5. Проверить permissions.
6. Проверить конфигурацию приложения.
7. Подтвердить root cause.
8. Внести минимальное исправление.
9. Проверить восстановление.

Основные команды:

```bash
systemctl status app.service
# Состояние сервиса.

journalctl -u app.service -n 50 --no-pager
# Последние сообщения журнала.

systemctl cat app.service
# Конфигурация unit.

systemctl show app.service -p User -p Group
# Пользователь и группа сервиса.

namei -l /opt/app/config/settings.conf
# Права всех компонентов пути.

stat /etc/app/config.yaml
# Метаданные и время изменения файла.

curl -i http://127.0.0.1:8080/health
# Проверка HTTP endpoint, если он существует.
```

---

### 11. Типичные ошибки

**Ошибка 1:** `active` означает `enabled`.

Нет. Это независимые характеристики.

**Ошибка 2:** `daemon-reload` перезапускает приложение.

Нет. Он перечитывает unit-файлы.

**Ошибка 3:** `disable` полностью запрещает запуск сервиса.

Нет. Сервис можно запустить вручную.

**Ошибка 4:** `Permission denied` нужно исправлять через `chmod 777`.

Нет. Нужно определить необходимые права.

**Ошибка 5:** `active (running)` гарантирует работоспособность приложения.

Нет. Нужно проверить функциональность приложения.

**Ошибка 6:** Достаточно исправить владельца файла журнала.

Не всегда. Если logrotate создаёт файл с неправильным владельцем, проблема повторится.

---

### 12. Главное правило

> Diagnose → Identify Root Cause → Apply Minimal Fix → Verify.

**Systemd показывает состояние процесса, но полноценную работоспособность приложения необходимо проверять отдельно.**