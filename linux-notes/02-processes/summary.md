# Module 02 — Linux Processes: Summary

Краткая шпаргалка для повторения темы Linux Processes.

---

# 1. Process

**Процесс** — запущенный экземпляр программы.

```text
PID  → ID процесса
PPID → PID родительского процесса
```

Упрощённая схема запуска программы из Bash:

```text
Bash
  ↓
fork()
  ↓
Child
  ↓
execve()
  ↓
Program
```

- `fork()` — создаёт дочерний процесс.
- `execve()` — заменяет выполняемую программу процесса новой.

---

# 2. Process States

| State | Значение |
|---|---|
| `R` | Running / Runnable |
| `S` | Interruptible Sleep |
| `D` | Uninterruptible Sleep |
| `T` | Stopped |
| `Z` | Zombie |

### R

Процесс выполняется **или готов к выполнению**.

`R` не означает, что процесс обязательно прямо сейчас физически выполняется на CPU.

### S

Процесс находится в прерываемом ожидании:

```text
таймер / данные / событие / ввод
```

### D

Непрерываемое ожидание операции ядра, часто I/O.

Пока процесс остаётся в `D`:

```text
SIGKILL не может завершить его немедленно
```

Длительный `D` → искать проблемы с I/O, storage, filesystem, NFS и т.д.

### T

Процесс остановлен.

```text
Ctrl+Z → SIGTSTP → T
```

### Z

Процесс уже завершён, но Parent ещё не забрал его exit status.

---

# 3. Zombie

```text
Child завершается
      ↓
Kernel сохраняет exit status
      ↓
Zombie
      ↓
PARENT вызывает wait()/waitpid()
      ↓
Parent получает exit status
      ↓
Zombie удаляется
```

Главное:

> **Parent вызывает `wait()` / `waitpid()`.**

Zombie уже завершён, поэтому:

```bash
# Не поможет против Zombie — завершать уже нечего.
kill -9 ZOMBIE_PID
```

Много Zombie с одинаковым `PPID` → расследовать Parent.

---

# 4. Zombie vs Orphan

```text
ZOMBIE
→ процесс уже завершён
→ Parent не забрал exit status
```

```text
ORPHAN
→ процесс жив
→ первоначальный Parent завершился
→ происходит reparenting
→ PPID изменяется
```

Важно:

```text
Parent завершился
≠
автоматический SIGHUP всем Child
```

---

# 5. Foreground / Background

Foreground:

```bash
# Bash ждёт завершения или остановки sleep.
sleep 300
```

Background:

```bash
# & запускает команду в background
# и сразу возвращает управление Bash.
sleep 300 &
```

Пример:

```text
[1] 5000
```

```text
[1]  → Job ID
5000 → PID
```

---

# 6. Job Control

```bash
# Показывает jobs текущего Bash.
jobs
```

```text
Ctrl+Z
→ SIGTSTP
→ остановить foreground job
```

```bash
# Продолжает job №1 в background.
bg %1
```

```bash
# Переводит job №1 в foreground.
fg %1
```

При:

```text
Ctrl+Z → bg → fg
```

PID процесса не меняется.

Важно:

```text
jobs ≠ все процессы Linux
```

---

# 7. SIGTSTP vs SIGSTOP

```text
SIGTSTP
→ обычно Ctrl+Z
→ можно обработать/игнорировать
```

```text
SIGSTOP
→ принудительная остановка
→ нельзя обработать/игнорировать/заблокировать
```

```bash
# Принудительно останавливает процесс.
kill -SIGSTOP PID

# Продолжает выполнение остановленного процесса.
kill -SIGCONT PID
```

---

# 8. Основные сигналы

| Signal | Назначение |
|---|---|
| `SIGHUP` | Hangup |
| `SIGINT` | Interrupt |
| `SIGKILL` | принудительное завершение |
| `SIGTERM` | запрос на завершение |
| `SIGCONT` | продолжить выполнение |
| `SIGSTOP` | принудительно остановить |
| `SIGTSTP` | terminal stop |

```bash
# Показывает список сигналов.
kill -l
```

---

# 9. SIGTERM vs SIGKILL

```bash
# Отправляет SIGTERM по умолчанию.
kill PID
```

```text
SIGTERM
→ запрос на завершение
→ можно обработать/игнорировать
→ позволяет выполнить graceful shutdown
```

```bash
# Отправляет SIGKILL.
kill -9 PID
```

```text
SIGKILL
→ принудительное завершение
→ нельзя обработать
→ нельзя проигнорировать
→ нельзя заблокировать
→ application-level cleanup невозможен
```

Правило:

```text
SIGTERM
   ↓
проверить результат
   ↓
SIGKILL только при необходимости
```

---

# 10. kill / pkill / killall

```bash
# Сигнал конкретному PID.
kill 5000
```

```bash
# Поиск процессов sleep по критериям
# и отправка им SIGTERM.
pkill sleep
```

```bash
# Отправляет SIGTERM процессам
# с именем sleep.
killall sleep
```

Важно:

> `kill` не обязательно убивает процесс. `kill` отправляет сигнал.

Например:

```bash
# Останавливает, но не завершает процесс.
kill -SIGSTOP 5000
```

---

# 11. ps / pgrep / jobs

```bash
# Показывает все процессы в расширенном формате.
ps -ef
```

```bash
# Показывает конкретный процесс.
ps -p 5000 -o pid,ppid,stat,cmd
```

```bash
# Ищет PID процессов nginx.
pgrep nginx
```

```bash
# Показывает jobs текущего Bash.
jobs
```

Удобная модель:

```text
ps
→ посмотреть процессы

pgrep
→ найти процессы

jobs
→ посмотреть jobs текущего Bash
```

---

# 12. Scheduler

Scheduler распределяет CPU между runnable-задачами.

```text
1 CPU
+
8 CPU-bound процессов
```

не означает, что восемь процессов физически выполняются одновременно.

Scheduler распределяет между ними CPU time.

При наличии нескольких CPU несколько задач могут выполняться физически параллельно.

---

# 13. nice

Диапазон:

```text
-20 ........ 0 ........ 19
 ↑                       ↑
выше                    ниже
CPU priority            CPU priority
```

```text
nice=-20 → высокий CPU priority
nice=0   → стандартный
nice=19  → низкий CPU priority
```

Запустить новый процесс:

```bash
# Запускает command с nice=10.
nice -n 10 command
```

Изменить существующий:

```bash
# Изменяет nice существующего процесса.
renice 15 -p PID
```

### Главное

```text
nice ≠ CPU limit
```

CPU-bound процесс с:

```text
nice=19
```

без конкурентов всё равно может использовать практически:

```text
100% одного CPU
```

`nice` имеет значение прежде всего при **конкуренции за CPU**.

---

# 14. nohup / & / disown

Не путать три разных механизма:

```text
&
→ запустить команду в background
```

```text
nohup
→ запустить программу с игнорированием SIGHUP
```

```text
disown
→ убрать существующий job
  из Job Control текущего Bash
```

Пример:

```bash
# nohup обеспечивает игнорирование SIGHUP,
# & запускает программу в background.
nohup ./backup.sh &
```

```bash
# Убирает job №1 из Job Control Bash.
disown %1
```

`nohup` и reparenting — разные механизмы.

---

# 15. Background и terminal input

Background job не должен читать из управляющего терминала.

Например:

```bash
# cat попытается читать stdin из терминала,
# находясь при этом в background.
cat &
```

Возможный результат:

```text
SIGTTIN
   ↓
Stopped
```

Если stdin приходит из файла:

```bash
# stdin берётся из файла, а не терминала.
cat < /etc/hosts &
```

процесс может нормально выполниться в background.

---

# 16. Базовый troubleshooting процесса

Если процесс не завершился после `SIGTERM`:

```text
НЕ НАЧИНАТЬ сразу с kill -9
```

Сначала:

```bash
# Проверить состояние, родителя, CPU и команду.
ps -p PID -o pid,ppid,stat,pcpu,cmd
```

```bash
# Проверить состояние systemd-сервиса.
systemctl status SERVICE
```

```bash
# Проверить журнал systemd-сервиса.
journalctl -u SERVICE
```

```bash
# Проверить последние сообщения ядра.
dmesg | tail -n 100
```

На CentOS 7:

```bash
# Проверить последние системные сообщения.
tail -n 100 /var/log/messages
```

Логика:

```text
State
  ↓
Service
  ↓
Logs
  ↓
Kernel / I/O
  ↓
Оценить последствия
  ↓
SIGKILL при необходимости
```

---

# 17. Главные ловушки на собеседовании

### ❌ Неправильно

```text
Zombie ждёт wait() от Parent.
```

### ✅ Правильно

```text
Parent вызывает wait()/waitpid()
и забирает exit status Child.
```

---

### ❌ Неправильно

```text
Ctrl+Z → SIGSTOP
```

### ✅ Правильно

```text
Ctrl+Z → SIGTSTP
```

---

### ❌ Неправильно

```text
Parent умер → SIGHUP всем детям.
```

### ✅ Правильно

```text
Смерть Parent и SIGHUP —
разные механизмы.
```

---

### ❌ Неправильно

```text
nice=19 → максимум 2% CPU.
```

### ✅ Правильно

```text
nice определяет относительный CPU priority.

Без конкурентов nice=19
может использовать ~100% одного CPU.
```

---

### ❌ Неправильно

```text
kill всегда убивает процесс.
```

### ✅ Правильно

```text
kill отправляет сигнал.
```

---

# Быстрое повторение

```text
PID  → Process ID
PPID → Parent Process ID

R → Running / Runnable
S → Interruptible Sleep
D → Uninterruptible Sleep
T → Stopped
Z → Zombie

Zombie → мёртв
Orphan → жив

Parent → wait()/waitpid()

Ctrl+Z → SIGTSTP
SIGSTOP → принудительный Stop
SIGCONT → Continue

SIGTERM → graceful termination request
SIGKILL → forced termination

jobs → jobs текущего Bash
ps → процессы
pgrep → поиск процессов

& → background
nohup → SIGHUP
disown → Bash Job Control

nice ↓ → CPU priority ↑
nice ↑ → CPU priority ↓
nice ≠ CPU limit
```