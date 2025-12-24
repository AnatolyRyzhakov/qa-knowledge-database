## Переменные окружения в Bash / Environment Variables in Bash

**RU:**
**Переменные окружения** — это динамические значения, которые хранятся внутри системы и доступны как самому шеллу, так и запущенным из него программам. Они используются для настройки среды: путей к файлам, настроек локали, ключей доступа или параметров тестов.

**EN:**
**Environment Variables** are dynamic key-value pairs stored within the system, accessible to the shell and any programs launched from it. They are used to configure the environment: file paths, locale settings, access keys, or test parameters.

---

### 1. Просмотр переменных / Viewing Variables

**RU:**
Вы можете вывести значение конкретной переменной, используя префикс `$`, или список всех переменных с помощью команды `env`.

**EN:**
You can display the value of a specific variable using the `$` prefix, or list all variables using the `env` command.

```bash
echo $USER      # Имя текущего пользователя / Current username
echo $PATH      # Пути поиска исполняемых файлов / Search path for executables
echo $PWD       # Текущая директория / Current directory

# Вывести все переменные окружения
# List all environment variables
env
```

---

### 2. Локальные vs Переменные окружения / Local vs Env Variables

**RU:**
В Bash важно различать локальные переменные (доступны только в текущей сессии шелла) и переменные окружения (доступны и дочерним процессам).

**EN:**
In Bash, it's crucial to distinguish between local variables (available only in the current shell session) and environment variables (available to child processes as well).

```bash
# 1. Локальная переменная (не видна в других программах)
# Local variable (not visible to other programs)
MY_VAR="secret"

# 2. Превращение в переменную окружения
# Turning it into an environment variable
export MY_VAR="secret"
```

---

### 3. Как задать переменную навсегда / Making Variables Persistent

**RU:**
Если просто написать `export`, переменная исчезнет после закрытия терминала. Чтобы сохранить её, нужно добавить команду в файл конфигурации профиля.

**EN:**
If you just use `export`, the variable will vanish after closing the terminal. To make it persistent, add the command to your profile configuration file.

- `~/.bashrc` — для интерактивных сессий Bash.
- `~/.bash_profile` — для login-шеллов (macOS часто использует его).

```bash
# Добавить в конец файла ~/.bashrc
# Add to the end of ~/.bashrc
export API_KEY="12345-qwerty"
```
*После редактирования примените изменения: `source ~/.bashrc`*

---

### 4. Использование в AutoQA / Usage in AutoQA

**RU:**
В автоматизации тестирования переменные окружения — это стандартный способ передачи конфигурации (URL стенда, логины/пароли, тип браузера) без изменения кода тестов.

**EN:**
In test automation, environment variables are the standard way to pass configurations (environment URLs, credentials, browser type) without changing the test code.

#### Запуск тестов с переменной / Running tests with a variable:
```bash
# Передаем URL стенда и браузер прямо в команду запуска
# Passing Base URL and Browser directly into the execution command
BROWSER=chrome BASE_URL=https://staging.example.com pytest tests/
```

#### Доступ из Python (Python access):
```python
import os

# Получаем значение из окружения
browser = os.getenv('BROWSER', 'firefox') # 'firefox' - значение по умолчанию
print(f"Running tests on {browser}")
```

---

### 5. Популярные зарезервированные переменные / Common Reserved Variables

| Переменная / Variable | Описание / Description |
| :--- | :--- |
| `$PATH` | Список папок, где система ищет программы. / Folders to search for commands. |
| `$HOME` | Путь к домашней директории пользователя. / Current user's home directory. |
| `$SHELL` | Путь к текущему шеллу (например, `/bin/bash`). / Path to current shell. |
| `$EDITOR` | Текстовый редактор по умолчанию. / Default text editor. |

---

### 6. Резюме / Summary

- `VAR=val` — создаёт локальную переменную.
- `export VAR=val` — делает переменную доступной для программ и скриптов.
- `env` — показывает все активные переменные окружения.
- Используйте `os.getenv()` в Python для чтения этих настроек.
