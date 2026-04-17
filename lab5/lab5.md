---
title: "Отчёт по лабораторной работе № 5"
subtitle: "Дискреционное разграничение прав в Linux. Исследование влияния дополнительных атрибутов"
author: "Скворцова Анастасия, НБИбд-01-24"
date: "2026"
lang: ru-RU
geometry: margin=2cm
---

# Цель работы

Изучить механизмы изменения идентификаторов процессов, применение SetUID-, SetGID- и Sticky-битов, а также получить практические навыки работы в консоли Linux с дополнительными атрибутами файлов и каталогов.

# Подготовка лабораторного стенда

Перед выполнением работы были проверены установка компиляторов `gcc` и `g++`, а также состояние SELinux. Для корректного выполнения заданий SELinux должен находиться в режиме `Permissive`.

Команды:

```bash
gcc -v
getenforce
whereis gcc
whereis g++
```

![Подготовка лабораторного стенда](/mnt/data/lab5_assets/lab5_scr1.png)

# Листинги программ

## simpleid.c

```c
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>

int
main ()
{
    uid_t uid = geteuid ();
    gid_t gid = getegid ();
    printf ("uid=%d, gid=%d\n", uid, gid);
    return 0;
}
```

## simpleid2.c

```c
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>

int
main ()
{
    uid_t real_uid = getuid ();
    uid_t e_uid = geteuid ();
    gid_t real_gid = getgid ();
    gid_t e_gid = getegid ();

    printf ("e_uid=%d, e_gid=%d\n", e_uid, e_gid);
    printf ("real_uid=%d, real_gid=%d\n", real_uid, real_gid);
    return 0;
}
```

## readfile.c

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

int
main (int argc, char* argv[])
{
    unsigned char buffer[16];
    size_t bytes_read;
    int i;
    int fd = open (argv[1], O_RDONLY);

    do
    {
        bytes_read = read (fd, buffer, sizeof (buffer));
        for (i = 0; i < bytes_read; ++i)
            printf("%c", buffer[i]);
    }
    while (bytes_read == sizeof (buffer));

    close (fd);
    return 0;
}
```

# Ход выполнения работы

## 1. Компиляция и запуск программы `simpleid`

Создана и откомпилирована программа `simpleid.c`, выводящая эффективные идентификаторы пользователя и группы. Затем её вывод был сравнен с системной командой `id`.

```bash
gcc simpleid.c -o simpleid
./simpleid
id
```

![Компиляция и запуск simpleid](/mnt/data/lab5_assets/lab5_scr2.png)

**Наблюдение.** Для обычного запуска значения, выводимые программой, совпадают с текущими правами пользователя `guest`.

## 2. Создание `simpleid2` и установка SetUID

Создана программа `simpleid2.c`, которая выводит реальные и эффективные идентификаторы. После компиляции владельцем файла назначен `root`, затем установлен SetUID-бит.

```bash
gcc simpleid2.c -o simpleid2
./simpleid2
chown root:guest /home/guest/simpleid2
chmod u+s /home/guest/simpleid2
ls -l simpleid2
```

![Создание simpleid2 и установка SetUID](/mnt/data/lab5_assets/lab5_scr3.png)

**Наблюдение.** После установки SetUID в правах файла появилась буква `s` в позиции прав владельца.

## 3. Проверка влияния SetUID и SetGID

Программа была запущена повторно после изменения владельца и специальных битов. Затем установлен SetGID-бит.

```bash
./simpleid2
id
chmod g+s /home/guest/simpleid2
ls -l simpleid2
```

![Проверка SetUID и SetGID](/mnt/data/lab5_assets/lab5_scr4.png)

**Наблюдение.** При запуске файла с SetUID эффективный UID становится равным UID владельца файла, то есть `root`, а реальный UID остаётся равным `guest`.

## 4. Компиляция `readfile` и проверка доступа к защищённому файлу

Программа `readfile.c` предназначена для чтения содержимого файла, переданного в качестве аргумента. Файл `readfile.c` был защищён так, чтобы его мог читать только `root`, а на программу `readfile` установлен SetUID.

```bash
gcc readfile.c -o readfile
chown root:root /home/guest/readfile.c
chmod 600 /home/guest/readfile.c
cat /home/guest/readfile.c
chown root:guest /home/guest/readfile
chmod u+s /home/guest/readfile
./readfile /home/guest/readfile.c
```

![Программа readfile и защищённый файл](/mnt/data/lab5_assets/lab5_scr5.png)

**Наблюдение.** Пользователь `guest` не может читать `readfile.c` напрямую, однако программа `readfile` с SetUID может сделать это от имени владельца файла.

## 5. Проверка чтения `/etc/shadow`

```bash
./readfile /etc/shadow
```

![Чтение /etc/shadow](/mnt/data/lab5_assets/lab5_scr6.png)

**Наблюдение.** При корректной установке владельца `root` и SetUID-бита программа получает эффективные права суперпользователя и способна читать `/etc/shadow`. Это демонстрирует потенциальную опасность SetUID-программ.

## 6. Исследование Sticky-бита каталога `/tmp`

Сначала были просмотрены права каталога `/tmp`, затем от имени пользователя `guest` создан файл `file01.txt`, а для категории «остальные» выданы права на чтение и запись.

```bash
ls -l / | grep tmp
echo "test" > /tmp/file01.txt
ls -l /tmp/file01.txt
chmod o+rw /tmp/file01.txt
ls -l /tmp/file01.txt
```

![Исследование Sticky-бита](/mnt/data/lab5_assets/lab5_scr7.png)

**Наблюдение.** Каталог `/tmp` изначально имеет права `drwxrwxrwt`, где буква `t` указывает на наличие Sticky-бита.

## 7. Проверка чтения и записи в файл `/tmp` от имени другого пользователя

```bash
cat /tmp/file01.txt
echo "test2" >> /tmp/file01.txt
cat /tmp/file01.txt
echo "test3" > /tmp/file01.txt
cat /tmp/file01.txt
```

![Чтение и запись в /tmp](/mnt/data/lab5_assets/lab5_scr8.png)

**Наблюдение.** Пользователь `guest2` смог читать файл и изменять его содержимое благодаря правам на сам файл.

## 8. Проверка удаления файла при наличии и отсутствии Sticky-бита

```bash
rm /tmp/file01.txt
chmod -t /tmp
ls -l / | grep tmp
rm /tmp/file01.txt
```

![Удаление файла при наличии и отсутствии Sticky-бита](/mnt/data/lab5_assets/lab5_scr9.png)

**Наблюдение.** При установленном Sticky-бите удалить файл не удалось. После снятия бита `t` пользователь `guest2` смог удалить чужой файл.

# Пояснение используемых команд

- `chown root:guest file` меняет владельца файла на пользователя `root` и группу файла на `guest`.
- `chmod u+s file` устанавливает SetUID-бит, из-за чего исполняемый файл запускается с эффективным UID владельца файла.
- `chmod g+s file` устанавливает SetGID-бит, и программа получает эффективный GID группы файла.
- `chmod -t /tmp` снимает Sticky-бит с каталога. При отсутствии этого бита пользователи, имеющие права на каталог, могут удалять чужие файлы.
- `chmod +t /tmp` возвращает Sticky-бит и восстанавливает защиту от удаления файлов посторонними пользователями.

# Выводы

В ходе выполнения лабораторной работы были исследованы дополнительные атрибуты защиты в Linux: SetUID, SetGID и Sticky-бит. Установлено, что SetUID позволяет выполнять программу с эффективными правами владельца файла, а SetGID — с эффективной группой файла. На примере программы `readfile` было показано, что SetUID может использоваться для доступа к защищённым данным, что делает такие программы потенциально опасными при неправильной настройке. Также установлено, что Sticky-бит защищает файлы в общедоступных каталогах, например в `/tmp`, от удаления пользователями, не являющимися их владельцами. Таким образом, цель работы достигнута.
