# MEPHI Linux Security 2026 — ДЗ №1

**Студент:** Хабарова Мария Антоновна  
**ОС:** РЕД ОС (NetMasha RED OS)  
**Дата:** 23.09.2026

---

## Раздел 1. Создание пользователя

Создан пользователь `user1` с UID `1234`, входящий в группу `students`, со сменой пароля каждые 90 дней (3 месяца).

```bash
sudo groupadd students
sudo useradd -u 1234 -g students -m user1
sudo passwd user1
sudo chage -M 90 user1
```

Проверка:
```bash
sudo chage -l user1
```
<img width="2880" height="1704" alt="Снимок экрана 2026-09-22 224519" src="https://github.com/user-attachments/assets/de6a26a6-7150-4396-aafa-919adaad3661" />

---

## Раздел 2. Мониторинг файлов и процессов

### 2.1. Поиск файлов с битом set-UID

```bash
sudo find / -perm /4000 -type f 2>/dev/null > ~/suid_files.out
```

Результат сохранён в `suid_files.out`.

### 2.2. Поиск процессов с EUID=0 и RUID≠0

В первом терминале запущена смена пароля:
```bash
su - user1
passwd
```
Во втором терминале выполнен поиск:
```bash
ps -eo pid,ruid,euid,comm | awk '$3 == 0 && $2 != 0' > ~/proc.out
cat ~/proc.out
```

В выводе виден процесс `passwd` с реальным UID пользователя `user1` и эффективным UID `0`.

Результат сохранён в `proc.out`.
<img width="2880" height="1704" alt="Снимок экрана 2026-09-23 201448" src="https://github.com/user-attachments/assets/27dccd97-4d96-431e-96a2-c9b3c01b5cbc" />

---

## Раздел 3. Изучение механизма set-UID

**Выбранная утилита:** `tee`  
**Привилегированная операция:** запись в `/etc/fstab` (обычный пользователь туда писать не может).

Копия была помещена в `/home/user1/`

```bash
# Копирование утилиты
sudo cp /usr/bin/tee /home/user1/netmashasuidtee

# Владелец — root (обязательно для работы SUID)
sudo chown root:root /home/user1/netmashasuidtee

# Установка SUID-бита
sudo chmod u+s /home/user1/netmashasuidtee

# Проверка прав
ls -l /home/user1/netmashasuidtee
```

Ожидаемый результат: `-rwsr-xr-x 1 root root ...`

Проверка от имени обычного пользователя:
```bash
su - user1

# Без SUID — отказ
echo "# test" >> /etc/fstab
# -bash: /etc/fstab: Permission denied

# Через SUID-копию — успешно
echo "# test line from user1" | /home/user1/netmashasuidtee -a /etc/fstab

# Проверка
tail -n 3 /etc/fstab
```
<img width="2880" height="1704" alt="Снимок экрана 2026-09-22 230909" src="https://github.com/user-attachments/assets/994e42ff-994b-4b51-93fa-1c09d4a4c641" />

---

## Раздел 4. Изучение механизма привилегий (capabilities)

**Выбранная утилита:** `chown`  
**Привилегированная операция:** смена владельца файла на произвольного пользователя.

Вместо полного SUID-бита утилите выдана только одна capability — `cap_chown`.

```bash
# Копирование утилиты
sudo cp /usr/bin/chown /home/user1/mycapchown

# Выдача capability
sudo setcap cap_chown+ep /home/user1/mycapchown

Проверка от имени обычного пользователя:
```bash
su - user1
cd /tmp
touch test.txt
/home/test1/mycapchown root testfile
ls -l test.txt
```

Владелец файла меняется на `root`, хотя команда запущена от `user1`.
<img width="2880" height="1704" alt="Снимок экрана 2026-09-22 231703" src="https://github.com/user-attachments/assets/38b3bbea-1e50-44c2-b9fe-14a1807ee033" />

---

## Раздел 5. Изучение механизма sudo

Пользователю `user1` разрешено менять системное время без ввода пароля.

```bash
sudo visudo
```

В конец файла `/etc/sudoers` добавлена строка:
```
user1 ALL=(ALL) NOPASSWD: /usr/bin/date
```

Проверка:
```bash
su - user1
sudo date -s "2026-01-01 12:00:00"
```

Пароль не запрашивается, время меняется.
<img width="2880" height="1704" alt="Снимок экрана 2026-09-22 232115" src="https://github.com/user-attachments/assets/449f540f-a708-4f50-9687-0edf1e193dad" />

---

## Раздел 6. Сбор артефактов

```bash
# История команд
history > ~/history.out

# Информация о домашней директории пользователя
stat /home/user1 > ~/stat.out

# Информация об утилитах из разделов 3 и 4
stat /usr/local/bin/netmashasuidtee >> ~/stat.out
stat /usr/local/bin/netmashacaphown >> ~/stat.out

# Информация о capabilities
getcap /usr/local/bin/netmashacaphown > ~/getcap.out

# Системные файлы
sudo cp /etc/passwd  ~/passwd.out
sudo cp /etc/shadow  ~/shadow.out
sudo cp /etc/group   ~/group.out
sudo cp /etc/sudoers ~/sudoers.out

Скриншот с уникальным номером сохранён как `mephi-screenshot.png`.
