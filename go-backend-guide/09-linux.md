# 09. Linux для бэкендера

Почти все Go-сервисы работают в Linux: на сервере или в контейнере. На собеседованиях спрашивают про процессы, сигналы, порты, права и логи, а на работе это нужно каждый день, когда что-то сломалось в проде.

**Когда проходить:** после файла 08 (Docker). Базовые команды терминала уже были в файле 01. Тему удобно растянуть на 1–2 недели по 20–30 минут в день параллельно со следующими файлами.

## Где практиковаться на Mac

macOS — тоже Unix, многие команды совпадают, но `ss`, `systemctl`, `journalctl`, `/proc` и `strace` есть только в Linux. Варианты:

```bash
# 1. Быстро: контейнер Ubuntu (без systemd)
docker run -it --rm --name lab ubuntu:24.04 bash
apt update && apt install -y procps iproute2 lsof curl strace htop dnsutils netcat-openbsd jq

# 2. Полноценная виртуалка с systemd
brew install --cask multipass
multipass launch --name dev
multipass shell dev
```

Для тем про systemd и journald нужна виртуалка, для остального хватит контейнера.

## Процессы

- **PID** — id процесса, **PPID** — id родителя. Процессы образуют дерево (`ps -ef --forest`, `pstree`).
- `ps aux`, `top`, `htop` — что запущено и сколько ест CPU и памяти.
- `/proc/<pid>/` — всё о процессе: `status` (память, потоки), `fd/` (открытые файлы), `limits`, `environ`, `cmdline`.
- **Zombie** — процесс завершился, а родитель не прочитал его код возврата (`wait`). **Orphan** — родитель умер, процесс «усыновляет» PID 1.
- Код возврата: `echo $?` после команды. 0 — успех, остальное — ошибка. `os.Exit(1)` в Go задаёт именно его.

## Сигналы (спрашивают почти всегда)

| Сигнал | Номер | Когда приходит | Можно перехватить? |
|---|---|---|---|
| SIGINT | 2 | Ctrl+C в терминале | да |
| SIGTERM | 15 | `kill <pid>` по умолчанию, `docker stop`, Kubernetes при остановке пода | да |
| SIGKILL | 9 | `kill -9`, OOM killer, Kubernetes после `terminationGracePeriodSeconds` | **нет** |
| SIGHUP | 1 | закрылся терминал; часто используют для перечитывания конфига | да |
| SIGQUIT | 3 | Ctrl+\ | да; Go по умолчанию **печатает стеки всех горутин** и выходит, удобно для отладки зависаний |

Go-сервис должен ловить SIGTERM и завершаться аккуратно (graceful shutdown из файла 06):

```go
sigs := make(chan os.Signal, 1)
signal.Notify(sigs, syscall.SIGTERM, syscall.SIGINT, syscall.SIGHUP)

for sig := range sigs {
    switch sig {
    case syscall.SIGHUP:
        log.Info("reloading config")
        if err := cfg.Reload(); err != nil {
            log.Error("reload", "err", err)
        }
    default:
        log.Info("shutting down", "signal", sig)
        shutdown() // srv.Shutdown(ctx), закрыть БД, дождаться воркеров
        return
    }
}
```

### Ловушка с PID 1 в контейнере
В Dockerfile `CMD ./app` (shell-форма) запускает `/bin/sh -c ./app`. PID 1 становится `sh`, а он не пересылает SIGTERM твоему приложению. В итоге `docker stop` ждёт 10 секунд и убивает всё через SIGKILL: graceful shutdown не срабатывает.
Правильно: **exec-форма** `CMD ["./app"]` или `ENTRYPOINT ["./app"]`. Если приложение само запускает дочерние процессы, добавь init-процесс (`docker run --init` или `tini`), чтобы он собирал зомби.

## Файлы, дескрипторы, права

- «Всё есть файл»: обычные файлы, сокеты, пайпы — у процесса это **файловые дескрипторы** (fd).
- Лимит открытых fd: `ulimit -n`, у процесса — `/proc/<pid>/limits`. Ошибка `too many open files` в Go обычно значит, что где-то не закрывается `resp.Body`, файл или соединение.
- `lsof -p <pid>` — какие файлы и сокеты открыл процесс. `ls -l /proc/<pid>/fd | wc -l` — сколько их.
- Права `rwx` для владельца, группы и остальных: `chmod 644 file` (rw-r--r--), `chmod 755 script` (rwxr-xr-x), `chown user:group file`.
- В контейнере приложение лучше запускать не от root: `USER 65532` в Dockerfile (distroless-образы так и делают).

## Сеть

```bash
ss -tlnp                  # какие TCP-порты слушаются и каким процессом
ss -tanp | grep 8080      # соединения на порту 8080 и их состояния
ss -s                     # сводка: сколько соединений в ESTAB, TIME-WAIT
curl -v http://localhost:8080/health   # заголовки запроса и ответа, TLS
nc -zv db.local 5432      # открыт ли порт
dig example.com           # DNS: какие IP вернулись
ip addr                   # интерфейсы и адреса
cat /etc/hosts /etc/resolv.conf
```

Знать: состояния TCP (LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT), что значит `0.0.0.0:8080` против `127.0.0.1:8080` (почему сервис в контейнере, слушающий 127.0.0.1, недоступен снаружи), что много CLOSE_WAIT обычно значит, что твой код не закрывает соединения.

## Ресурсы, память, cgroups

```bash
free -h          # память
df -h            # место на дисках
du -sh *         # что занимает место в папке
vmstat 1         # CPU, память, swap раз в секунду
dmesg | grep -i oom   # убивал ли процесс OOM killer
```

- Контейнер ограничивается через **cgroups**: лимиты CPU и памяти из Docker и Kubernetes — это они.
- Превысил лимит памяти → ядро убивает процесс (OOMKilled, код выхода 137 = 128 + 9).
- В Go: `GOMEMLIMIT` подсказывает GC, когда собирать мусор агрессивнее (ставят около 80–90% лимита контейнера). Начиная с Go 1.25 `GOMAXPROCS` по умолчанию учитывает CPU-лимит cgroup, раньше для этого брали `go.uber.org/automaxprocs`.

## systemd и логи

Без Kubernetes сервис на сервере обычно запускают через systemd:

```ini
# /etc/systemd/system/tasks-api.service
[Unit]
Description=Tasks API
After=network-online.target

[Service]
ExecStart=/usr/local/bin/tasks-api
EnvironmentFile=/etc/tasks-api.env
User=tasks
Restart=on-failure
RestartSec=3
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tasks-api   # запустить и добавить в автозагрузку
systemctl status tasks-api
journalctl -u tasks-api -f              # логи в реальном времени
journalctl -u tasks-api --since "10 min ago"
```

## Работа с текстом (логи, конфиги)

```bash
tail -f app.log | grep ERROR
grep -rn "TODO" ./internal
# топ-10 IP из access-лога
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
# только поле msg из JSON-логов (slog)
cat app.log | jq -r 'select(.level=="ERROR") | .msg'
sed -i 's/8080/9090/' config.env
find . -name "*.go" -mtime -1      # изменённые за сутки
```

Плюс: пайпы `|`, перенаправления `>`, `>>`, `2>&1`, `xargs`, переменные окружения (`export`, `env`), `crontab -e` для задач по расписанию.

## Отладка

- `strace -f -p <pid>` — какие системные вызовы делает процесс (висит на `read`? на `connect`?).
- `curl localhost:6060/debug/pprof/goroutine?debug=2` — если в сервисе подключён pprof (файл 16).
- `kill -QUIT <pid>` — дамп горутин Go-процесса в stderr.

## Ресурсы
- *The Linux Command Line* (William Shotts) — бесплатная книга, linuxcommand.org
- **explainshell.com** — разбирает любую команду по частям
- **man**-страницы: `man ss`, `man 7 signal`
- Julia Evans, зины про Linux и сети (wizardzines.com)

## Задачи
- [ ] ⭐ **Процессы руками.** В контейнере Ubuntu запусти `sleep 1000 &`, найди его через `ps`, посмотри `/proc/<pid>/status`, убей через SIGTERM. Повтори с `kill -9`.
- [ ] ⭐ **Разбор логов.** Сгенерируй Go-программой файл на 10 000 строк JSON-логов со случайными уровнями и путями. Одной командой выведи топ-5 путей с ошибками (`jq` + `sort` + `uniq -c`).
- [ ] ⭐⭐ **Сигналы в Go.** Программа ловит SIGTERM (graceful shutdown с таймаутом), SIGHUP (перечитывает конфиг из файла) и логирует каждый сигнал. Проверь через `kill -HUP`, `kill -TERM`, `kill -QUIT`.
- [ ] ⭐⭐ **PID 1.** Собери образ Tasks API с `CMD ./app` и с `CMD ["./app"]`. Сравни `time docker stop` в обоих случаях и логи завершения. Объясни разницу.
- [ ] ⭐⭐ **Утечка дескрипторов.** Напиши HTTP-клиент, который в цикле делает запросы и не закрывает `resp.Body`. Смотри на рост fd через `ls /proc/<pid>/fd | wc -l` и на соединения через `ss`. Исправь и сравни.
- [ ] ⭐⭐ **OOM.** Запусти Go-программу, которая постепенно выделяет память, в контейнере с `--memory=100m`. Посмотри код выхода (137) и `docker inspect` (OOMKilled). Затем поставь `GOMEMLIMIT=80MiB` для программы, которая создаёт много мусора, и сравни поведение.
- [ ] ⭐⭐⭐ **systemd-сервис.** В виртуалке multipass разверни бинарник Tasks API через unit-файл: автозапуск, перезапуск при падении, конфиг через `EnvironmentFile`, логи через journald. Убей процесс через `kill -9` и проверь, что systemd его поднял.

## Вопросы для самопроверки
1. Чем SIGTERM отличается от SIGKILL? Что происходит при `docker stop` и при удалении пода в Kubernetes?
2. Почему shell-форма `CMD` в Dockerfile ломает graceful shutdown?
3. Что такое файловый дескриптор? Откуда берётся `too many open files`?
4. Как узнать, какой процесс занял порт 8080?
5. Что значит код выхода 137? Что такое cgroups?
6. Чем zombie-процесс отличается от orphan?
7. Почему сервис в контейнере, слушающий `127.0.0.1`, недоступен с хоста?
