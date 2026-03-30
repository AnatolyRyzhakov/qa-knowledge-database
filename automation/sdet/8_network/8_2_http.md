# 📘 Глава 8. Сети и Протоколы (Уровень OSI)
## Тема 8.2. Эволюция веба: HTTP/2 vs HTTP/3 (QUIC, 0-RTT, Multiplexing)

### Предисловие:
Эволюция протокола HTTP — это одна из самых "дорогих" тем на архитектурных интервью в BigTech компаниях. В 2026 году HTTP/3 (на базе QUIC) уже не является экспериментальной технологией — это стандарт де-факто для мобильных и высоконагруженных приложений, поддерживаемый Nginx, Caddy и глобальными CDN [8].

Для Senior SDET понимание HTTP/3 необходимо, так как классические инструменты нагрузочного и API-тестирования, заточенные под TCP, могут давать искаженные результаты при тестировании современных сетей.

### Введение: Стандарты и Метрики Производительности
В рамках стандарта качества программного обеспечения **ISO/IEC 25010** ключевую роль играет характеристика **Performance Efficiency** (Эффективность производительности), в частности подхарактеристика **Time behaviour** (Временное поведение: TTFB, Latency) [6]. 
Переход от HTTP/2 к HTTP/3 был продиктован не функциональными пробелами, а именно физическими ограничениями производительности в нестабильных мобильных сетях (High-latency, Packet Loss) [2][5].

---

### Часть 1. Проблема HTTP/2: TCP Head-of-Line Blocking

Для понимания HTTP/3 нужно знать, что именно было сломано в HTTP/2.

Протокол HTTP/2 внедрил **Мультиплексирование (Multiplexing)**. Вместо того чтобы открывать 6 разных TCP-соединений для скачивания 6 картинок (как в HTTP/1.1), HTTP/2 начал скачивать все картинки параллельно внутри *одного* TCP-соединения [1][5]. 

**Архитектурный крах (TCP-level HOL Blocking):** 
HTTP/2 решил проблему блокировки на прикладном уровне (L7), но столкнулся с транспортным (L4)[8]. TCP — это протокол строгой очередности (Ordered byte stream). Если во время передачи 100 мультиплексированных потоков теряется хотя бы *один* сетевой пакет (например, от первой картинки), протокол TCP **блокирует абсолютно все потоки**, пока потерянный пакет не будет переотправлен и доставлен. В сетях с потерей пакетов более 2% старый HTTP/1.1 с шестью параллельными соединениями парадоксально работал быстрее, чем современный HTTP/2 [8].

---

### Часть 2. HTTP/3 и магия QUIC

HTTP/3 отказывается от TCP. Он работает поверх протокола **QUIC (Quick UDP Internet Connections)**, который базируется на UDP [1][2].

**Три суперсилы QUIC (Что спрашивают на интервью):**

1. **Истинное мультиплексирование (Без HOL Blocking):** В QUIC каждый поток (Stream) полностью независим. Если пакет из Потока №1 теряется, Потоки №2–100 продолжают загружаться без пауз [2][8].
2. **Встроенное шифрование (Mandatory TLS 1.3):** В TCP шифрование настраивалось отдельным слоем. QUIC интегрирует "рукопожатие" (Handshake) TLS прямо в транспортный слой, сокращая время установки безопасного соединения [2][5].
3. **Миграция соединений (Connection Migration):** В TCP соединение привязано к IP-адресу (4-Tuple). Если пользователь вышел из дома (Wi-Fi) и переключился на 5G (сменился IP), TCP-соединение рвется. QUIC идентифицирует сессию по уникальному `Connection ID`. Загрузка фильма не прервется при смене сети [1][8].

---

### Часть 3. 0-RTT (Zero Round Trip Time) и риски безопасности

**1-RTT** (один цикл туда-обратно) требуется QUIC для нового клиента, чтобы договориться о ключах шифрования [3].
**0-RTT** — это функция HTTP/3 для *возвращающихся* пользователей. Клиент запоминает криптографические параметры прошлой сессии и отправляет HTTP-запрос (например, `GET /api/data`) **в самом первом пакете**, не дожидаясь ответа сервера [1][4]. Время установки соединения падает до нуля [4].

🛡️ **SDET Security Alert (Атака повторного воспроизведения - Replay Attack):**
Данные 0-RTT не защищены от повторного воспроизведения хакером на уровне сети. На архитектурном ревью Senior SDET обязан проверить: **0-RTT должен быть разрешен ТОЛЬКО для идемпотентных запросов (GET, HEAD)**. Если конфигурация сервера (Nginx/Caddy) позволяет принимать `POST` или `DELETE` запросы через 0-RTT (например, `/api/transfer-money`), злоумышленник может перехватить пакет и отправить его 5 раз, списав деньги 5 раз [3].

---

### Часть 4. Инженерия тестирования HTTP/3 (Как это делает SDET)

Тестирование HTTP/3 требует специфической конфигурации фреймворков.

#### 1. Проверка доступности (Alt-Svc)
HTTP/3 не может быть запущен по умолчанию (браузер изначально не знает, поддерживает ли сервер UDP). Обнаружение происходит через заголовок ответа `Alt-Svc` (Alternative Service) при первоначальном обращении по HTTP/1.1 или HTTP/2 [10].
В API-тестах SDET проверяет этот заголовок:
```bash
curl -I https://example.com | grep alt-svc
# Ожидаемый результат: Alt-Svc: h3=":443"; ma=86400 
```

#### 2. Настройка Playwright / Puppeteer для HTTP/3
Современные версии браузеров имеют поддержку QUIC, но в headless-режиме ее часто нужно форсировать (Enforcing) через аргументы командной строки [11].
```python
# Пример запуска Playwright с принудительным HTTP/3[11]
browser = await playwright.chromium.launch(
    args=[
        "--enable-quic",
        "--origin-to-force-quic-on=www.example.com:443"
    ]
)
```

#### 3. Нагрузочное тестирование (Performance Paradox)
*Ловушка для автоматизатора:* Если вы запустите k6 или JMeter для тестирования HTTP/3 внутри идеальной сети дата-центра (Datacenter-to-Datacenter), **HTTP/2 покажет лучшие результаты, чем HTTP/3** [3].
*Почему?* Реализация TCP глубоко оптимизирована в ядре Linux, в то время как QUIC часто работает в "пространстве пользователя" (user-space), создавая накладные расходы процессора (Context-switch overhead) [3].

**Правильный подход (The Last Mile Testing):** HTTP/3 выигрывает только на "последней миле" (от сотовой вышки до телефона). Senior SDET обязан использовать инструменты симуляции сети (например, `Toxiproxy`), искусственно внедряя **2–5% потери пакетов (Packet Loss)** и высокий Latency. Именно в этих условиях HTTP/3 покажет ускорение загрузки на 40–50% по сравнению с HTTP/2 [1][8].

---

### Часть 5. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns
1.  **Нагрузочное тестирование по умолчанию:** Использование стандартных сэмплеров (samplers) в инструментах нагрузки, которые не умеют работать с UDP. Ваши тесты покажут 100% покрытие, но де-факто они будут идти по фолбэку (Fallback) через HTTP/2.
2.  **Игнорирование UDP-файрволов:** Настройка CI/CD агентов без открытия порта `443 UDP`. Браузер будет пытаться инициировать QUIC-соединение, "отваливаться" по таймауту и возвращаться на TCP, что увеличит время прогона тестов.
3.  **Использование 0-RTT для изменения состояния (Mutations):** Одобрение релизов, где `POST` запросы обслуживаются сервером через "Early Data" (0-RTT) [3].

#### ✅ Best Practices
1.  **Continuous Validation (Проверка деградации):** Написание Python-скриптов с использованием клиента `quiche` или `curl --http3`, чтобы непрерывно проверять, что инфраструктура не откатилась на HTTP/2 из-за неправильного конфига Nginx [10].
2.  **Мониторинг `Alt-Svc` заголовка:** Автотесты API (слой L7) всегда должны ассертить наличие заголовка `Alt-Svc: h3=":443"`. Это часть Quality Gates.
3.  **Тестирование Connection Migration:** В E2E мобильных сценариях (Appium) программное переключение сети на эмуляторе с Wi-Fi на Cellular. Автотест должен проверить, что загрузка файла по HTTP/3 не прервалась (в отличие от TCP).

***

### 📚 Sources (Источники)
1. [HTTP vs. HTTP/2 vs. HTTP/3: What's the Difference? - PubNub (Nov 2023)](https://www.pubnub.com/blog/http-vs-http-2-vs-http-3-whats-the-difference/)
2. [HTTP/3 vs. HTTP/2 — Understanding the Key Differences - Hike SEO (Sep 2025)](https://www.hikeseo.co/learn/technical/http-3-vs-http-2)
3. [Understanding Head-of-Line Blocking: HTTP/2 vs. HTTP/3 (QUIC) in Production - Substack (Mar 2026)](https://systemdr.substack.com/p/understanding-head-of-line-blocking)
4. [HTTP/3 is Fast! Multiplexing & 0-RTT - Request Metrics (Feb 2025)](https://requestmetrics.com/web-performance/http3-is-fast/)
5. [Enhancing API Performance with HTTP/2 and HTTP/3 Protocols - Zuplo (Aug 2025)](https://zuplo.com/learning-center/enhancing-api-performance-with-http-2-and-http-3-protocols)
6. [ISO/IEC 25010: Performance Efficiency & Time Behaviour - ISO25000.com](https://iso25000.com/index.php/en/iso-25000-standards/iso-25010)
7. [Evaluating System Quality Using ISO/IEC 25010 - UNM OJS (2025)](https://journal.unm.ac.id/index.php/JESSI/article/view/10266)
8. [HTTP/3 and QUIC in Production: A Practical Deployment Guide for 2026 - DEV Community (Mar 2026)](https://dev.to/linou518/http3-and-quic-in-production-a-practical-deployment-guide-for-2026-3n8e)
9. [HTTP/3 and QUIC: Prepare your network for testing - Keysight (Jul 2022)](https://www.keysight.com/blogs/en/tech/nwvs/2022/07/08/http3-and-quic-prepare-your-network-for-the-most-important-transport-change-in-decades)
10. [How to Test HTTP/3 Connectivity - OneUptime (Mar 2026)](https://oneuptime.com/blog/test-http3-connectivity-over-ipv6)
11. [HTTP3 vs HTTP2 testing script with Puppeteer/Chrome - GitHub (2025/2026)](https://github.com/KiweeEu/http3-test)
