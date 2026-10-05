# Module 03 — Practice

Практические задачи по Linux permissions, ACL, users и groups.

Главный принцип troubleshooting:

> Не исправлять проблему до того, как найдена и подтверждена её причина.

---

# 1. Shared directory и SGID

## Ситуация

Несколько разработчиков работают с файлами в общей директории:

```text
/srv/webapp
```

Пользователи входят в группу:

```text
developers
```

Проблема: файлы, созданные одним пользователем, могут получать неправильную группу или быть недоступны для записи другим разработчикам.

## Диагностика

```bash
ls -ld /srv/webapp
# Проверяем владельца, группу и permissions общей директории.
```

```bash
id alex
# Проверяем primary и supplementary groups пользователя alex.
```

```bash
id bob
# Проверяем primary и supplementary groups пользователя bob.
```

## Решение с SGID

```bash
sudo chown root:developers /srv/webapp
# Назначаем developers группой общей директории.
```

```bash
sudo chmod 2775 /srv/webapp
# 2 устанавливает SGID.
# Новые объекты внутри директории будут наследовать группу developers.
```

Проверка:

```bash
ls -ld /srv/webapp
# Проверяем наличие s в group execute позиции.
```

Пример:

```text
drwxrwsr-x root developers /srv/webapp
```

## Важный нюанс

SGID решает проблему наследования **группы**, но не гарантирует право записи.

Если пользователь работает с:

```text
umask 022
```

новый файл обычно будет:

```text
-rw-r--r--
```

Группа сможет читать файл, но не изменять его.

Для совместной работы может потребоваться подходящая `umask` или Default ACL.

---

# 2. Permission denied из-за директории в пути

## Ситуация

Пользователь Bob состоит в группе `developers`.

Файл:

```text
/opt/app/config/settings.conf
```

имеет:

```text
appuser developers 640
```

То есть группа имеет право читать файл.

Но Bob получает:

```text
Permission denied
```

## Диагностика

Проверяем пользователя:

```bash
id bob
# Проверяем, действительно ли bob входит в developers.
```

Проверяем весь путь:

```bash
namei -l /opt/app/config/settings.conf
# Показывает permissions каждого компонента пути.
```

Обнаруживаем:

```text
/opt/app/config
```

с правами, где у группы отсутствует `x`.

## Причина

Чтобы обратиться к файлу внутри директории, пользователь должен иметь `x` на каждом компоненте пути.

Наличие `r` на самом файле недостаточно.

Также Linux не переключится на `others`, если пользователь уже подходит под group class.

## Исправление

```bash
sudo chmod g+x /opt/app/config
# Добавляем группе право прохода через директорию.
```

Проверяем именно от имени Bob:

```bash
sudo -u bob cat /opt/app/config/settings.conf
# Проверяем, может ли bob реально прочитать файл после исправления.
```

---

# 3. logrotate изменил владельца файла

## Ситуация

Приложение работает от пользователя:

```text
webapp
```

и записывает лог:

```text
/var/log/webapp/app.log
```

После logrotate приложение перестало записывать лог.

## Диагностика

Проверяем каталог и файлы:

```bash
ls -ld /var/log/webapp
# Проверяем владельца и permissions каталога логов.
```

```bash
ls -l /var/log/webapp/
# Сравниваем текущий и ротированный лог.
```

Обнаруживаем:

```text
app.log     root:adm
app.log.1   webapp:webapp
```

Проверяем конфигурацию logrotate.

Например, там присутствует:

```text
create 0640 root adm
```

## Root cause

После ротации logrotate создаёт новый `app.log`:

```text
root:adm 0640
```

Процесс `webapp` больше не имеет права записи.

## Исправление текущего файла

```bash
sudo chown webapp:webapp /var/log/webapp/app.log
# Возвращаем текущий лог приложению.
```

## Исправление причины

В конфигурации logrotate:

```text
create 0640 webapp webapp
```

Теперь новые логи будут создаваться с правильным владельцем.

## Проверка

```bash
sudo -u webapp sh -c 'echo "test" >> /var/log/webapp/app.log'
# Проверяем возможность записи именно от имени webapp.
```

Главный вывод:

> Недостаточно исправить текущий файл. Нужно найти механизм, который создал неправильные permissions или ownership.

---

# 4. ACL mask ограничивает права

## Ситуация

Для Bob настроен ACL:

```text
user:bob:rw-
```

Но Bob может только читать файл.

## Диагностика

```bash
getfacl file.txt
# Показываем ACL и effective permissions.
```

Получаем:

```text
user:bob:rw-        #effective:r--
mask::r--
```

## Root cause

ACL entry разрешает:

```text
rw-
```

но mask разрешает максимум:

```text
r--
```

Поэтому эффективные права Bob:

```text
r--
```

## Исправление

```bash
setfacl -m m::rw file.txt
# Расширяем ACL mask до read + write.
```

## Проверка

```bash
getfacl file.txt
# Проверяем ACL после изменения mask.
```

```bash
sudo -u bob sh -c 'echo "test" >> file.txt'
# Проверяем реальное право записи от имени bob.
```

---

# 5. Shared directory: SGID + Default ACL

Для общей директории необходимо:

1. сохранять общую группу;
2. автоматически выдавать группе нужные permissions.

SGID отвечает за первое.

Default ACL может использоваться для второго.

```bash
sudo chmod g+s /srv/project
# Устанавливаем SGID для наследования group ownership.
```

```bash
sudo setfacl -m d:g:developers:rwx /srv/project
# Добавляем Default ACL для developers.
# Новые объекты получают ACL entry для этой группы.
```

Проверка:

```bash
getfacl /srv/project
# Проверяем Access ACL и Default ACL директории.
```

Важно:

> Default ACL является шаблоном для новых объектов и не предоставляет доступ к самой родительской директории.

---

# 6. Создание пользователя deploy

## Задача

Создать пользователя:

```text
deploy
```

с:

- home `/home/deploy`;
- shell `/bin/bash`;
- primary group `developers`;
- supplementary group `docker`.

## Проверка групп

```bash
getent group developers
# Проверяем существование developers.
```

```bash
getent group docker
# Проверяем существование docker.
```

## Создание

```bash
sudo useradd -m -s /bin/bash -g developers -G docker deploy
# -m создаёт home directory.
# -s задаёт login shell.
# -g задаёт primary group.
# -G задаёт supplementary groups.
```

## Проверка

```bash
getent passwd deploy
# Проверяем запись пользователя через NSS.
```

```bash
id deploy
# Проверяем UID, primary GID и supplementary groups.
```

```bash
ls -ld /home/deploy
# Проверяем существование и ownership домашней директории.
```

---

# 7. Опасность usermod -G

Допустим, deploy уже входит в:

```text
docker
```

и его нужно дополнительно добавить в:

```text
monitoring
```

Неправильный вариант:

```bash
sudo usermod -G monitoring deploy
# ОПАСНО: заменяет текущий список supplementary groups.
# deploy может потерять membership в docker.
```

Правильно:

```bash
sudo usermod -aG monitoring deploy
# -a означает append.
# monitoring добавляется к существующим supplementary groups.
```

Проверка:

```bash
id deploy
# Проверяем полный список групп после изменения.
```

---

# 8. Изменение групп и старый shell

## Ситуация

Пользователь `deploy` находится в группе `monitoring`.

У него уже открыта login session.

Администратор удаляет его из группы.

Проверка из административной сессии:

```bash
id deploy
# Показывает актуальное членство пользователя согласно системным источникам.
```

Но внутри старого shell пользователя:

```bash
id
# Показывает credentials текущего процесса.
# Старый shell может продолжать содержать старую supplementary group.
```

## Причина

Изменение `/etc/group` или другого источника пользователей не переписывает credentials уже запущенных процессов.

Новая login session получает новый список групп.

---

# 9. System account для приложения

## Задача

Приложению `myapp` нужен отдельный пользователь.

Интерактивный login не требуется.

Создание:

```bash
sudo useradd -r -M -s /usr/sbin/nologin myapp
# -r создаёт system account.
# -M не создаёт home directory.
# nologin запрещает обычную интерактивную login shell.
```

Проверка:

```bash
getent passwd myapp
# Проверяем UID, GID, home field и shell системного пользователя.
```

```bash
id myapp
# Проверяем UID/GID пользователя.
```

При этом процесс всё равно можно запустить с его UID:

```bash
sudo -u myapp id
# Демонстрирует, что nologin не запрещает запуск процессов от имени UID.
```

---

# 10. Директория данных system account

Для приложения создаём:

```text
/var/lib/myapp
```

```bash
sudo mkdir /var/lib/myapp
# Создаём каталог данных приложения.
```

```bash
sudo chown myapp:myapp /var/lib/myapp
# Передаём каталог сервисному пользователю.
```

```bash
sudo chmod 700 /var/lib/myapp
# Только myapp получает rwx.
```

Проверка:

```bash
sudo -u myapp touch /var/lib/myapp/test.dat
# Проверяем, что сервисный пользователь может создавать файлы.
```

Обычный пользователь не должен иметь такой возможности:

```bash
touch /var/lib/myapp/hack.txt
# Ожидаем Permission denied для пользователя без w+x на директорию.
```

---

# 11. User offboarding

## Ситуация

Необходимо удалить пользователя:

```text
alex
```

Но на сервере могут оставаться принадлежащие ему файлы.

Сначала проверяем:

```bash
id alex
# Смотрим UID, primary group и supplementary groups пользователя.
```

```bash
sudo find / -user alex 2>/dev/null
# Ищем файлы, владельцем которых является alex.
```

Обнаружен важный проектный файл:

```text
/var/www/project/file1.txt
```

Его нужно сохранить.

Передаём:

```bash
sudo chown deploy:developers /var/www/project/file1.txt
# Передаём проектный файл новому владельцу и группе.
```

Проверяем:

```bash
ls -l /var/www/project/file1.txt
# Подтверждаем новый ownership.
```

Повторяем поиск:

```bash
sudo find / -user alex 2>/dev/null
# Проверяем, какие файлы alex остались перед удалением аккаунта.
```

Удаляем аккаунт:

```bash
sudo userdel -r alex
# Удаляем пользователя, home directory и mail spool, где это применимо.
```

---

# 12. Проверка orphaned UID/GID

После удаления пользователя нельзя полагаться только на:

```text
-user alex
```

Имени `alex` уже может не существовать.

Файловая система хранит числовой UID.

Поэтому проверяем:

```bash
sudo find / -uid 1002 2>/dev/null
# Ищем файлы с бывшим UID пользователя alex.
```

Если удаляется его группа:

```bash
sudo find / -gid 1003 2>/dev/null
# Проверяем, не остались ли файлы со старым GID.
```

Только после проверки можно удалить ненужную группу:

```bash
sudo groupdel alex
# Удаляем группу alex после проверки файлов с её GID.
```

---

# 13. Универсальный алгоритм Permission denied

При проблемах с доступом:

```text
Кто выполняет операцию?
        ↓
Какой UID/GID и supplementary groups?
        ↓
Кому принадлежит объект?
        ↓
Какие обычные permissions?
        ↓
Есть ли x на всём пути?
        ↓
Есть ли ACL?
        ↓
Что разрешает ACL mask?
        ↓
Есть ли special bits?
        ↓
Найдена root cause
        ↓
Минимальное исправление
        ↓
Проверка от имени реального пользователя
```

Основной набор диагностических команд:

```bash
id username
# Проверяем identity и группы пользователя.
```

```bash
ls -ld /path
# Проверяем ownership и permissions директории.
```

```bash
namei -l /full/path/to/file
# Проверяем permissions каждого компонента пути.
```

```bash
getfacl /path/to/file
# Проверяем ACL и mask.
```

```bash
stat /path/to/file
# Получаем подробные метаданные объекта.
```

```bash
sudo -u username command
# Выполняем проверку от имени пользователя,
# у которого реально возникала проблема.
```

Главное правило:

> Diagnose → confirm root cause → minimal fix → verify.