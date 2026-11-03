# 01. Окружение на macOS, терминал и Git

## 1. Homebrew — менеджер пакетов для Mac

Почти всё дальше ставится через него. Открой **Терминал** (Cmd+Space → «Terminal») и выполни:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

В конце установщик покажет 2–3 команды «Next steps»: **выполни их**, иначе `brew` не будет найден.
Проверка: `brew --version`.

> На Mac с процессором Apple Silicon (M1–M4) Homebrew ставится в `/opt/homebrew`, на Intel в `/usr/local`. Это нормально.

## 2. Go

```bash
brew install go
go version        # должна вывестись версия
go env GOPATH     # обычно ~/go
```

Чтобы бинарники, установленные через `go install`, запускались из терминала, добавь в `~/.zshrc`
(на Mac по умолчанию оболочка zsh):

```bash
echo 'export PATH="$PATH:$(go env GOPATH)/bin"' >> ~/.zshrc
source ~/.zshrc
```

## 3. Редактор

- **VS Code** (бесплатно): `brew install --cask visual-studio-code`, затем расширение **Go** (от Go Team at Google).
  При первом открытии .go-файла оно предложит установить инструменты (gopls, dlv и др.): соглашайся.
- **GoLand** (JetBrains) — мощнее, но платный (есть пробный период и скидки для студентов).

Полезные настройки VS Code: форматирование при сохранении (`editor.formatOnSave`), организация импортов.

## 4. Остальные инструменты

```bash
brew install git golangci-lint jq
brew install --cask orbstack    # Docker для Mac (лёгкий и быстрый)
# или: brew install --cask docker   (Docker Desktop)
```

- **OrbStack / Docker Desktop** — запускать базы данных, Redis, Kafka в контейнерах. После установки открой приложение один раз.
- **golangci-lint** — линтер, находит ошибки и плохой стиль.
- **jq** — удобно смотреть JSON в терминале.
- Для работы с API: `curl` уже есть на Mac. Можно поставить Postman или Bruno (`brew install --cask bruno`).

Проверка: `docker run hello-world`.

## 5. Терминал: минимум, который нужно знать

| Команда | Что делает |
|---|---|
| `pwd`, `ls -la`, `cd`, `mkdir -p`, `touch` | навигация и создание |
| `cp`, `mv`, `rm -r` | копировать, переместить, удалить |
| `cat`, `less`, `head`, `tail -f` | смотреть файлы и логи |
| `grep -rn "text" .` | поиск по файлам |
| `find . -name "*.go"` | поиск файлов |
| `ps aux`, `kill PID`, `lsof -i :8080` | процессы и кто занял порт |
| `export VAR=value`, `env` | переменные окружения |
| `cmd1 \| cmd2`, `>`, `>>` | пайпы и перенаправление вывода |
| `curl -X POST -d '{}' -H 'Content-Type: application/json' localhost:8080/x` | HTTP-запросы |
| `chmod +x script.sh` | сделать файл исполняемым |

Ресурс: «The Missing Semester of Your CS Education» (MIT, есть русские субтитры на YouTube): лекции про shell, git, отладку.

## 6. Git

```bash
git config --global user.name "Твоё Имя"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

SSH-ключ для GitHub:

```bash
ssh-keygen -t ed25519 -C "you@example.com"
pbcopy < ~/.ssh/id_ed25519.pub   # скопирует ключ в буфер (это macOS-команда)
```
Затем GitHub → Settings → SSH and GPG keys → New SSH key → вставить.

Основные команды: `init`, `clone`, `status`, `add`, `commit`, `log --oneline --graph`, `branch`, `switch -c`, `merge`, `rebase`, `push`, `pull`, `stash`, `diff`, `reset`, `revert`.

Ресурс: **learngitbranching.js.org** — интерактивный тренажёр (есть русский язык).

## Задачи

- [ ] ⭐ **Установка.** Поставь всё из этого файла. Убедись, что работают `go version`, `docker run hello-world`, `golangci-lint --version`.
- [ ] ⭐ **Первая программа.**
  ```bash
  mkdir -p ~/code/go-learning/01-setup/hello && cd $_
  go mod init github.com/<твой-ник>/go-learning/hello
  ```
  Напиши `main.go`, который печатает «Hello, Go!», текущую дату и имя пользователя из переменной окружения `USER`.
  Запусти через `go run .`, собери бинарник `go build -o hello` и запусти `./hello`.
- [ ] ⭐ **Репозиторий.** Создай на GitHub репозиторий `go-learning`, добавь `.gitignore` (бинарники, `.DS_Store`, `.env`), запушь первую программу.
- [ ] ⭐ **Терминал.** Одной командой посчитай, сколько строк во всех `.go` файлах внутри папки (подсказка: `find` + `xargs` + `wc -l`).
- [ ] ⭐⭐ **Порты.** Запусти `python3 -m http.server 8080` в одном окне. В другом найди процесс, который слушает порт 8080, и завершь его через `kill`.
- [ ] ⭐⭐ **Git-ветки и конфликт.** Создай ветку `feature`, измени в ней строку в файле. Вернись в `main`, измени ту же строку по-другому. Сделай `merge`, получи конфликт и разреши его. Потом повтори через `rebase`.
- [ ] ⭐⭐ **Git-тренажёр.** Пройди на learngitbranching.js.org разделы «Main» 1–4 и «Remote» 1–2.
- [ ] ⭐⭐ **Pull Request.** Сделай ветку, внеси изменение, запушь и открой Pull Request на GitHub сам себе. Посмотри diff, смержи.
