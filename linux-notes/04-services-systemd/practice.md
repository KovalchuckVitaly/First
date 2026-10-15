# Module 04 — Services & systemd
# Practice & Troubleshooting

## 1. Basic Service Management

### Проверка состояния сервиса

```bash
systemctl status nginx.service
# Показывает состояние сервиса и последние сообщения журнала.
```

Пример:

```text
● nginx.service - A high performance web server
     Loaded: loaded (...; enabled)
     Active: active (running)
   Main PID: 1234 (nginx)
```

Важно различать:

- `active` — сервис сейчас работает.
- `enabled` — сервис настроен на автозапуск.

### Управление сервисом

```bash
sudo systemctl start nginx.service
# Запускает сервис.
```

```bash
sudo systemctl stop nginx.service
# Останавливает сервис.
```

```bash
sudo systemctl restart nginx.service
# Перезапускает сервис.
```

```bash
sudo systemctl reload nginx.service
# Запрашивает перечитывание конфигурации без полного перезапуска, если поддерживается.
```

### Проверка автозапуска

```bash
systemctl is-enabled nginx.service
# Проверяет состояние автозапуска.
```

```bash
sudo systemctl enable nginx.service
# Включает автозапуск.
```

```bash
sudo systemctl disable nginx.service
# Отключает автозапуск.
```

Важно:

`disable` не останавливает уже работающий сервис.

`enable` не запускает сервис немедленно.

---

## 2. Работа с Unit Files

### Просмотр конфигурации сервиса

```bash
systemctl cat nginx.service
# Показывает unit-файл и drop-in overrides.
```

Пример собственного unit-файла:

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

### Основные параметры

| Параметр | Назначение |
|---|---|
| `Description` | Описание сервиса |
| `After` | Порядок запуска |
| `Type` | Тип сервиса |
| `User` | Пользователь процесса |
| `Group` | Группа процесса |
| `WorkingDirectory` | Рабочая директория |
| `ExecStart` | Команда запуска |
| `Restart` | Политика перезапуска |
| `RestartSec` | Задержка перед перезапуском |
| `WantedBy` | Подключение unit к target при enable |

После изменения unit-файла:

```bash
sudo systemctl daemon-reload
# Перечитывает конфигурацию unit-файлов systemd.
```

Если изменились параметры запуска приложения:

```bash
sudo systemctl restart app.service
# Перезапускает сервис с обновлёнными параметрами.
```

---

# Troubleshooting Cases

Все кейсы ниже основаны на учебных сценариях, которые мы разбирали во время занятий.

---

## 3. Case №1 — payments-api.service: Permission Denied

### Ситуация

На сервере работает приложение `payments-api`.

После запуска сервис переходит в состояние `failed`.

### Симптомы

```bash
systemctl status payments-api.service
# Проверяем состояние сервиса.
```

Результат:

```text
● payments-api.service - Payments API
     Active: failed (Result: exit-code)
```

### Диагностика

Первым делом проверяем журнал:

```bash
journalctl -u payments-api.service -n 50 --no-pager
# Показывает последние 50 записей журнала сервиса.
```

В журнале обнаружена ошибка доступа к конфигурации:

```text
Permission denied: /opt/payments-api/config/app.conf
```

Проверяем конфигурацию unit:

```bash
systemctl cat payments-api.service
# Показывает настройки запуска сервиса.
```

В unit-файле указано:

```ini
[Service]
User=payments-api
Group=payments
```

Это означает, что приложение запускается от имени пользователя `payments-api`.

Проверяем путь к конфигурационному файлу:

```bash
namei -l /opt/payments-api/config/app.conf
# Показывает permissions каждого компонента пути.
```

Обнаруживаем:

```text
drwx------ root root config
```

### Root Cause

Каталог:

```text
/opt/payments-api/config
```

имеет права:

```text
700 root:root
```

Следовательно, пользователь `payments-api` не может пройти через этот каталог.

Даже если у самого `app.conf` правильные permissions, доступ к файлу будет заблокирован.

### Исправление

Необходимо предоставить сервисному пользователю минимально необходимый доступ к конфигурации.

Возможные варианты:

- Настроить подходящую группу и group permissions.
- Использовать ACL для конкретного сервисного пользователя.
- Исправить владельца каталога, если это соответствует принятой модели доступа.

Не следует автоматически использовать:

```bash
sudo chmod 777 /opt/payments-api/config
# Небезопасный вариант: предоставляет всем пользователям полный доступ.
```

Конкретное исправление зависит от требований безопасности приложения.

### Проверка

После исправления permissions:

```bash
sudo systemctl restart payments-api.service
# Повторно запускает приложение.
```

```bash
systemctl status payments-api.service
# Проверяет состояние сервиса.
```

```bash
journalctl -u payments-api.service -n 20 --no-pager
# Проверяет отсутствие повторных ошибок доступа.
```

### Вывод

При ошибке `Permission denied` необходимо проверять не только сам файл, но и все родительские каталоги.

---

## 4. Case №2 — reporting.service: Logrotate Permission Issue

### Ситуация

На production-сервере после ночной ротации логов приложение перестало запускаться.

Пользователи получают HTTP 502.

### Симптомы

```bash
systemctl status reporting.service
# Проверяем состояние сервиса.
```

Результат:

```text
Active: failed (Result: exit-code)
```

### Диагностика

Проверяем журнал:

```bash
journalctl -u reporting.service -n 50 --no-pager
# Просматриваем ошибки сервиса.
```

Ошибка:

```text
Permission denied: /var/log/reporting/application.log
```

Проверяем права:

```bash
namei -l /var/log/reporting/application.log
# Проверяем permissions всех компонентов пути.
```

Результат:

```text
drwxr-x--- root reporting reporting
-rw-r----- root reporting application.log
```

Проверяем unit:

```bash
systemctl cat reporting.service
# Определяем пользователя и группу процесса.
```

```ini
[Service]
User=reporting
Group=reporting
```

Приложение работает от имени `reporting`, но файл журнала принадлежит `root`.

Права файла:

```text
640 root:reporting
```

означают:

- root — чтение и запись.
- reporting — только чтение.
- others — нет доступа.

Приложение не может записывать сообщения в журнал.

### Поиск Root Cause

Проверяем содержимое каталога:

```bash
ls -l /var/log/reporting/
# Показывает владельцев и время изменения файлов журнала.
```

Обнаруживаем:

```text
application.log      root:reporting
application.log.1    reporting:reporting
```

Это указывает на возможную проблему с ротацией логов.

Проверяем конфигурацию:

```bash
cat /etc/logrotate.d/reporting
# Показывает правила ротации журнала приложения.
```

Конфигурация:

```text
/var/log/reporting/application.log {
    daily
    rotate 7
    missingok
    notifempty
    compress
    delaycompress
    create 0640 root reporting
}
```

Проблемный параметр:

```text
create 0640 root reporting
```

После ротации новый файл создаётся с владельцем `root` и группой `reporting`.

Но приложение работает от имени `reporting` и не имеет права записи.

### Root Cause

Неправильная конфигурация logrotate создаёт файл журнала с неподходящим владельцем.

### Временное восстановление

```bash
sudo chown reporting:reporting /var/log/reporting/application.log
# Передаёт файл журнала сервисному пользователю.
```

```bash
sudo systemctl restart reporting.service
# Перезапускает приложение.
```

```bash
systemctl status reporting.service
# Проверяет, запустился ли сервис.
```

### Постоянное исправление

В файле:

```text
/etc/logrotate.d/reporting
```

изменяем:

```text
create 0640 root reporting
```

на:

```text
create 0640 reporting reporting
```

Теперь новые файлы журнала будут принадлежать сервисному пользователю.

Проверяем конфигурацию:

```bash
sudo logrotate -d /etc/logrotate.d/reporting
# Выполняет диагностический прогон без фактической ротации.
```

### Проверка приложения

```bash
curl -i http://127.0.0.1:8080/health
# Проверяет HTTP-ответ локального приложения.
```

Ожидаемый результат:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"UP"}
```

Важно: локальный HTTP 200 ещё не доказывает, что внешний HTTP 502 исчез.

При необходимости дополнительно проверяем reverse proxy.

### Вывод

Недостаточно исправить permissions существующего файла.

Нужно определить, какой механизм создал неправильные permissions, иначе проблема повторится.

---

## 5. Case №3 — api.service: YAML Configuration Error

### Ситуация

После изменения конфигурации приложение `api.service` не запускается.

### Симптомы

```bash
systemctl status api.service
# Проверяем состояние сервиса.
```

Результат:

```text
Active: failed (Result: exit-code)
```

В журнале:

```bash
journalctl -u api.service
# Показывает журнал приложения.
```

Обнаружена ошибка:

```text
yaml.parser.ParserError
in "/etc/api/config.yaml", line 18, column 5
expected <block end>, but found '<block mapping start>'
```

### Первичная гипотеза

Приложение не смогло разобрать YAML-конфигурацию.

Ошибка может находиться не непосредственно на указанной строке, а выше.

### Проверка файла

```bash
less /etc/api/config.yaml
# Просматривает конфигурацию приложения.
```

Проверяем специальные символы:

```bash
sed -n '1,22l' /etc/api/config.yaml
# Показывает строки 1–22 и визуализирует специальные символы.
```

Явных проблем с отступами не обнаружено.

### Дополнительная диагностика

Возникает противоречие:

- Журнал указывает на ошибку YAML.
- Текущее содержимое файла выглядит корректно.

Проверяем время изменения:

```bash
ls -l /etc/api/config.yaml
# Показывает время последнего изменения содержимого файла.
```

Результат:

```text
-rw-r----- 1 root api 487 Oct 14 21:52 /etc/api/config.yaml
```

Сопоставляем события:

| Событие | Время |
|---|---|
| Неудачный запуск приложения | 21:47 |
| Изменение конфигурации | 21:52 |

### Root Cause

Приложение завершилось при чтении предыдущей версии конфигурации.

После неудачного запуска файл был изменён, но приложение ещё не запускалось с новой конфигурацией.

Важно: по одним только временным меткам нельзя установить, кто изменил файл и какие именно изменения были внесены.

### Восстановление

```bash
sudo systemctl restart api.service
# Запускает приложение заново с текущей конфигурацией.
```

Проверяем:

```bash
systemctl status api.service
# Проверяет состояние приложения.
```

Результат:

```text
Active: active (running)
```

Журнал сообщает:

```text
Configuration loaded successfully
PostgreSQL connection established
Redis connection established
HTTP server listening on 0.0.0.0:8080
```

### Проверка HTTP

```bash
curl -i http://127.0.0.1:8080/health
# Отправляет HTTP-запрос к локальному API.
```

Результат:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"UP"}
```

### Вывод

При диагностике важно учитывать хронологию событий.

Ошибки в журнале могут относиться к предыдущей версии конфигурации.

Необходимо сопоставлять:

- Время возникновения ошибки.
- Время изменения файла.
- Время последнего запуска приложения.

---

## 6. General Troubleshooting Checklist

Если systemd-сервис не работает:

### 1. Проверить состояние

```bash
systemctl status app.service
# Показывает состояние сервиса.
```

### 2. Изучить журнал

```bash
journalctl -u app.service -n 50 --no-pager
# Показывает последние 50 записей журнала.
```

### 3. Проверить unit

```bash
systemctl cat app.service
# Показывает конфигурацию unit.
```

### 4. Проверить пользователя процесса

```bash
systemctl show app.service -p User -p Group
# Показывает настройки пользователя и группы.
```

### 5. Проверить permissions

```bash
namei -l /path/to/config
# Проверяет права всех компонентов пути.
```

### 6. Проверить конфигурацию приложения

```bash
stat /path/to/config
# Показывает метаданные файла и время его изменения.
```

### 7. Исправить подтверждённую причину

Внести минимально необходимое изменение.

### 8. Проверить восстановление

```bash
systemctl status app.service
# Проверяет состояние сервиса.
```

```bash
journalctl -u app.service -n 20 --no-pager
# Проверяет последние сообщения журнала.
```

```bash
curl -i http://127.0.0.1:8080/health
# Проверяет HTTP-ответ приложения, если endpoint существует.
```

---

## 7. Основные выводы

1. `systemctl status` — отправная точка диагностики.
2. `journalctl` помогает установить причину неисправности.
3. `systemctl cat` показывает, как systemd запускает приложение.
4. `Permission denied` требует проверки пользователя процесса и permissions.
5. При проблемах с журналами необходимо учитывать logrotate.
6. Ошибки конфигурации следует сопоставлять со временем изменения файлов.
7. `active (running)` не гарантирует полноценную работоспособность приложения.
8. После исправления необходимо проверить функциональность сервиса.

Главный принцип:

> Diagnose → Identify Root Cause → Apply Minimal Fix → Verify.