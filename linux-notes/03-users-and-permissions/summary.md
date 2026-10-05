# Module 03 — Users & Permissions — Summary

Краткая шпаргалка по permissions, ACL, users и groups.

---

## 1. Basic permissions

```text
r = 4
w = 2
x = 1
```

Классы:

```text
u = owner
g = group
o = others
```

Пример:

```text
-rw-r----- = 640

owner  → rw-
group  → r--
others → ---
```

```bash
chmod 640 file.txt
# Устанавливает owner=rw, group=r, others=---.
```

```bash
chmod g+w file.txt
# Добавляет группе право записи.
```

---

## 2. File vs Directory

### File

```text
r → читать
w → изменять
x → выполнять
```

### Directory

```text
r → получить список имён
w → создавать / удалять / переименовывать
x → проходить через директорию и обращаться к объектам
```

Для доступа:

```text
/a/b/file
```

нужен `x` на всех директориях пути.

```bash
namei -l /a/b/file
# Показывает permissions каждого компонента пути.
```

Удаление файла зависит в первую очередь от `w+x` родительской директории, а не от `w` самого файла.

---

## 3. Как Linux выбирает permission class

```text
UID == owner?
    ↓ yes
owner permissions

иначе пользователь подходит под group class?
    ↓ yes
group permissions

иначе
others permissions
```

Права классов не складываются.

---

## 4. Ownership

```bash
sudo chown deploy file.txt
# Меняет owner файла.
```

```bash
sudo chown deploy:developers file.txt
# Меняет owner и group.
```

```bash
sudo chgrp developers file.txt
# Меняет только group.
```

---

## 5. umask

Типичные requested permissions:

```text
file      → 666
directory → 777
```

Примеры:

```text
umask 022 → files 644 / dirs 755
umask 002 → files 664 / dirs 775
umask 027 → files 640 / dirs 750
umask 077 → files 600 / dirs 700
```

`umask` убирает permission bits из прав, запрошенных программой.

---

## 6. Special bits

```text
SUID   = 4
SGID   = 2
Sticky = 1
```

### SUID

Executable запускается с effective UID владельца файла.

```text
4755
```

### SGID

Executable → effective GID группы файла.

Directory → новые объекты наследуют group ownership директории.

```bash
sudo chmod 2775 /srv/project
# Устанавливает SGID + rwxrwxr-x.
```

### Sticky bit

Ограничивает удаление/переименование чужих файлов в shared writable directory.

Классический пример:

```text
/tmp → drwxrwxrwt
```

---

## 7. ACL

Посмотреть:

```bash
getfacl file.txt
# Показывает ACL объекта.
```

Добавить пользователя:

```bash
setfacl -m u:bob:rw file.txt
# Даёт bob read/write через ACL.
```

Добавить группу:

```bash
setfacl -m g:developers:rw file.txt
# Даёт developers read/write через ACL.
```

---

## 8. ACL mask

Mask ограничивает:

```text
named users
owning group
named groups
```

Пример:

```text
user:bob:rw-
mask::r--

effective → r--
```

Исправление:

```bash
setfacl -m m::rw file.txt
# Устанавливает ACL mask read/write.
```

После `chmod` extended ACL нужно проверить снова.

---

## 9. Default ACL

Default ACL — шаблон ACL для новых объектов внутри директории.

```bash
sudo setfacl -m d:g:developers:rwx /srv/project
# Задаёт Default ACL для developers.
```

Для shared directory:

```text
SGID        → наследует group ownership
Default ACL → наследует permissions
```

Default ACL не даёт доступ к самой родительской директории.

---

# Users & Groups

## 10. Основные файлы

```text
/etc/passwd → информация об аккаунтах
/etc/shadow → password hashes + aging
/etc/group  → группы
/etc/skel   → шаблон нового home
```

---

## 11. UID / GID

```text
UID → User ID
GID → Group ID
```

Linux kernel работает с числовыми UID/GID.

```bash
id deploy
# Показывает UID, primary GID и supplementary groups.
```

```bash
getent passwd deploy
# Получает информацию о пользователе через NSS.
```

```bash
getent group developers
# Получает информацию о группе через NSS.
```

---

## 12. /etc/passwd

Формат:

```text
username:x:UID:GID:GECOS:HOME:SHELL
```

Пример:

```text
deploy:x:1006:1005::/home/deploy:/bin/bash
```

---

## 13. Primary / Supplementary Groups

У пользователя:

```text
1 primary group
0..N supplementary groups
```

Добавить supplementary group:

```bash
sudo usermod -aG monitoring deploy
# Добавляет monitoring, сохраняя остальные supplementary groups.
```

Осторожно:

```bash
sudo usermod -G monitoring deploy
# Заменяет список supplementary groups.
```

---

## 14. Создание пользователя

```bash
sudo useradd -m -s /bin/bash deploy
# Создаёт deploy, home directory и задаёт Bash как login shell.
```

Проверка:

```bash
getent passwd deploy
# Проверяет запись пользователя.
```

```bash
id deploy
# Проверяет UID/GID/groups.
```

---

## 15. Password / Account

```bash
sudo passwd -l deploy
# Блокирует password authentication.
```

```bash
sudo passwd -u deploy
# Разблокирует пароль.
```

```bash
sudo chage -l deploy
# Показывает password aging.
```

```bash
sudo chage -d 0 deploy
# Требует сменить пароль при следующем login.
```

Блокировка пароля не обязательно блокирует SSH key authentication.

---

## 16. System Account

```bash
sudo useradd -r -M -s /usr/sbin/nologin myapp
# Создаёт system account без физического home и без обычной login shell.
```

`nologin` запрещает обычный интерактивный login, но не запрещает запуск процессов с UID пользователя.

```bash
sudo -u myapp id
# Выполняет id от имени system account.
```

---

## 17. Group changes и процессы

Изменение membership не изменяет credentials уже запущенных процессов.

```text
/etc/group изменился
        ↓
старый shell → может иметь старые groups
новый login → получает актуальные groups
```

---

## 18. User Offboarding

До удаления:

```bash
sudo find / -user alex 2>/dev/null
# Ищет файлы пользователя alex.
```

Передать важные файлы:

```bash
sudo chown deploy:developers /var/www/project/file1.txt
# Передаёт ownership проектного файла.
```

Удалить:

```bash
sudo userdel -r alex
# Удаляет аккаунт, home и mail spool, где это применимо.
```

Проверить orphaned UID:

```bash
sudo find / -uid 1002 2>/dev/null
# Ищет оставшиеся файлы со старым UID.
```

Проверить orphaned GID:

```bash
sudo find / -gid 1003 2>/dev/null
# Ищет оставшиеся файлы со старым GID.
```

---

# Permission Denied — Troubleshooting

Не использовать `chmod 777` как способ диагностики.

Порядок:

```text
1. Кто выполняет операцию?
        ↓
2. UID / GID / groups
        ↓
3. Owner / group объекта
        ↓
4. Standard permissions
        ↓
5. x на всём пути
        ↓
6. ACL
        ↓
7. ACL mask
        ↓
8. Дополнительные ограничения
        ↓
9. Root cause
        ↓
10. Minimal fix
        ↓
11. Verify
```

Основные инструменты:

```bash
id username
# Проверяет identity и groups.
```

```bash
ls -ld /path
# Проверяет owner/group/permissions.
```

```bash
namei -l /full/path/file
# Проверяет permissions всего пути.
```

```bash
getfacl /path/file
# Проверяет ACL и mask.
```

```bash
stat /path/file
# Показывает подробные metadata объекта.
```

```bash
sudo -u username command
# Проверяет операцию от имени реального пользователя.
```

---

## Главное из Module 03

```text
Permissions ≠ только chmod.

Для нормальной диагностики нужно понимать:

UID/GID
    +
owner/group/others
    +
directory permissions
    +
umask
    +
SUID/SGID/sticky
    +
ACL/mask
    +
users/groups
    +
credentials процессов
```

Главный принцип:

> Diagnose → confirm root cause → minimal fix → verify.