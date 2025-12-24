## Коды возврата в Bash / Exit Codes in Bash

**RU:**
Каждая команда в Bash после завершения возвращает числовой код (от 0 до 255), называемый **Exit Status** или **Return Code**. Этот код сообщает операционной системе (или родительскому скрипту), успешно ли выполнилась задача.

Основное правило: **0 — успех, любое другое число — ошибка.**

**EN:**
Every Bash command, upon completion, returns a numeric code (from 0 to 255) known as the **Exit Status** or **Return Code**. This code tells the operating system (or parent script) whether the task was successful.

The golden rule is: **0 means success, any other number means failure.**

---

### 1. Как проверить код? / How to Check the Code?

**RU:**
Для получения кода последней выполненной команды используется специальная переменная `$?`.

**EN:**
To get the exit code of the last executed command, use the special variable `$?`.

```bash
# Пример успеха / Success example
ls /usr/bin
echo $?  # Результат: 0

# Пример ошибки / Error example
ls /non_existent_folder
echo $?  # Результат: 2 (или другое число, отличное от 0)
```

---

### 2. Зарезервированные коды / Reserved Exit Codes

**RU:**
Хотя вы можете использовать любые числа для своих ошибок, Bash имеет зарезервированные значения:

**EN:**
While you can use any number for your own errors, Bash has several reserved values:

| Код / Code | Значение / Meaning | Описание / Description |
| :--- | :--- | :--- |
| **0** | **Success** | Команда выполнена успешно. / Command completed successfully. |
| **1** | **General Error** | Общая ошибка (например, деление на ноль). / Miscellaneous errors. |
| **2** | **Misuse of builtins** | Неправильное использование встроенных команд (неверные аргументы). / Incorrect usage. |
| **126** | **Command invoked cannot execute** | Команда найдена, но нет прав на выполнение. / Permission problem. |
| **127** | **Command not found** | Команда не найдена (ошибка в PATH или опечатка). / Command not found. |
| **128+n** | **Fatal error signal "n"** | Завершение по сигналу (например, `130` = Ctrl+C, т.к. сигнал SIGINT это 2). / Fatal signal. |
| **255** | **Exit status out of range** | Выход за пределы 0-255 (используется `exit -1`). / Exit code out of range. |

---

### 3. Использование в логике / Usage in Logic

**RU:**
Вы можете использовать коды возврата для управления логикой скрипта с помощью операторов `&&` (И) и `||` (ИЛИ).

**EN:**
You can use exit codes to control script flow using `&&` (AND) and `||` (OR) operators.

```bash
# Выполнить вторую команду ТОЛЬКО если первая успешна
# Run second command ONLY if the first one succeeds
mkdir my_logs && touch my_logs/app.log

# Выполнить вторую команду ТОЛЬКО если первая провалилась
# Run second command ONLY if the first one fails
ls /tmp/test_report || echo "Report not found!"
```

#### В блоках IF / Inside IF blocks:
```bash
if grep -q "ERROR" server.log; then
    echo "Found errors in log!"
    exit 1
fi
```

---

### 4. Exit Codes в AutoQA и CI/CD

**RU:**
В автоматизации тестирования коды возврата критически важны:
1. **Fail Fast:** Используйте `set -e` в начале скрипта, чтобы он немедленно прекращал работу, если любая команда вернула ошибку.
2. **Custom Exits:** Если ваши тесты упали, скрипт должен явно вызвать `exit 1`, чтобы CI-система (Jenkins, GitLab CI) пометила билд как упавший.

**EN:**
In test automation, exit codes are crucial:
1. **Fail Fast:** Use `set -e` at the beginning of your script to make it exit immediately if any command fails.
2. **Custom Exits:** If your tests fail, the script must explicitly call `exit 1` so the CI system (Jenkins, GitLab CI) marks the build as failed.

```bash
#!/bin/bash
set -e # Остановить скрипт при первой же ошибке / Exit on error

echo "Running API tests..."
pytest tests/api_tests.py

echo "All tests passed!"
exit 0
```

---

### 5. Резюме / Summary

- `exit 0` — Всё прошло по плану.
- `exit 1` (или выше) — Что-то сломалось.
- Всегда проверяйте `$?` при отладке.
- В CI/CD пайплайнах **всегда** следите за тем, чтобы ваш скрипт возвращал не-ноль при падении тестов.
