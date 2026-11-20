# Write-up: Valley (FTP → PCAP → SSH → Python Library Hijacking)

## Enumeration

Сканируем цель:

```bash
sudo nmap -sS -sV -sC -p- --min-rate=1000 10.146.185.63
```

```text
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
37370/tcp open  ftp     vsftpd 3.0.3
```

FTP анонимно недоступен.

## HTTP

Открываем сайт:

![Главная](img/Pasted_image_20261119183040.png)

По ссылке `/pricing` редиректит на `/pricing/pricing.html`:

![pricing.html](img/Pasted_image_20261119183133.png)

На странице `/pricing` находим `note.txt`:

![note.txt](img/Pasted_image_20261119183239.png)

## Обход через null byte

Пробуем эндпоинт (`00`) в `/static/`:

![Null byte](img/Pasted_image_20261119185936.png)

Читаем `dev notes from valleyDev`:

```text
-add wedding photo examples
-redo the editing on #4
-remove /dev1243224123123
-check for SIEM alerts
```

Находим скрытую директорию `/dev1243224123123`:

![dev1243224123123](img/Pasted_image_20261119190301.png)

В `dev.js` находим креды:

```javascript
if (username === "siemDev" && password === "california")
```

![dev.js](img/Pasted_image_20261119190925.png)

## FTP

В `devNotes37370.txt` — заметки для FTP-сервера:

```text
dev notes for ftp server:
-stop reusing credentials
-check for any vulnerabilies
-stay up to date on patching
-change ftp port to normal port
```

Подключаемся на нестандартный порт:

```bash
ftp 10.146.185.63 -P 37370
# siemDev:california
```

![FTP](img/Pasted_image_20261119191252.png)

Скачиваем все `pcapng` файлы и открываем в Wireshark.

## Анализ PCAP

В `siemHTTP2.pcapng` находим креды:

![siemHTTP2.pcapng](img/Pasted_image_20261119192207.png)

```text
valleyDev:ph0t0s1234
```

Заходим по SSH:

```bash
ssh valleyDev@10.146.185.63
```

Забираем `user.txt`:

![user.txt](img/Pasted_image_20261119220849.png)

## Повышение прав → valley

В `/home` лежит `valleyAuthenticator`:

![valleyAuthenticator](img/Pasted_image_20261120120303.png)

Бинарь просит логин и пароль:

![Запрос пароля](img/Pasted_image_20261120120405.png)

Скачиваем его на свою машину:

```bash
scp ./valleyAuthenticator kali@192.168.192.3:/home/kali/
```


`strings` по бинарю:

```text
e6722920bab2326f8217e4
bf6b1b58ac
ddJ1cc76ee3
beb60709056cfbOW
elcome to Valley Inc. Authentica
[k0rHh
 is your usernad
Ol: /passwXd.{
```

Строка `e6722920bab2326f8217e4bf6b1b58ac` — MD5-хэш. Крекаем:

![MD5](img/Pasted_image_20261120121001.png)

Пароль — `liberty123`. Логин — `valley`:

```text
valley:liberty123
```

## PrivEsc → root

Смотрим `crontab`:

```bash
cat /etc/crontab
```

![crontab](img/Pasted_image_20261120114528.png)

Видим:

```text
python3 /photos/script/photosEncrypt.py
```

Смотрим скрипт:

```bash
cat /photos/script/photosEncrypt.py
```

```python
#!/usr/bin/python3
import base64
for i in range(1,7):
    image_path = "/photos/p" + str(i) + ".jpg"
    with open(image_path, "rb") as image_file:
        image_data = image_file.read()
    encoded_image_data = base64.b64encode(image_data)
    output_path = "/photos/photoVault/p" + str(i) + ".enc"
    with open(output_path, "wb") as output_file:
        output_file.write(encoded_image_data)
```

Скрипт импортирует `base64`, и запускается по cron от root. Ищем, где лежит `base64.py`:

```bash
find / -name "base64.py" 2>/dev/null
# /usr/lib/python3.8/base64.py
```

Мы входим в группу `valleyAdmin`, поэтому можем редактировать этот файл:

![valleyAdmin](img/Pasted_image_20261120121821.png)

Добавляем reverse shell в `base64.py`:

![Reverse shell в base64.py](img/Pasted_image_20261120122141.png)

Ждём срабатывания cron — получаем root-шелл:

![root shell](img/Pasted_image_20261120122530.png)

## Итоговая схема (Kill Chain)

`HTTP + null byte (dev notes)` → `dev.js (siemDev:california)` → `FTP 37370 (pcapng)` → `Wireshark (valleyDev:ph0t0s1234)` → `SSH` → `valleyAuthenticator (MD5 → liberty123)` → `cron photosEncrypt.py + base64.py hijack (valleyAdmin)` → `root`