# Write-up: OverTheWire Bandit


---

## Level 0

Подключаемся:

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

Пароль: `bandit0`

Читаем `readme`:

```bash
cat readme
```

```text
The password you are looking for is: 6y2kwnwK6grgvwvpvLaa2T1cpFEKOhNR
```

Пароль: `6y2kwnwK6grgvwvpvLaa2T1cpFEKOhNR`

---

## Level 0 → 1

В домашней директории лежит файл с именем `-`. Обычный `cat -` не сработает, потому что `-` для многих утилит означает stdin. Передаём путь явно:

```bash
cat ~/'-'
```

Пароль: `PK8fYLZg2hnHSz83plBL1iEPKdD3QToB`

---

## Level 1 → 2

Файл называется `--spaces in this filename--`. Пробелы и ведущие дефисы ломают разбор аргументов, поэтому берём имя в кавычки:

```bash
cat ~/"--spaces in this filename--"
```

Пароль: `7ZZ2LFrykP2zEyvBl4m3clcL7tGYJPME`

---

## Level 2 → 3

В директории `inhere/` есть скрытый файл (начинается с точки). `ls` без `-a` его не покажет:

```bash
cd inhere/
ls -la
cat ...Hiding-From-You
```

Пароль: `xzTXq1rDJQVVAzdv5cHq1TQytTWufAMq`

---

## Level 3 → 4

Файлы в `inhere/` называются `-file00` … `-file09`. Ведущий дефис снова мешает — используем `./`. Сначала определяем тип каждого файла через `file`, ищем человекочитаемый ASCII:

```bash
file ./​*
cat ./-file07
```

Пароль: `6C7h9GD8M6ai5nr7wo1RonrzFjj9yIrG`

---

## Level 4 → 5

Нужен файл размером ровно 1033 байта, человекочитаемый, не исполняемый. Ищем через `find`:

```bash
find ~/inhere/ -size 1033c -type f
```

Находим `/home/bandit5/inhere/maybehere07/.file2`:

```bash
cat /home/bandit5/inhere/maybehere07/.file2
```

Пароль: `pXa26xhMWaC2SvDotA4r9EgZkulOeSBW`

---

## Level 5 → 6

Файл где-то в системе, владелец `bandit7`, группа `bandit6`, размер 33 байта. Ищем по всей ФС и глушим ошибки доступа:

```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

Находим `/var/lib/dpkg/info/bandit7.password`:

```bash
cat /var/lib/dpkg/info/bandit7.password
```

Пароль: `Bmnnvf82KzQlfxgAI2d1zYbr1u9pr3E3`

---

## Level 6 → 7

В `data.txt` нужно найти строку со словом `millionth`:

```bash
grep "millionth" data.txt
```

Пароль: `VR1ljMayciFxbnUokuQmJFw6QC9VKtub`

---

## Level 7 → 8

Нужна единственная строка, встречающаяся один раз. Сортируем и ищем уникальную:

```bash
sort data.txt | uniq -u
```

Пароль: `EjmOSvuAu7sGAHqHVcBDPirRe9T03kxl`

---

## Level 8 → 9

В файле есть непечатаемые символы, но нужная строка — человекочитаемая, перед ней `=`. `strings` вытаскивает печатаемые последовательности:

```bash
strings data.txt | grep =
```

Пароль: `B0s2khmbT9u0geKuOoVGW3JZKhndE3BG`

---

## Level 9 → 10

Файл закодирован в base64:

```bash
cat data.txt | base64 -d
```

Пароль: `pYfOY6HwUsDj5rL9UvyhU7MCmv8vN5Ro`

---

## Level 10 → 11

ROT13. `tr` со сдвигом на 13 букв:

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Пароль: `GROozWPO8QyN0mGrjUkID0WCYkZiQxrN`

---

## Level 11 → 12

Файл — hexdump со вложенными слоями сжатия и архивов. Работаем в `/tmp`, потому что домашняя директория только для чтения. На каждом шаге смотрим `file`, переименовываем под нужное расширение и распаковываем.

```bash
mkdir /tmp/test512 && cp data.txt /tmp/test512/ && cd /tmp/test512
xxd -r data.txt dec
mv dec dec.gz && gunzip dec.gz
mv dec dec.bz2 && bzip2 -d dec.bz2
mv dec dec.gz && gunzip dec.gz
mv dec dec.tar && tar -xvf dec.tar
tar -xvf data5.bin
mv data6.bin data6.bz2 && bzip2 -d data6.bz2
tar -xvf data6
mv data8.bin data8.gz && gunzip data8.gz
cat data8
```

Пароль: `qQYQiHOBPR8zR61qxYqX45quvihF2uzk`

---

## Level 12 → 13

Дан приватный SSH-ключ. Копируем его на свою машину, ограничиваем права (иначе ssh откажется) и заходим с ним:

```bash
chmod 600 sshkey.private
ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
```

Внутри:

```bash
cat /etc/bandit_pass/bandit14
```

Пароль: `aaWecNkG4FhxJQxz07uiwzVP6bJiYS65`

---

## Level 13 → 14

На `localhost:30000` слушает сервис, который проверяет пароль текущего уровня и отдаёт следующий. Отправляем пароль через `nc`:

```bash
nc localhost 30000
```

Пароль: `pbLYuZtTg4MgaqfJx8jbA9gKKGqM68A7`

---

## Level 14 → 15

На `localhost:30001` тот же чекер, но за TLS. Подключаемся через `openssl s_client`, отправляем пароль:

```bash
openssl s_client -connect localhost:30001
```

Пароль: `kS0Hf0u5HiXFwKMKFqXvPdOTNGGa0X8V`

---

## Level 15 → 16

На диапазоне `31000–32000` несколько портов. `nmap -sV` показывает, что `31790` — SSL-сервис. Подключаемся и отправляем текущий пароль:

```bash
openssl s_client -quiet -connect localhost:31790
```

Сервис отдаёт приватный SSH-ключ для `bandit17`. Сохраняем, `chmod 600`, заходим.

Пароль: `OQxXZjELndr90zuhOTDYBEomI0SZITXI` (получен через `diff passwords.new passwords.old`).

---

## Level 17 → 18

Два файла `passwords.new` и `passwords.old`. Ищем различающуюся строку:

```bash
diff passwords.new passwords.old
```

Новая строка — пароль для bandit18.

Пароль: `KpsOfPkcP7i1FlIExk2QEjyt6dw8dxZI`

---

## Level 18 → 19

При логине shell читает `.bashrc`, который сразу выкидывает нас. Поэтому выполняем команду напрямую через ssh:

```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme
```

Пароль: `KpsOfPkcP7i1FlIExk2QEjyt6dw8dxZI` (для чтения `readme` в домашней директории bandit18), результат — пароль bandit19.

Пароль bandit19: `4pIjcunZ0fK2vmp3IwfG8Vf7VhxD6pOA`

---

## Level 19 → 20

В домашней директории лежит SUID-бинарь `bandit20-do`, который позволяет выполнять команды от имени bandit20:

```bash
./bandit20-do cat /etc/bandit_pass/bandit20
```

Пароль: `4pIjcunZ0fK2vmp3IwfG8Vf7VhxD6pOA` → `4pIjcunZ0fK2vmp3IwfG8Vf7VhxD6pOA` (bandit20).

---

## Level 20 → 21

`suconnect` — SUID-бинарь, который подключается к указанному порту, читает пароль текущего уровня и, если он верный, отдаёт пароль следующего. Поднимаем свой слушатель, отправляем пароль:

```bash
nc -lvp 1234
# в другом окне:
./suconnect 1234
```

Пароль: `bW9kBv5WC3P4yoDyf12LSdGuNz5ka6hY`

---

## Level 21 → 22

Смотрим cron-задания:

```bash
cat /etc/cron.d/cronjob_bandit22
cat /usr/bin/cronjob_bandit22.sh
```

Скрипт кладёт пароль bandit22 в `/tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv`. Читаем:

```bash
cat /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
```

Пароль: `RYVux2rHEm9tiXHmLFzuR7Vhx6AZQMEz`

---

## Level 22 → 23

Снова cron. Скрипт считает md5 от строки `I am user bandit23` и кладёт пароль в `/tmp/<md5>`. Вычисляем имя файла сами:

```bash
cat /tmp/$(echo I am user bandit23 | md5sum | cut -d ' ' -f 1)
```

Пароль: `gKXDTAXnIz3OBxiPjRZ2uqutUlPZrBsw`

---

## Level 23 → 24

Здесь cron выполняет любой скрипт из `/var/spool/bandit24/foo/`. Пишем свой скрипт, который копирует пароль bandit24 в наш `/tmp`:

```bash
cat > /tmp/test512/script.sh <<'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/test512/pass.txt
EOF
chmod 777 -R /tmp/test512
cp /tmp/test512/script.sh /var/spool/bandit24/foo/
```

Ждём минуту, читаем `/tmp/test512/pass.txt`.

Пароль: `hVQMk3lJNsmQ7VF3ubyrNNBom7BOgVXv`

---

## Level 24 → 25

На `localhost:30002` чекер принимает `password pin`. Пин — 4 цифры. Перебираем в цикле:

```bash
for code in {1000..9999}; do
  payload="hVQMk3lJNsmQ7VF3ubyrNNBom7BOgVXv $code"
  response=$(echo "$payload" | ncat localhost 30002)
  if [[ "$response" != *"Wrong! Please enter the correct current password and pincode. Try again."* ]]; then
    echo "$response"
    break
  fi
  sleep 0.1
done
```

Пароль: `SoHfqMOEqIX2IYKVciZxvgpR9a2Djx4P`

---

## Level 25 → 26

При логине shell bandit26 запускает `/usr/bin/showtext`, который делает `exec more ~/text.txt`. `more` при узком окне терминала переходит в постраничный режим — и оттуда можно вызвать shell через `:shell`. Нужно сильно уменьшить окно терминала, чтобы `more` запагинировал.

```bash
ssh -i bandit26.sshkey bandit26@bandit.labs.overthewire.org -p 2220
```

Как только попали в `more`, жмём `v` (или `:shell`, в зависимости от версии), попадаем в vi, оттуда `:shell` → `$SHELL`.

Внутри:

```bash
./bandit27-do cat /etc/bandit_pass/bandit27
```

Пароль: `STJLJBRRphMxKB392CT4iOr5CbzPU9ER` (в прохождении был записан без хвоста — используйте актуальное значение).

---

## Level 26 → 27

Из шелла bandit26 используем SUID-бинарь `bandit27-do`:

```bash
./bandit27-do cat /etc/bandit_pass/bandit27
```

Пароль: `STJLJBRRphMxKB392CT4iOr5CbzPU9ER`

---

## Level 27 → 28

Клонируем git-репозиторий (пароль — от bandit27-git):

```bash
git clone ssh://bandit27-git@bandit.labs.overthewire.org:2220/home/bandit27-git/repo
cd repo && cat README
```

Пароль: `y8Yd2ssKcpHpud7UvOSOxwamRMzIGIeQ`

---

## Level 28 → 29

В `README.md` пароль замазан. Смотрим историю коммитов — в одном из предыдущих коммитов пароль был в открытом виде:

```bash
git log
git checkout df10da19c2625f7c59e071e16030bd587d8c409d
cat README.md
```

Пароль: `Em7eGtqaMySwNFjCpwzzHhLhospOcdt0`

---

## Level 29 → 30

В `master` пароля нет. Смотрим все ветки:

```bash
git branch -a
git checkout dev
cat README.md
```

В ветке `dev` лежит пароль.

Пароль: `jq9Dfg2rXsfYsWMgFuKlXhphjdH7USgX`

---

## Level 30 → 31

Обычный `README.md` пустой. Пароль спрятан в git-теге:

```bash
git tag
git show secret
```

Пароль: `82NkymblpGBYmIXG6ZQ8YldBYstHpfUf`

---

## Level 31 → 32

Нужно запушить файл `key.txt` с содержимым `May I come in?` в ветку `master`. `.gitignore` блокирует `key.txt`, поэтому добавляем через `-f` или правим `.gitignore`:

```bash
echo -n "May I come in?" > key.txt
git add -f key.txt
git commit --amend -m "Add key.txt correctly"
git push -f
```

Сервер в ответ отдаёт пароль:

```text
remote: Well done! Here is the password for the next level:
remote: pWuj5jBQ6IgV0NXwiH6g1pXRF8S1YvbT
```

> В логе видно ещё и `remote: Wrong!` — это хук сервера проверяет содержимое; если бы содержимое было неверным, пуш был бы отклонён (`pre-receive hook declined`). У нас сработало, потому что содержимое корректное.

Пароль: `pWuj5jBQ6IgV0NXwiH6g1pXRF8S1YvbT`

---

## Level 32 → 33

При логине запускается «UPPERCASE SHELL», которая переводит ввод в верхний регистр. Поэтому команды не работают — `cat` превращается в `CAT`. Пробуем переменные:

- `$0` — имя текущего шелла (например, `sh`), его перевод в верхний регистр ничего не ломает, потому что `$0` подставляется *после* uppercasing;
- `$1`, `$2` и т.д. — позиционные параметры.

Вводим `$0`, получаем интерактивный shell в обход uppercasing и читаем пароль:

```bash
$0
cat /etc/bandit_pass/bandit33
```

Пароль: `u4P2CyPOwPGLe94RdD9Uo2FxFwvnFswM`

---

## Level 33 → 34

Финальный уровень. Читаем `README.txt` — поздравление, дальше уровней нет.

```bash
cat README.txt
```

Поздравляю, Bandit пройден.
