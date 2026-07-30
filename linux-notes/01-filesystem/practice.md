# Practice — Linux Filesystem

## Experiment 1. Абсолютные и относительные пути

Создадим учебную структуру:

```bash
# Создаёт вложенную структуру каталогов.
mkdir -p ~/filesystem-lab/projects/demo
```

```bash
# Переходит в каталог filesystem-lab.
cd ~/filesystem-lab
```

Абсолютный путь:

```bash
# Переходит в каталог, используя полный путь от корня файловой системы.
cd ~/filesystem-lab/projects/demo
```

Относительный путь:

```bash
# Переходит на два уровня выше относительно текущего каталога.
cd ../..
```

Проверка:

```bash
# Показывает текущий рабочий каталог.
pwd
```

## Experiment 2. Просмотр inode

```bash
# Создаёт файл с текстом Hello Linux.
echo "Hello Linux" > original.txt
```

```bash
# Показывает номер inode и подробную информацию о файле.
ls -li original.txt
```

## Experiment 3. Hard link

```bash
# Создаёт дополнительное имя hardlink.txt для inode файла original.txt.
ln original.txt hardlink.txt
```

```bash
# Показывает, что оба имени имеют одинаковый inode.
ls -li original.txt hardlink.txt
```

Изменим содержимое через второе имя:

```bash
# Добавляет строку через hardlink.txt.
# Поскольку оба имени указывают на один inode, original.txt тоже изменится.
echo "Second line" >> hardlink.txt
```

```bash
# Показывает содержимое original.txt.
cat original.txt
```

Удаляем исходное имя:

```bash
# Удаляет только запись original.txt из каталога.
# Данные сохраняются, потому что остаётся hardlink.txt.
rm original.txt
```

```bash
# Проверяет, что содержимое остаётся доступным через hardlink.txt.
cat hardlink.txt
```

## Experiment 4. Symbolic link

```bash
# Создаёт файл target.txt.
echo "Symbolic link target" > target.txt
```

```bash
# Создаёт символическую ссылку symlink.txt, содержащую путь к target.txt.
ln -s target.txt symlink.txt
```

```bash
# Показывает подробную информацию и направление символической ссылки.
ls -li target.txt symlink.txt
```

Удаляем целевой файл:

```bash
# Удаляет целевой файл символической ссылки.
rm target.txt
```

```bash
# Показывает, что symlink.txt осталась, но её цель отсутствует.
ls -l symlink.txt
```

Попытка прочитать ссылку:

```bash
# Пытается открыть цель символической ссылки.
# Команда завершится ошибкой, потому что цель была удалена.
cat symlink.txt
```

## Experiment 5. cp создаёт новый inode

```bash
# Создаёт исходный файл.
echo "Copy test" > source.txt
```

```bash
# Создаёт отдельную копию файла.
cp source.txt copy.txt
```

```bash
# Показывает, что файлы имеют разные inode.
ls -li source.txt copy.txt
```

Изменение копии:

```bash
# Добавляет текст только в copy.txt.
echo "Changed copy" >> copy.txt
```

```bash
# Показывает, что source.txt не изменился.
cat source.txt
```

## Experiment 6. mv в пределах одной файловой системы

```bash
# Создаёт каталог назначения.
mkdir -p moved
```

```bash
# Запоминаем inode файла до перемещения.
ls -li source.txt
```

```bash
# Перемещает файл в другой каталог.
mv source.txt moved/source.txt
```

```bash
# Проверяем inode после перемещения.
# В пределах одной файловой системы inode обычно останется прежним.
ls -li moved/source.txt
```

## Experiment 7. Удалённый, но открытый файл

В первом терминале:

```bash
# Создаёт файл размером 100 МБ, заполненный нулевыми байтами.
# dd используется здесь для создания тестового большого файла.
dd if=/dev/zero of=large.log bs=1M count=100
```

```bash
# Открывает файл и продолжает ожидать новые строки.
# Процесс tail удерживает файловый дескриптор открытым.
tail -f large.log
```

Во втором терминале:

```bash
# Удаляет имя файла из каталога.
# Процесс tail при этом продолжает удерживать открытый файл.
rm large.log
```

```bash
# Ищет открытые удалённые файлы.
sudo lsof +L1
```

После завершения `tail` через `Ctrl+C` файловый дескриптор будет закрыт, и файловая система сможет освободить занятое место.

## Cleanup

После экспериментов:

```bash
# Возвращается в домашний каталог.
cd ~
```

```bash
# Рекурсивно удаляет учебный каталог со всеми созданными файлами.
# Использовать осторожно: rm -rf удаляет данные без подтверждения.
rm -rf ~/filesystem-lab
```