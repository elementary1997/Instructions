# Инструкция: Монтирование удалённого каталога через SSHFS от root

Данная инструкция позволяет смонтировать каталог удалённого Linux-сервера локально с помощью SSHFS через пользователя root с автомонтированием через `/etc/fstab`.

---

## Оглавление

1. [Установка SSHFS](#1-установка-sshfs)
2. [Генерация SSH-ключа для root](#2-генерация-ssh-ключа-для-root)
3. [Копирование публичного ключа на удалённый сервер](#3-копирование-публичного-ключа-на-удалённый-сервер)
4. [Предварительное подключение для known_hosts](#4-предварительное-подключение-для-known_hosts)
5. [Настройка /etc/fuse.conf](#5-настройка-etcfuseconf)
6. [Создание локальной точки монтирования](#6-создание-локальной-точки-монтирования)
7. [Добавление записи в /etc/fstab](#7-добавление-записи-в-etcfstab)
8. [Проверка автомонтирования](#8-проверка-автомонтирования)

---

## 1. Установка SSHFS

Установите пакет SSHFS на локальную машину:

```bash
sudo apt update
sudo apt install sshfs -y
```

Для других дистрибутивов:

```bash
# CentOS/RHEL/Fedora
sudo yum install sshfs

# Arch Linux
sudo pacman -S sshfs
```

---

## 2. Генерация SSH-ключа для root

Перейдите в оболочку root и создайте SSH-ключ:

```bash
sudo -i
ssh-keygen -t rsa -b 4096
```

При запросе:
- **Путь к файлу**: Оставьте по умолчанию (`/root/.ssh/id_rsa`)
- **Пароль (passphrase)**: Нажмите Enter (оставьте пустым для автоматизации)

Ключ будет создан в `/root/.ssh/id_rsa` (приватный) и `/root/.ssh/id_rsa.pub` (публичный).

---

## 3. Копирование публичного ключа на удалённый сервер

Скопируйте публичный ключ root на удалённый сервер:

```bash
ssh-copy-id username@remote_server
```

Замените:
- `username` — имя пользователя на удалённом сервере
- `remote_server` — IP-адрес или доменное имя удалённого сервера

Пример:
```bash
ssh-copy-id user@192.168.1.19
```

Введите пароль удалённого пользователя для завершения копирования.

---

## 4. Предварительное подключение для known_hosts

Выполните тестовое монтирование вручную для добавления отпечатка хоста в `/root/.ssh/known_hosts`:

```bash
mkdir -p /mnt/test
sshfs username@remote_server:/remote/path /mnt/test -o allow_other,IdentityFile=/root/.ssh/id_rsa
```

Проверьте доступность файлов:

```bash
ls -la /mnt/test
```

Размонтируйте каталог:

```bash
umount /mnt/test
```

---

## 5. Настройка /etc/fuse.conf

Откройте конфигурационный файл FUSE:

```bash
sudo nano /etc/fuse.conf
```

Найдите и раскомментируйте (уберите `#`) строку:

```
user_allow_other
```

Если строка отсутствует, добавьте её в файл. Сохраните изменения (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 6. Создание локальной точки монтирования

Создайте каталог для монтирования:

```bash
mkdir -p /home/user/obsidian-remote
```

Установите правильного владельца (замените `user` на ваше имя пользователя):

```bash
chown -R user:user /home/user/obsidian-remote
```

Узнайте свой UID и GID:

```bash
id -u user  # Выведет UID (например, 1000)
id -g user  # Выведет GID (например, 1000)
```

Запомните эти значения для следующего шага.

---

## 7. Добавление записи в /etc/fstab

Откройте файл `/etc/fstab` для редактирования:

```bash
sudo nano /etc/fstab
```

Добавьте строку в конец файла:

```
username@remote_server:/remote/path /local/mount/point fuse.sshfs _netdev,IdentityFile=/root/.ssh/id_rsa,allow_other,default_permissions,uid=1000,gid=1000,reconnect 0 0
```


### Описание параметров:

| Параметр | Описание |
|----------|----------|
| `_netdev` | Монтирование после инициализации сети |
| `IdentityFile=/root/.ssh/id_rsa` | Путь к приватному SSH-ключу root |
| `allow_other` | Разрешает доступ другим пользователям |
| `default_permissions` | Проверяет права доступа по правам удалённой ФС |
| `uid=1000` | UID владельца файлов (замените на свой) |
| `gid=1000` | GID группы файлов (замените на свой) |
| `reconnect` | Автоматическое переподключение при разрыве |

Сохраните файл (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 8. Проверка автомонтирования

Примените изменения и смонтируйте все записи из fstab:

```bash
sudo mount -a
```

Проверьте успешность монтирования:

```bash
df -h | grep obsidian-remote
mount | grep sshfs
ls -la /home/user/obsidian-remote
```

Если монтирование прошло успешно, вы увидите содержимое удалённого каталога.

### Перезагрузка для проверки автомонтирования:

```bash
sudo reboot
```

После перезагрузки проверьте, что каталог смонтирован автоматически:

```bash
df -h | grep obsidian-remote
```

---

## Устранение неполадок

### Проблема: "Connection refused"

**Решение**: Проверьте, что SSH-сервис запущен на удалённом сервере:

```bash
ssh username@remote_server
systemctl status sshd
```

### Проблема: "Permission denied (publickey)"

**Решение**: Убедитесь, что публичный ключ скопирован корректно:

```bash
ssh username@remote_server "cat ~/.ssh/authorized_keys"
```

### Проблема: Каталог не монтируется при загрузке

**Решение**: Добавьте опции `noauto,x-systemd.automount` в fstab для монтирования по требованию:

```
user@192.168.1.19:/mnt/obsidian /home/user/obsidian-remote fuse.sshfs noauto,x-systemd.automount,_netdev,IdentityFile=/root/.ssh/id_rsa,allow_other,default_permissions,uid=1000,gid=1000,reconnect 0 0
```

### Проблема: "Transport endpoint is not connected"

**Решение**: Размонтируйте и смонтируйте повторно:

```bash
sudo umount /home/user/obsidian-remote
sudo mount /home/user/obsidian-remote
```

---

## Дополнительные опции SSHFS

Вы можете добавить следующие опции в запись fstab через запятую:

- `compression=yes` — Включить сжатие данных при передаче
- `ServerAliveInterval=15` — Поддерживать соединение активным (отправка keepalive каждые 15 сек)
- `ServerAliveCountMax=3` — Количество попыток keepalive перед разрывом
- `port=2222` — Использовать нестандартный SSH-порт
- `StrictHostKeyChecking=no` — Отключить проверку отпечатка хоста (не рекомендуется)

Пример с дополнительными опциями:

```
user@192.168.1.19:/mnt/obsidian /home/user/obsidian-remote fuse.sshfs _netdev,IdentityFile=/root/.ssh/id_rsa,allow_other,default_permissions,uid=1000,gid=1000,reconnect,compression=yes,ServerAliveInterval=15,ServerAliveCountMax=3 0 0
```

---

## Размонтирование

Для ручного размонтирования каталога:

```bash
sudo umount /home/user/obsidian-remote
```

Для принудительного размонтирования (при зависании):

```bash
sudo umount -l /home/user/obsidian-remote
```

---

## Примечания

- Все операции монтирования через fstab выполняются от имени root, но файлы внутри каталога доступны пользователю с указанными `uid` и `gid`.
- Для работы с Obsidian просто откройте смонтированный каталог как vault через "Open folder as vault".
- SSHFS использует шифрование SSH, поэтому все данные передаются безопасно.
- При работе через медленное соединение рекомендуется включить опцию `compression=yes`.

---
