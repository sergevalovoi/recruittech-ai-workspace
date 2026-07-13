# Troubleshooting — подключение Telegram (Windows + Docker)

Ошибки по этапам, в порядке возникновения.

## До установки

### Нет Docker
https://docker.com/products/docker-desktop (~1GB, ~4GB на диске). При установке — включить **WSL 2**.

### Не хватает места на диске
Проверить: `Get-PSDrive C | Select-Object Free`. Освободить место или почистить старые Docker-образы: `docker system prune -a`.

## Логин на my.telegram.org

### Появилась CAPTCHA
Решить вручную в браузере, продолжить логин, при необходимости зайти в "API development tools" самостоятельно.

### Код подтверждения не приходит

С февраля 2023 Telegram у части аккаунтов молча не присылает код при новой API-сессии — ни в приложение, ни SMS. Известный баг upstream (Telethon #3835, #4041, #4050), надёжного фикса нет.

**Не помогает:** ждать дольше, запрашивать код повторно (триггерит FloodWait), VPN, переустановка Telegram.

**Что делать:** проверить ВСЕ устройства с Telegram (телефон/десктоп/веб), подождать 2 полные минуты. Если так и не пришёл — здесь описан только вариант с обычным логином (телефон+код); если он не работает для конкретного аккаунта, это тупик для данного пути, нужен другой номер/аккаунт.

### "Short name already taken"
Имя приложения должно быть уникальным глобально. Добавить цифры/инициалы: `имя_claude_2026`.

### "Too many tries" / rate limit
my.telegram.org лимитирует попытки логина. Подождать 15-30 минут, не жать "Send code" повторно.

## Генерация session string

### PHONE_NUMBER_INVALID
Международный формат: `+79123456789`, не `89123456789`.

### PHONE_CODE_INVALID
Использовать самый свежий код (Telegram может присылать несколько). Код истекает через 5 минут.

### PASSWORD_HASH_INVALID (ошибка 2FA)
Проверить caps lock. Если пароль забыт — Telegram > Настройки > Приватность > Двухэтапная проверка > Забыли пароль (нужна почта восстановления). Без неё — тупик, нужен другой аккаунт.

### FloodWait при авторизации
Слишком много попыток. Ошибка содержит время ожидания (например "FloodWait 300" = 5 минут). В крайних случаях — до 24 часов. Не повторять попытки до истечения таймера.

### Генерация зависла / нет ответа
Ctrl+C, проверить интернет, повторить через минуту. Если используется VPN — попробовать без него (или наоборот, если Telegram заблокирован в регионе).

### "docker: command not found"
Docker Desktop не в PATH — перезапустить PowerShell или использовать полный путь `& "C:\Program Files\Docker\Docker\resources\bin\docker.exe"`.

## После установки

### "Could not find the input entity"
Проблема формата chat ID:
- Пользователи: числовой ID или username без @
- Каналы: username (без @) или числовой ID
- Супергруппы: добавить `-100` перед числовым ID (например `-1001234567890`)

### Черновики не появляются в Telegram
Открыть нужный чат в приложении, проскроллить в конец, при необходимости перезапустить Telegram. Черновики — по каждому чату отдельно.

### Session expired / "AUTH_KEY_UNREGISTERED"
Session string больше не валиден — сгенерировать новый:
```powershell
docker run -it --rm bayramannakov/telegram-mcp:latest python session_string_generator.py
```
Затем обновить переменную окружения:
```powershell
[System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', 'НОВАЯ_СТРОКА', 'User')
```
Перезапустить PowerShell/Claude Code.

### "AuthKeyDuplicatedError" — Telegram навсегда убил ключ сессии

**Симптом:** telegram-mcp подключается, но инструменты падают с ошибкой о том, что один auth key использовался с двух IP одновременно.

**Причина:** один и тот же session string использовался **двумя живыми подключениями одновременно** — два окна Claude Code, или интерактивное окно + фоновая задача. Отличается от `AUTH_KEY_UNREGISTERED` (это истёкшая/завершённая сессия) — здесь ключ убит навсегда, восстановить нельзя, только перевыпустить.

**Восстановление:** сгенерировать новый session string (шаги выше), обновить переменную окружения.

**Постоянное решение:** каждой фоновой задаче/дополнительному окну — свой отдельный session string, не делить один ключ.

### MCP-сервер не отвечает

Проверить, что команда в `claude mcp add` использует `docker run --rm -i`, а не `docker run --rm` — флаг `-i` держит stdin открытым, без него Docker сразу закрывает stdin и MCP не может общаться с Claude Code.

```powershell
claude mcp remove telegram-mcp
claude mcp add telegram-mcp -s user -- docker run --rm -i -e TELEGRAM_API_ID -e TELEGRAM_API_HASH -e TELEGRAM_SESSION_STRING bayramannakov/telegram-mcp:latest python -c "import builtins,sys;_p=builtins.print;builtins.print=lambda *a,**k:_p(*a,**dict(k,file=k.get('file',sys.stderr)));from main import main;main()"
claude mcp list
```
Перезапустить Claude Code.

### Telegram-инструменты молча не работают
Самая частая причина — **Docker Desktop не запущен**. Проверить иконку в системном трее, запустить, подождать 1-2 минуты. Перезапуск MCP-регистрации не нужен — просто запусти Docker.

### "claude: command not found"
Если `claude` установлен через npx, а не глобально:
```powershell
npm install -g @anthropic-ai/claude-code
```

### `claude mcp add` зависает внутри Claude Code
Запускать `claude mcp add` только из отдельного окна PowerShell, не из активной сессии Claude Code — иначе команда пытается создать вложенный процесс Claude и виснет.

### MCP возвращает ошибки / нечитаемый ответ
Причина: образ Docker пишет `print()` в stdout до старта MCP-протокола, что портит JSON-RPC (MCP использует stdout только для протокола). Команда регистрации в Шаге 4 уже содержит обёртку `python -c ...`, которая перенаправляет `print()` в stderr — если ошибка повторяется, проверить, что команда регистрации скопирована **целиком**, без обрезки.

### Несколько Docker-образов telegram-mcp
Нормально при обновлении образа. Почистить неиспользуемые: `docker image prune`.

## Частные случаи

### Несколько аккаунтов Telegram
У каждого — свой session string. Активен может быть только один telegram-mcp за раз. Для переключения: сгенерировать session string другого аккаунта, обновить переменную окружения, перезапустить Claude Code.

### Корпоративный VPN блокирует Telegram
Отключить VPN на время установки, или попросить IT добавить в белый список API-эндпоинты Telegram.

### Смена номера / включение 2FA после установки
Session string остаётся валидным при смене номера и включении/выключении 2FA (это токен уже после авторизации). «Завершить все другие сессии» в Telegram — **обнуляет** session string, нужна новая генерация.

## Если ничего не помогло

1. Проверить Docker Desktop запущен
2. Пере-сгенерировать session string с нуля (Шаг 2 основного скилла)
3. Issues upstream-инструмента: https://github.com/chigwell/telegram-mcp/issues
