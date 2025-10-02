# Домашнее задание 1 (Linux Архитектура и файловые системы)

Выполнил: Целиков Данил

## Содержание

1. [Задание 1. Kernel and Module Inspection](#задание-1-kernel-and-module-inspection)

2. [Задание 2. Наблюдение за VFS](#задание-2-наблюдение-за-vfs)

3. [Задание 3. LVM Management](#задание-3-lvm-management)

4. [Задание 4. Использование pseudo filesystem](#задание-4-использование-pseudo-filesystem)

## Задание 1. Kernel and Module Inspection

### Продемонстрировать версию ядра вашей ОС.

```shell
uname -a
```

Для более краткого вывода:

```shell
uname -r
```

![1_1.png](images/1_1.png)

### Показать все загруженные модули ядра.

```shell
lsmod
```

![1_2.png](images/1_2.png)

### Отключить автозагрузку модуля cdrom.

так как модуль cdrom builtin(встроен в ядро), я не понял как его отключить и в lsmod он не показывается. Но список
команд для его отключения представлен ниже

```shell
echo "blacklist cdrom" | sudo tee /etc/modprobe.d/blacklist-cdrom.conf
sudo update-initramfs -u
sudo reboot
```

![1_3.png](images/1_3.png)
![1_3_2.png](images/1_3_2.png)

### Найти и описать конфигурацию ядра (файл конфигурации, параметр CONFIG_XFS_FS).

```shell
grep CONFIG_XFS_FS /boot/config-$(uname -r)
```

![1_4.png](images/1_4.png)

CONFIG_XFS_FS - параметр ядра, который включает поддержку файловой системы XFS

Если значение равно y или m:

* y - поддержка вкомпилирована в ядро
* m - поддержка реализована как загружаемый модуль
* n - поддержка отключена

## Задание 2. Наблюдение за VFS

### Используйте strace для анализа команды cat /etc/os-release > /dev/null.

![2_1.png](images/2_1.png)

### Описать открываемый и читаемый файл, объяснить отсутствие записывающих вызовов в выводе.

`openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3`

* Система загружает кэш динамических библиотек для поиска зависимостей программы `cat`

`close(3) = 0`

* Закрывается файл кэша библиотек после использования

`openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3`

* Загружается основная библиотека libc, необходимая для работы программы

`read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\237\2\0\0\0\0\0"..., 832) = 832`

* Читается заголовок исполняемого файла библиотеки (символы `\177ELF` - магическое число ELF)

`close(3) = 0`

* Закрывается файл библиотеки после загрузки

`openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3`

* Загружаются настройки локализации и языковые стандарты

`close(3) = 0`

* Закрывается файл локалей после использования

`openat(AT_FDCWD, "/etc/os-release", O_RDONLY) = 3`

* Открывается файл `/etc/os-release` только для чтения - это основной файл задания

`read(3, "PRETTY_NAME=\"Ubuntu 22.04 LTS\"\nN"..., 131072) = 378`

* Читается 378 байт данных из файла os-release в буфер размером 131072 байт

`write(1, "PRETTY_NAME=\"Ubuntu 22.04 LTS\"\nN"..., 378) = 378`

* Данные записываются в stdout (дескриптор 1), но из-за `> /dev/null` они отбрасываются

`read(3, "", 131072) = 0`

* Чтение возвращает 0 байт - достигнут конец файла

`close(3) = 0`

* Закрывается файл `/etc/os-release`

`close(1) = 0`

* Закрывается стандартный вывод (дескриптор 1)

`close(2) = 0`

* Закрывается стандартный поток ошибок (дескриптор 2)

`+++ exited with 0 +++`

* Программа успешно завершилась с кодом возврата 0

Почему нет вызовов write? Потому что `> /dev/null перенаправляет стандартный вывод в /dev/null`

## Задание 3. LVM Management

### Добавить к своей виртуальной машине  диск /dev/sdb размером 2GB.

![3_1.png](images/3_1.png)

```shell
lsblk
```

![3_1_2.png](images/3_1_2.png)

### Создать раздел на /dev/sdb, используя fdisk или parted.

```shell
sudo fdisk /dev/sdb
```

![3_2.png](images/3_2.png)

где:

* n - новый раздел
* p - primary
* 1 - номер раздела
* 2048 - первый сектор по умолчанию
* 4194303 - последний сектор по умолчанию (весь диск)
* w - записать изменения

Проверяем

```shell
sudo fdisk /dev/sdb
```

![3_2_1.png](images/3_2_1.png)

Раздел `sdb1` появился

### Создать Physical Volume (PV) на этом разделе.

```shell
sudo pvcreate /dev/sdb1
```

![3_3.png](images/3_3.png)

Проверяем

```shell
sudo pvdisplay
```

![3_3_2.png](images/3_3_2.png)

### Создать Volume Group (VG) с именем vg_highload.

```shell
sudo vgcreate vg_highload /dev/sdb1
```

![3_4.png](images/3_4.png)

Проверяем

```shell
sudo vgdisplay vg_highload
```

![3_4_2.png](images/3_4_2.png)

### Создать два Logical Volume (LV): data_lv (1200 MiB) и logs_lv (оставшееся место).

```shell
sudo lvcreate -L 1200M -n data_lv vg_highload
sudo lvcreate -l 100%FREE -n logs_lv vg_highload
```

![3_5.png](images/3_5.png)

Проверяем

```shell
sudo lvdisplay vg_highload
```

![3_5_2.png](images/3_5_2.png)

### Отформатировать data_lv как ext4 и примонтировать в /mnt/app_data.

```shell
sudo mkfs.ext4 /dev/vg_highload/data_lv
sudo mkdir -p /mnt/app_data
sudo mount /dev/vg_highload/data_lv /mnt/app_data
```

![3_6.png](images/3_6.png)

Проверяем

```shell
df -h /mnt/app_data
```

![3_6_2.png](images/3_6_2.png)

### Отформатировать logs_lv как xfs и примонтировать в /mnt/app_logs.

```shell
sudo mkfs.xfs /dev/vg_highload/logs_lv
sudo mkdir -p /mnt/app_logs
sudo mount /dev/vg_highload/logs_lv /mnt/app_logs
```

![3_7.png](images/3_7.png)

Проверяем

```shell
df -h /mnt/app_logs
```

![3_7_2.png](images/3_7_2.png)

## Задание 4. Использование pseudo filesystem

### Извлечь из /proc модель CPU и объём памяти (KiB).

```shell
cat /proc/cpuinfo | grep "model name"
cat /proc/meminfo | grep MemTotal
```

![4_1.png](images/4_1.png)

### Используя /proc/\$$/status, найдите Parent Process ID (PPid) вашего текущего shell. что означает $$ ?

```shell
cat /proc/$$/status | grep PPid
```

![4_2.png](images/4_2.png)

Что означает \$$? $$ - это специальная переменная в bash, которая содержит PID (Process ID) текущего shell-процесса

### Определить настройки I/O scheduler для основного диска /dev/sda.

```shell
cat /sys/block/sda/queue/scheduler
```

![4_3.png](images/4_3.png)

Текущий активный scheduler показывается в квадратных скобках \[mq-deadline]. none означает, что управление передано
контроллеру диска

### Определить размер MTU для основного сетевого интерфейса (например, eth0 или ens33).

Сначала определим основной интерфейс

```shell
ip route show default | awk '{print $5}' | head -1
```

![4_4.png](images/4_4.png)

```shell
ip link | grep mtu
```

![4_4_2.png](images/4_4_2.png)

MTU 1500