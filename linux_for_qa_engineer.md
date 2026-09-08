# Linux для QA Engineer

Практический конспект Linux для QA Engineer: только то, что реально помогает тестировать приложения, API, сервисы и окружение, читать логи и диагностировать проблемы на CI/CD или тестовом сервере.

---

## 1. Linux: базовое понимание

**Linux** — ядро ОС. На практике обычно говорят о Linux-дистрибутиве: Ubuntu, Debian, RHEL, Rocky, AlmaLinux и т.д.

Что важно QA:

- понимать файловую систему;
- уметь работать с процессами;
- читать логи;
- проверять сеть и порты;
- понимать права доступа;
- запускать и останавливать сервисы;
- находить причины `permission denied`, `connection refused`, `no such file`, `no space left`;
- работать с SSH;
- выполнять базовую диагностику окружения.

### Основные каталоги

| Каталог | Что обычно находится |
|---|---|
| `/` | корень файловой системы |
| `/home` | домашние каталоги пользователей |
| `/etc` | конфигурация системы и сервисов |
| `/var/log` | логи |
| `/tmp` | временные файлы |
| `/opt` | дополнительное ПО |
| `/usr/bin` | исполняемые программы |
| `/usr/lib` | библиотеки |
| `/dev` | устройства |
| `/proc` | информация о процессах и ядре |
| `/boot` | файлы загрузки системы |

Для QA особенно полезны:

```bash
/etc
/var/log
/tmp
/home
/proc
```

---

# 2. Основные команды

## Навигация

```bash
pwd                 # текущий каталог
ls                  # содержимое
ls -la              # включая скрытые файлы
cd /path             # перейти
cd ..                # на уровень выше
cd ~                 # домашний каталог
```

## Работа с файлами

```bash
touch test.txt       # создать файл
mkdir test           # создать каталог
mkdir -p a/b/c       # создать вложенные каталоги
cp file.txt backup/  # копировать
mv old.txt new.txt   # переместить/переименовать
rm file.txt          # удалить
rm -r directory      # удалить каталог
```

## Чтение файлов

```bash
cat file.txt
less file.txt
head file.txt
tail file.txt
tail -f app.log
```

Для QA особенно важна команда:

```bash
tail -f app.log
```

Она позволяет смотреть лог в реальном времени во время выполнения теста.

## Поиск

```bash
find /path -name "*.log"
find /path -type f -name "config*"
grep "ERROR" app.log
grep -i "error" app.log
grep -R "timeout" /var/log/
```

Комбинация:

```bash
tail -f app.log | grep -i error
```

показывает только ошибки, появляющиеся в логе.

---

# 3. Работа с файлами и диском

## Размер файлов

```bash
ls -lh file.log
du -sh /path/to/directory
df -h
```

### `df` vs `du`

- `df` — сколько места свободно на файловой системе;
- `du` — сколько места занимают конкретные файлы/каталоги.

Например:

```bash
df -h
du -sh /var/log/*
```

Если приложение внезапно перестало писать логи, одна из первых проверок:

```bash
df -h
```

---

# 4. Права доступа

Linux использует права:

- `r` — read;
- `w` — write;
- `x` — execute.

Категории:

- `u` — owner;
- `g` — group;
- `o` — others.

Проверить:

```bash
ls -l file.txt
```

Пример:

```text
-rw-r--r-- 1 user user 1234 file.txt
```

Это означает:

```text
owner  -> rw-
group  -> r--
others -> r--
```

## Частые права

```text
644 -> rw-r--r--
755 -> rwxr-xr-x
600 -> rw-------
```

### chmod

```bash
chmod 644 config.yaml
chmod 755 script.sh
chmod +x script.sh
```

### chown

```bash
chown user:group file.txt
```

### Почему это важно QA

Типичная ошибка:

```text
Permission denied
```

Проверяем:

```bash
ls -l file
whoami
id
```

Для каталога важно понимать:

- `r` — можно посмотреть список файлов;
- `w` — можно создавать/удалять файлы;
- `x` — можно заходить в каталог и обращаться к объектам внутри.

---

# 5. Переменные окружения

Environment variables часто используются приложениями и тестами:

```text
BASE_URL
API_URL
DB_HOST
DB_PORT
ENV
TOKEN
```

Посмотреть:

```bash
env
printenv
echo $PATH
echo $BASE_URL
```

Создать:

```bash
export BASE_URL=https://test.example.com
```

Запустить приложение с переменной:

```bash
BASE_URL=http://localhost:8080 ./app
```

### Почему важно QA

Одна и та же сборка может работать по-разному из-за:

- неправильного `BASE_URL`;
- неправильного `DB_HOST`;
- отсутствующей переменной;
- неправильного окружения;
- отсутствующего secret/config.

---

# 6. Процессы

**Процесс** — запущенный экземпляр программы.

Посмотреть процессы:

```bash
ps aux
ps -ef
```

Интерактивно:

```bash
top
htop
```

Найти процесс:

```bash
ps aux | grep nginx
pgrep nginx
```

Получить PID:

```bash
pgrep nginx
```

## Завершение процесса

Корректно:

```bash
kill <PID>
```

Принудительно:

```bash
kill -9 <PID>
```

По возможности сначала используют обычный `SIGTERM`, а `SIGKILL` оставляют для случаев, когда процесс не завершается нормально.

## Что такое PID

PID — уникальный идентификатор процесса.

Например:

```bash
ps -p 1234 -f
```

---

# 7. Состояния процессов

В `ps` можно увидеть состояние процесса:

| Статус | Значение |
|---|---|
| `R` | выполняется или готов к выполнению |
| `S` | ожидает событие |
| `D` | ожидает I/O |
| `T` | остановлен |
| `Z` | zombie |

Для QA достаточно понимать:

- `R` — процесс работает;
- `S` — обычно нормально, процесс ждёт;
- `D` — возможна проблема с I/O;
- `Z` — завершился, но родитель еще не забрал его статус.

---

# 8. Сервисы и systemd

На современных Linux-серверах часто используется `systemd`.

Основные команды:

```bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
```

Проверить, запущен ли сервис:

```bash
systemctl is-active nginx
```

Посмотреть упавшие сервисы:

```bash
systemctl --failed
```

### `restart` vs `reload`

`restart`:

```text
остановить -> запустить
```

`reload`:

```text
перечитать конфигурацию без полного перезапуска
```

Конкретное поведение зависит от сервиса.

### Почему это важно QA

Если тестируемый сервис не отвечает:

```bash
systemctl status myapp
journalctl -u myapp -n 100
```

---

# 9. Логи

Логи — один из главных инструментов QA при исследовании дефектов.

Ищем:

- `ERROR`;
- `WARN`;
- `Exception`;
- `Timeout`;
- `Connection refused`;
- `Permission denied`;
- `500`;
- stack trace;
- идентификатор запроса;
- timestamp.

## Файловые логи

```bash
tail -f /var/log/app.log
tail -n 100 /var/log/app.log
grep -i "error" /var/log/app.log
```

## systemd journal

```bash
journalctl
journalctl -u myapp
journalctl -u myapp -n 100
journalctl -u myapp -f
journalctl -b
```

### Фильтрация по времени

```bash
journalctl --since "10 minutes ago"
journalctl --since "2026-09-08 15:00"
```

### Почему `journalctl -u SERVICE` полезен

Можно быстро получить логи конкретного сервиса без поиска нужного файла в `/var/log`.

---

# 10. История команд

```bash
history
history | grep ssh
history | grep systemctl
```

История Bash обычно хранится в:

```bash
~/.bash_history
```

---

# 11. SSH

SSH используется для удаленного доступа к Linux-серверам.

Подключение:

```bash
ssh user@host
```

Подробный режим:

```bash
ssh -v user@host
```

Проверка порта:

```bash
nc -vz host 22
```

## SSH-ключи

Обычно:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

или старый вариант:

```text
~/.ssh/id_rsa
~/.ssh/id_rsa.pub
```

**Приватный ключ нельзя передавать другим людям.**

## Копирование файлов

```bash
scp file.txt user@host:/tmp/
scp user@host:/tmp/file.txt .
```

Для больших каталогов/синхронизации:

```bash
rsync -av folder/ user@host:/tmp/folder/
```

---

# 12. Сеть

Для QA важно уметь ответить на вопросы:

- доступен ли сервер;
- разрешается ли DNS;
- открыт ли порт;
- слушает ли приложение нужный порт;
- устанавливается ли TCP-соединение;
- отвечает ли HTTP API.

## IP и интерфейсы

```bash
ip addr
ip link
ip route
```

## DNS

```bash
nslookup example.com
dig example.com
```

## Проверка доступности

```bash
ping example.com
```

Важно: отсутствие ответа на `ping` не обязательно означает, что HTTP/SSH недоступен — ICMP может быть заблокирован.

## Проверка TCP-порта

```bash
nc -vz example.com 443
```

## Какие порты слушаются

```bash
ss -tulpen
ss -lntp
```

Например:

```bash
ss -lntp | grep 8080
```

Если приложение должно работать на `8080`, но ничего не слушает:

```text
Connection refused
```

— вероятно, сервис не запущен, слушает другой порт или доступ блокируется.

---

# 13. curl — главный инструмент QA

`curl` позволяет тестировать HTTP/HTTPS API прямо из терминала.

## GET

```bash
curl https://example.com
```

## Заголовки

```bash
curl -I https://example.com
```

## Подробный запрос

```bash
curl -v https://example.com
```

## POST JSON

```bash
curl -X POST https://example.com/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}'
```

## Authorization

```bash
curl https://example.com/api/user \
  -H "Authorization: Bearer TOKEN"
```

## Сохранить ответ

```bash
curl -o response.json https://example.com/api/data
```

## Проверка HTTP-кода

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
```

### Что проверять QA

При API-тестировании полезно смотреть:

- HTTP status;
- response headers;
- response body;
- время ответа;
- redirect;
- TLS;
- cookies;
- authentication;
- connection errors.

---

# 14. HTTP-коды

| Код | Значение |
|---|---|
| `200` | OK |
| `201` | Created |
| `204` | No Content |
| `301/302` | Redirect |
| `400` | Bad Request |
| `401` | Authentication required/failed |
| `403` | Forbidden |
| `404` | Not Found |
| `405` | Method Not Allowed |
| `409` | Conflict |
| `422` | Unprocessable Content |
| `429` | Too Many Requests |
| `500` | Internal Server Error |
| `502` | Bad Gateway |
| `503` | Service Unavailable |
| `504` | Gateway Timeout |

---

# 15. 502 / 503 / 504

Это особенно важно при тестировании web-приложений.

### 502 Bad Gateway

Proxy получил некорректный ответ от upstream или не смог корректно с ним взаимодействовать.

Например:

```text
Nginx -> application
```

Проверяем:

```bash
systemctl status nginx
systemctl status myapp
ss -lntp
curl -v http://127.0.0.1:3000
journalctl -u nginx -n 100
```

### 503 Service Unavailable

Сервис временно недоступен.

Возможные причины:

- приложение остановлено;
- перегрузка;
- health check не проходит;
- сервис исключен из балансировщика.

### 504 Gateway Timeout

Gateway/proxy не дождался ответа upstream.

Проверяем:

- latency приложения;
- состояние upstream;
- сеть;
- timeout;
- нагрузку;
- логи proxy и приложения.

---

# 16. DNS

DNS преобразует доменные имена в IP и другие записи.

Основные записи:

| Тип | Назначение |
|---|---|
| `A` | IPv4 |
| `AAAA` | IPv6 |
| `CNAME` | псевдоним имени |
| `MX` | почтовый сервер |
| `TXT` | текстовые данные, например SPF/DKIM/verification |

Проверка:

```bash
dig example.com
dig A example.com
dig CNAME example.com
dig MX example.com
```

### Типичная проблема

Приложение работает по IP, но не по домену.

Проверяем:

```bash
dig example.com
curl -v https://example.com
```

---

# 17. Порты и соединения

Порт идентифицирует сетевой endpoint внутри хоста.

Часто встречаются:

| Порт | Сервис |
|---:|---|
| `22` | SSH |
| `53` | DNS |
| `80` | HTTP |
| `443` | HTTPS |
| `3306` | MySQL |
| `5432` | PostgreSQL |
| `6379` | Redis |
| `8080` | часто web/application |

Не стоит считать, что конкретный сервис обязан использовать стандартный порт — его можно изменить в конфигурации.

---

# 18. Firewall

Firewall фильтрует сетевой трафик.

Для QA важно понимать:

```text
приложение работает
        ↓
порт слушается
        ↓
но firewall может блокировать доступ
```

На Linux можно встретить:

- `nftables`;
- `iptables`;
- `firewalld`;
- `ufw`.

Проверка firewalld:

```bash
firewall-cmd --state
firewall-cmd --list-all
```

Для Ubuntu часто используется:

```bash
ufw status
```

Не нужно глубоко изучать правила firewall для обычного QA. Важно уметь отличить проблему приложения от проблемы сетевого доступа.

---

# 19. Память

Основная команда:

```bash
free -h
```

Также:

```bash
top
htop
cat /proc/meminfo
```

В `free` важно понимать:

- `total` — вся RAM;
- `used` — используемая память;
- `free` — непосредственно свободная;
- `available` — сколько памяти примерно доступно приложениям без серьезного давления на swap;
- `buff/cache` — кэш и buffers.

### Swap

```bash
swapon --show
free -h
```

Большой и постоянный swap может указывать на нехватку RAM и приводить к деградации производительности.

---

# 20. CPU и load average

```bash
uptime
top
nproc
```

`load average` показывает среднюю нагрузку системы за:

```text
1 / 5 / 15 минут
```

Важно сравнивать load с количеством CPU.

Например:

```text
4 CPU
load average: 4.0
```

не означает автоматически проблему.

Высокий load может быть связан не только с CPU, но и с ожиданием I/O.

---

# 21. Диск и I/O

Если тесты неожиданно стали медленными, проблема может быть не в приложении, а в диске.

Проверить:

```bash
df -h
iostat -xz 1
vmstat 1
```

Если `iostat` отсутствует, обычно он входит в пакет `sysstat`.

Признаки возможной I/O-проблемы:

- высокий `await`;
- большая очередь;
- высокий `%util`;
- процессы в состоянии `D`;
- высокий `iowait`.

---

# 22. `No space left on device`

Ошибка может означать не только отсутствие гигабайтов.

Проверить:

```bash
df -h
df -i
```

Возможные причины:

1. закончились блоки;
2. закончились inode;
3. достигнута quota.

Если:

```text
df -h -> место есть
df -i -> 100%
```

значит закончились inode — слишком много файлов.

---

# 23. `df` показывает много занятого, а `du` мало

Частая причина — удаленный файл продолжает удерживаться открытым процессом.

Проверить:

```bash
lsof +L1
```

Пример:

```text
app -> /var/log/app.log (deleted)
```

Файл уже удален из каталога, но место еще занято, пока процесс не закроет файловый дескриптор.

---

# 24. Файловые дескрипторы

Процесс работает не только с файлами, но и с:

- sockets;
- pipes;
- devices;
- files.

Каждый открытый объект представлен файловым дескриптором.

Посмотреть:

```bash
ls -l /proc/<PID>/fd
lsof -p <PID>
```

Ошибка:

```text
Too many open files
```

обычно означает достижение лимита файловых дескрипторов.

Проверить:

```bash
ulimit -n
cat /proc/<PID>/limits
```

---

# 25. Файлы конфигурации

Многие Linux-приложения хранят конфигурацию в:

```text
/etc
```

Например:

```bash
/etc/nginx/
/etc/ssh/
/etc/systemd/
```

Для конкретного сервиса удобно:

```bash
systemctl cat nginx
```

Это помогает понять:

- какой бинарник запускается;
- с какими аргументами;
- какой пользователь используется;
- где могут находиться конфиги;
- какие environment variables передаются.

---

# 26. Симлинки

Symbolic link — ссылка на путь другого объекта.

Создание:

```bash
ln -s /path/original /path/link
```

Проверка:

```bash
ls -l
readlink -f link
```

Если target удален, ссылка становится broken symlink.

---

# 27. `/proc`

`/proc` — виртуальная файловая система с информацией от ядра.

Для QA полезно:

```bash
/proc/<PID>/status
/proc/<PID>/cmdline
/proc/<PID>/environ
/proc/<PID>/fd/
/proc/<PID>/limits
```

Например:

```bash
cat /proc/1234/status
cat /proc/1234/cmdline
tr '\0' '\n' < /proc/1234/environ
```

---

# 28. Exit code

Каждая команда возвращает код завершения.

```bash
echo $?
```

Обычно:

```text
0    -> успешно
!= 0 -> ошибка
```

Пример:

```bash
curl https://example.com
echo $?
```

Это важно для:

- shell-скриптов;
- CI/CD;
- автотестов;
- проверки команд в pipeline.

---

# 29. Bash: полезный минимум

### Переменные

```bash
NAME="test"
echo "$NAME"
```

### Аргументы

```bash
$0   # имя скрипта
$1   # первый аргумент
$2   # второй
$@   # все аргументы
$#   # количество аргументов
$?   # exit code
```

### Условия

```bash
if [ "$STATUS" = "200" ]; then
    echo "OK"
fi
```

### Pipe

```bash
command1 | command2
```

stdout первой команды передается stdin второй.

Пример:

```bash
ps aux | grep nginx
```

### Перенаправление

```bash
command > output.txt
command >> output.txt
command 2> error.txt
command > output.txt 2>&1
```

- `>` — перезаписать;
- `>>` — добавить;
- `2>` — stderr;
- `2>&1` — направить stderr туда же, куда stdout.

---

# 30. Архивы

QA часто получает логи или артефакты в архиве.

### tar

```bash
tar -cf logs.tar logs/
tar -xf logs.tar
```

С gzip:

```bash
tar -czf logs.tar.gz logs/
tar -xzf logs.tar.gz
```

Посмотреть содержимое:

```bash
tar -tf logs.tar.gz
```

### gzip

```bash
gzip file.log
gunzip file.log.gz
```

---

# 31. Полезные команды для диагностики

### Кто я?

```bash
whoami
id
```

### Где я?

```bash
pwd
```

### Что запущено?

```bash
ps aux
systemctl --type=service
```

### Что слушает порт?

```bash
ss -lntp
```

### Доступен ли порт?

```bash
nc -vz host 8080
```

### Работает ли HTTP?

```bash
curl -v http://host:8080
```

### Что с DNS?

```bash
dig example.com
```

### Что с диском?

```bash
df -h
df -i
```

### Что с памятью?

```bash
free -h
```

### Что с CPU/load?

```bash
uptime
top
```

### Что с логами?

```bash
journalctl -u service -n 100
tail -f app.log
```

---

# 32. Типовые QA-сценарии

## Приложение не отвечает

Проверять по порядку:

```bash
systemctl status myapp
ps aux | grep myapp
ss -lntp
curl -v http://127.0.0.1:8080
journalctl -u myapp -n 100
```

Логика:

```text
Процесс существует?
        ↓
Сервис запущен?
        ↓
Порт слушается?
        ↓
HTTP отвечает локально?
        ↓
Есть ошибки в логах?
```

---

## API возвращает 500

Не стоит сразу создавать баг только по статусу.

Проверить:

```bash
curl -v ...
journalctl -u myapp -n 100
```

Искать:

- exception;
- stack trace;
- DB connection error;
- timeout;
- invalid configuration;
- dependency failure.

---

## API возвращает 502

Проверить:

```bash
systemctl status nginx
systemctl status myapp
ss -lntp
curl http://127.0.0.1:<app-port>
journalctl -u nginx -n 100
journalctl -u myapp -n 100
```

Цель — понять, проблема в proxy или upstream.

---

## Тест внезапно стал медленным

Проверить:

```bash
uptime
top
free -h
iostat -xz 1
df -h
```

Смотреть:

- CPU;
- RAM;
- swap;
- load;
- disk I/O;
- свободное место.

---

## Приложение не может создать файл

Проверить:

```bash
ls -ld /path
ls -l /path
whoami
id
df -h
df -i
```

Возможные причины:

- нет `w`;
- нет `x` на каталоге;
- нет места;
- закончились inode;
- quota.

---

## Тесты падают с `Connection refused`

Проверить:

```bash
ss -lntp
systemctl status myapp
nc -vz 127.0.0.1 8080
```

Основные гипотезы:

- приложение не запущено;
- порт неправильный;
- приложение слушает только другой interface;
- firewall;
- сервис упал.

---

## Тесты падают с `Connection timed out`

Отличается от `Connection refused`.

`refused` обычно означает, что соединение дошло до хоста, но никто не принимает его на этом endpoint или соединение активно отвергнуто.

`timeout` означает, что ответа вовремя нет.

Проверяем:

```bash
ping host
nc -vz host 8080
ip route
ss -s
```

Также возможны firewall, routing или проблемы сети.

---

# 33. Минимальный workflow QA при проблеме

Хороший алгоритм:

```text
1. Воспроизвести проблему
        ↓
2. Зафиксировать точное время
        ↓
3. Проверить HTTP/API response
        ↓
4. Проверить состояние сервиса
        ↓
5. Проверить логи
        ↓
6. Проверить CPU/RAM/Disk
        ↓
7. Проверить сеть/DNS/порт
        ↓
8. Сформировать гипотезу
        ↓
9. Проверить гипотезу
        ↓
10. Добавить факты в баг-репорт
```

В баг-репорте полезно указывать:

- окружение;
- timestamp;
- endpoint;
- HTTP method;
- status code;
- request/response;
- correlation/request ID;
- relevant logs;
- шаги воспроизведения;
- expected/actual result.

---

# 34. Что QA не обязательно знать глубоко

Для обычной позиции QA Engineer обычно не требуется глубоко знать:

- внутреннюю реализацию scheduler;
- CFS и `vruntime`;
- написание daemon через `fork()`/`setsid()`;
- реализацию syscalls;
- устройство GRUB;
- BIOS/UEFI internals;
- RAID/LVM administration;
- kernel modules;
- NUMA;
- KVM/QEMU;
- сложные iptables/nftables rules;
- восстановление файловых систем;
- создание собственных systemd units.

Это полезно для DevOps/SRE/System Administration, но для QA обычно имеет низкий приоритет.

---

# 35. Что нужно знать уверенно на собеседовании QA

## Must know

```text
Linux filesystem
cd / ls / pwd
cp / mv / rm
cat / less / head / tail
grep / find
chmod / chown
ps / top / kill
systemctl
journalctl
ssh / scp
ip / ss
ping / nc
curl
df / du
free
environment variables
exit codes
stdout / stderr / pipe
```

## Нужно понимать

```text
процесс и PID
сервис
логирование
права доступа
IP / port
DNS
HTTP status codes
CPU / RAM / disk
connection refused vs timeout
502 / 503 / 504
```

## Можно знать поверхностно

```text
systemd internals
filesystem internals
inode
swap
firewall
mount/fstab
```

---

# 36. Короткая шпаргалка

```bash
# Files
pwd
ls -la
cd
cp
mv
rm
find
grep
cat
less
tail -f

# Permissions
ls -l
chmod
chown
whoami
id

# Processes
ps aux
top
pgrep
kill
kill -9

# Services
systemctl status SERVICE
systemctl restart SERVICE
systemctl --failed

# Logs
journalctl -u SERVICE
journalctl -u SERVICE -f
tail -f app.log
grep -i error app.log

# Network
ip addr
ip route
ss -lntp
ping HOST
nc -vz HOST PORT
dig DOMAIN

# HTTP/API
curl -v URL
curl -I URL
curl -X POST URL
curl -H "Header: value" URL

# Resources
uptime
free -h
df -h
df -i
du -sh PATH
iostat -xz 1

# Environment
env
printenv
echo $VAR
export VAR=value

# Debug
lsof -p PID
lsof +L1
cat /proc/PID/status
cat /proc/PID/limits
```

---

# 37. Что должен уметь QA после изучения этого раздела

QA Engineer должен уметь самостоятельно:

1. Подключиться к Linux-серверу по SSH.
2. Найти нужный файл или лог.
3. Посмотреть лог в реальном времени.
4. Найти ошибку через `grep`.
5. Проверить, запущено ли приложение.
6. Найти PID процесса.
7. Проверить, какой порт слушает приложение.
8. Выполнить API-запрос через `curl`.
9. Проверить HTTP status и response.
10. Проверить DNS.
11. Проверить доступность TCP-порта.
12. Проверить права пользователя.
13. Найти проблемы с диском и памятью.
14. Посмотреть логи systemd-сервиса.
15. Отличить проблему приложения от проблемы окружения.
16. Собрать технические факты для bug report.

Главная цель Linux для QA — **не стать системным администратором, а уметь быстро локализовать проблему и собрать достаточно технической информации, чтобы понять: проблема в тесте, приложении, конфигурации, сети или инфраструктуре.**
