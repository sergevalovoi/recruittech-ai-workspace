---
name: telegram-mcp-setup
description: |
  Мастер подключения личного Telegram к Claude Code через telegram-mcp (Docker).
  Ведёт через весь процесс: API credentials на my.telegram.org, session string,
  безопасное хранение, регистрация MCP-сервера, проверка подключения.

  Триггеры: "подключить телеграм", "telegram mcp", "connect telegram", "telegram setup",
  "настроить telegram для claude"
license: MIT
compatibility: |
  Windows 11 + Docker Desktop — единственный путь, описанный здесь (вся команда на Windows).
  Тот же образ и тот же паттерн уже работает у Сержа лично — если Docker Desktop запущен,
  подключение стабильно.
metadata:
  version: "1.0.0"
  category: setup
  telegram-mcp-repo: https://github.com/chigwell/telegram-mcp
  source: адаптировано под Windows-команду из MIT-скилла github.com/BayramAnnakov/telegram-mcp-setup
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
---

# Подключение Telegram к Claude Code

Даёт Claude доступ к твоим чатам: читать историю, искать сообщения, готовить черновики ответов — прямо из Claude Code. Используется, например, скиллами `/candidate-research` и `/lead-research`, если кандидат/лид упоминается в Telegram-переписке.

## Глоссарий

- **MCP (Model Context Protocol)** — способ подключения внешних инструментов к Claude Code. Как система плагинов.
- **Session string** — токен авторизации для Telegram-аккаунта. **Чувствительнее пароля** — см. раздел «Риски» ниже.
- **Docker** — запускает telegram-mcp в изолированном контейнере. Нужен Docker Desktop, ~1GB скачать, ~4GB на диске.
- **API ID / API Hash** — бесплатные credentials от Telegram для регистрации своего «приложения».

## Как ходят данные

1. **telegram-mcp работает локально** на твоей машине и забирает сообщения с серверов Telegram.
2. Когда Claude обрабатывает эти сообщения (сортирует, ищет, готовит черновик) — **содержимое уходит в Anthropic API**, как обычный запрос к Claude.
3. **Сам session string остаётся локально** — никуда не отправляется.

То есть: credentials — только локально. Содержимое сообщений — уходит в Anthropic, когда ты просишь Claude что-то с ними сделать. Это то же самое, что вручную вставить текст сообщения в чат с Claude.

## Pre-flight check

Открой PowerShell и проверь:
```powershell
Write-Host "Docker: $(try { docker --version } catch { 'NOT FOUND' })"
Write-Host "Claude CLI: $(try { claude --version } catch { 'NOT FOUND' })"
```

Если Docker не установлен — https://docker.com/products/docker-desktop, при установке включить **WSL 2** (галка при установке или отдельно `wsl --install`). Требует перезагрузки. ~4GB на диске.

**Время:** 10-15 мин если Docker уже стоит, 30-45 мин если ставить с нуля.

## Шаг 1 — API credentials Telegram (~5 мин)

1. Открыть https://my.telegram.org в браузере
2. Ввести номер телефона (+7... для России), код подтверждения из Telegram
3. Нажать **"API development tools"**
4. Если приложения ещё нет — создать:
   - App title: `Claude Code Integration`
   - Short name: уникальное, только буквы, 5-32 символа (например `имя_claude`) — если занято, добавить цифры
   - Platform: Desktop
5. Записать **api_id** (число) и **api_hash** (32-символьная строка)

## Шаг 2 — Session string (~3-5 мин)

Открыть **новое окно PowerShell** (это интерактивный шаг — потребует ввода):

```powershell
docker run -it --rm bayramannakov/telegram-mcp:latest python session_string_generator.py
```

Спросит по очереди: API ID, API Hash, номер телефона (+7...), код подтверждения (проверить все устройства с Telegram), 2FA-пароль (если включён).

**Результат:** строка ~300-400 символов. Если короче 100 символов — что-то пошло не так, повторить.

**Если код не приходит** — подожди 2 минуты, проверь все активные сессии Telegram (телефон/десктоп/веб), не запрашивай коды повторно (это триггерит rate limit). Детали — `references/troubleshooting.md`.

## Шаг 3 — Хранение credentials

Простой вариант — переменные окружения пользователя:

```powershell
[System.Environment]::SetEnvironmentVariable('TELEGRAM_API_ID', 'ТВОЙ_API_ID', 'User')
[System.Environment]::SetEnvironmentVariable('TELEGRAM_API_HASH', 'ТВОЙ_API_HASH', 'User')
[System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', 'ТВОЙ_SESSION_STRING', 'User')
```

Закрыть и открыть PowerShell снова, проверить: `echo $env:TELEGRAM_API_ID`

## Шаг 4 — Регистрация MCP-сервера (~2 мин)

**Важно:** команду `claude mcp add` нельзя запускать из активной сессии Claude Code (зависнет) — открой **отдельное окно PowerShell**.

```powershell
claude mcp add telegram-mcp -s user -- docker run --rm -i -e TELEGRAM_API_ID -e TELEGRAM_API_HASH -e TELEGRAM_SESSION_STRING bayramannakov/telegram-mcp:latest python -c "import builtins,sys;_p=builtins.print;builtins.print=lambda *a,**k:_p(*a,**dict(k,file=k.get('file',sys.stderr)));from main import main;main()"
```

(Оборачивание в `python -c ...` нужно, чтобы `print()` внутри образа не портил протокол MCP — не убирать.)

Проверить: `claude mcp list` — `telegram-mcp` должен быть в списке.

**Docker Desktop должен быть запущен каждый раз**, когда используешь Telegram в Claude Code. Если закрыть Docker Desktop — инструменты Telegram молча перестанут работать (без явной ошибки).

## Шаг 5 — Проверка

1. Полностью перезапустить Claude Code
2. Спросить: «покажи мои чаты в Telegram»
3. Должен появиться список чатов — если нет, см. `references/troubleshooting.md`

## Несколько окон / фоновых задач — избегать AuthKeyDuplicatedError

**Один и тот же session string не может использоваться двумя живыми подключениями одновременно.** Если открыть два окна Claude Code или запустить фоновую задачу поверх интерактивной сессии — Telegram **навсегда убивает ключ** (`AuthKeyDuplicatedError`), спасает только новый session string.

Если планируешь больше одного окна или фоновые задачи — сделай для каждой **отдельный session string** (повторить Шаг 2, сохранить под другим именем переменной) — два разных ключа никогда не конфликтуют, даже одновременно.

## Риски session string

Session string **чувствительнее пароля**:
- Пароль требует 2FA. Session string **обходит 2FA** — это токен уже после аутентификации.
- Действует **бессрочно**, пока сессию явно не завершить в Telegram.
- **Нет уведомления**, если кто-то использует украденный session string.

Если скомпрометирован — злоумышленник читает все чаты, шлёт сообщения от твоего имени, скачивает файлы, видит контакты — и всё это **без единого алерта**.

**Практика безопасности:**
1. Никогда не делиться session string
2. Не автоотправлять сообщения — только черновики (`save_draft`), если это доступно
3. Периодически проверять активные сессии: Telegram > Настройки > Приватность > Активные сессии
4. Обновлять session string раз в ~90 дней как гигиену

## Полный откат (если скомпрометирован или просто отключить)

1. **Завершить сессию в Telegram:** Настройки > Приватность > Активные сессии > найти нужную > Завершить (или «Завершить все другие сессии»)
2. **Удалить переменные окружения:**
   ```powershell
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_API_ID', $null, 'User')
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_API_HASH', $null, 'User')
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', $null, 'User')
   ```
3. **Убрать MCP-регистрацию:** `claude mcp remove telegram-mcp`
4. **Опционально отозвать API credentials** на https://my.telegram.org/apps

## Справочник

- `references/troubleshooting.md` — все известные ошибки по этапам (не приходит код, Docker не стартует, MCP не отвечает, garbled response и т.д.)
- upstream-инструмент: https://github.com/chigwell/telegram-mcp
- Docker-образ: `bayramannakov/telegram-mcp:latest`
