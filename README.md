# Ukrainizator Bot 🇺🇦

**Автоматичний переклад Telegram-повідомлень українською з локальним LibreTranslate та визначенням мови.**
Python 3.11+, FastAPI, HTTPX, SQLite, Telegram Bot API та ngrok.

Python-переробка [оригінального проєкту Roman Halaiko](https://github.com/Roman-Gal/telegram-ukrainian-translator),
початкова ревізія `b4cdb0e`. Оригінальна реалізація Node.js + DeepL доступна у вихідному репозиторії.
Переклад виконує локальний [LibreTranslate](https://github.com/LibreTranslate/LibreTranslate) з моделями Argos Translate.
API-ключ для локального сервісу не потрібен. Бот і сервіс працюють як два Python-процеси:
Telegram → ngrok → бот `8081` → LibreTranslate `5000` → відповідь українською.

## Поведінка

- Локально визначає мову (Lingua для кирилиці, `langdetect` для інших алфавітів); впевнено визначену українську пропускає без запиту до API.
- Інший текст перекладає українською й надсилає відповідь на початкове повідомлення в тому самому чаті/темі.
- Працює з текстом, підписами до медіа, особистими чатами, групами та дописами каналів.
- Для російської додає один із 20 випадкових мовних жартів із ненормативною лексикою до того самого повідомлення.
- За замовчуванням жарт потребує локального score ≥0.75, щонайменше 3 літер і відсутності `і`, `ї`, `є`, `ґ`.
  Для короткої кирилиці після перевірки інших кириличних мов додатково порівнюються українська й російська.
  Якщо API визначив іншу мову, жарт вимикається. Пауза за замовчуванням відсутня: кожне визначене російське повідомлення отримує фразу.
- Не відлунює стікери, повідомлення ботів, `via_bot`, автоматичні пересилання каналів і сторонні команди.
- `/help`, `/status`, `/translate текст`; останню команду також можна дати у відповідь на повідомлення.
- Секрет webhook перевіряється; update спочатку зберігається в SQLite, потім підтверджується Telegram.
- Черга й доставлені частини відповіді переживають перезапуск; тимчасові помилки повторюються до 6 спроб.
- Довгі переклади діляться на частини. HTML/Markdown не інтерпретується, прев’ю посилань вимкнене.

Короткі, змішані, транслітеровані фрази та суржик можуть визначатися неправильно.
Score детектора — евристика, а не гарантія. Неоднозначні повідомлення можна перекласти через `/translate`.

## Запуск у Windows

Відкрийте PowerShell у цій папці:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
.\.venv\Scripts\python.exe -c "import secrets; print(secrets.token_hex(32))"
```

Якщо `.env` уже створений, редагуйте його, не перезаписуючи. У підготовленій локальній копії секрет уже згенерований.
У `.env` вставте **токен нового бота** від [@BotFather](https://t.me/BotFather) і згенерований `WEBHOOK_SECRET`.
Токен планнера не підходить: встановлення іншого webhook для того самого токена перенаправить його оновлення.
Реальні секрети зберігаються тільки локально; `.env`, база, конфігурація ngrok і логи виключені з Git.

```dotenv
TELEGRAM_BOT_TOKEN=YOUR_NEW_BOT_TOKEN
WEBHOOK_SECRET=YOUR_RANDOM_SECRET
PORT=8081
TRANSLATION_PROVIDER=libretranslate
TRANSLATION_API_URL=http://127.0.0.1:5000/translate
TRANSLATION_API_KEY=
API_TIMEOUT_SECONDS=60
```

### Локальний LibreTranslate

Сервіс використовує окремий Python 3.12 environment, щоб залежності моделей не конфліктували з ботом на Python 3.14.
Python 3.12 має бути встановлений та доступний через `py -3.12`.

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\setup-libretranslate.ps1
powershell -ExecutionPolicy Bypass -File .\scripts\start-libretranslate.ps1
```

Перший скрипт встановлює LibreTranslate 1.9.6 у `.tools/libretranslate/.venv`.
Другий запускає `http://127.0.0.1:5000` і при першому запуску завантажує 6 моделей для `en,uk,ru,pl`.
Моделі та кеші лежать у `data/libretranslate/` і повторно не завантажуються без потреби.
Дочекайтеся `Running on http://127.0.0.1:5000`, залиште цю консоль відкритою та запустіть ngrok-скрипт в іншій консолі.
`Ctrl+C` у першій консолі зупинить локальний перекладач.

У підготовленій локальній копії бібліотека та ці моделі **вже встановлені**.
Потрібно лише запустити сервіс, якщо він не працює, і додати токен нового Telegram-бота в `.env`.
LibreTranslate не виставляється через ngrok — доступний локально; назовні виходить тільки webhook бота.

Без прямої мовної пари LibreTranslate використовує проміжну мову, зазвичай англійську.
Тому для російської → української залишайте у списку `en,ru,uk`.
Щоб додати, наприклад, німецьку:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\start-libretranslate.ps1 -Languages en,uk,ru,pl,de -UpdateModels
```

Перед зміною списку зупиніть попередній сервіс. `-UpdateModels` оновлює індекс і додає нові пари;
для звичайного запуску цей прапорець не потрібний. Уже встановлені моделі збережуться.
Мови поза завантаженим набором не підтримуються; коротший список не видаляє старі моделі.
Фрази російською, англійською та польською перевірені справжнім локальним перекладом.
Якість місцями буквальна: це машинний переклад, який варто перевіряти перед публікацією важливого тексту.
Джерела: [встановлення LibreTranslate](https://docs.libretranslate.com/guides/installation/),
[мовні моделі й проміжні мови](https://docs.libretranslate.com/guides/supported_languages/).

Перевірити сервіс без Telegram (три фіксовані речення):

```powershell
.\.venv\Scripts\python.exe -X utf8 -m translator.check_translation
```

Бот перевіряє `/languages` та наявність маршрутів `en→uk`, `ru→uk` **до реєстрації webhook**.
Якщо сервіс не запущений чи української моделі немає, запуск зупиняється з помилкою.
Після зміни налаштувань API перезапустіть процес бота.
За потреби режим `TRANSLATION_PROVIDER=disabled` залишає тільки довідку й статус без автоперекладу.

### Автоматичний ngrok

#### Спільний тунель із планнером

Поточна локальна конфігурація використовує один ngrok URL для двох незалежних ботів:

- `/telegram/webhook` → планнер `127.0.0.1:8080`;
- `/translator/telegram/webhook` → перекладач `127.0.0.1:8081`;
- спільний Python-шлюз слухає `127.0.0.1:8082`; ngrok спрямований на цей порт.

У `.env` перекладача задано `PUBLIC_WEBHOOK_PATH=/translator/telegram/webhook`.
Секрети й токени ботів окремі. Код, процес і база планнера не змінюються шлюзом.
Поки ngrok і LibreTranslate працюють, запустити/перевірити перекладач та шлюз можна так:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\start-shared.ps1
```

Обидва Python-процеси запускаються приховано; логи та PID — у `data/`.
Якщо чинний тунель уже веде на `8082`, скрипт використовує його.
Якщо він веде на планнер `8080`, скрипт спочатку пробує створити новий автоматичний HTTPS-тунель на `8082`,
перевіряє його і переприв’язує обидва webhook, зберігаючи новий origin у двох `.env`.
Наявний endpoint не видаляється й не замінюється. Якщо тариф не дозволяє додатковий endpoint, скрипт завершується помилкою.

Зміну існуючого endpoint через `PUT /api/endpoints` не використовуємо: ngrok спершу зупиняє його,
а повторне використання колишнього автоматичного домену може відхилити як платний custom subdomain.
Для нового запуску ngrok вручну використовуйте `ngrok http 8082` і актуальний origin у двох `.env`.
При перезапуску планнера його стандартним `restart.ps1` тунель може знову вести на `8080`;
після цього потрібно повторно підключити спільний режим.

#### Окремий тунель лише для перекладача

Для цього режиму поставте `PUBLIC_WEBHOOK_PATH=/telegram/webhook` у `.env` та перезапустіть процес бота.

```powershell
ngrok config add-authtoken YOUR_NGROK_AUTHTOKEN
powershell -ExecutionPolicy Bypass -File .\scripts\start-ngrok.ps1
```

Скрипт знаходить ngrok у PATH, `.tools/ngrok/ngrok.exe` або в сусідньому `telegram-task-planner/.tools/ngrok/`.
Конфігурацію авторизації бере з цього проєкту, стандартної папки ngrok або сусіднього планнера, у такому порядку.
Початкові файли не змінює; окремим конфігом задає inspector `127.0.0.1:4041`.
Сервер перекладача використовує `8081`, тому не займає стандартний порт планнера `8080`.
ngrok запускається приховано; скрипт знаходить HTTPS URL, передає його боту та реєструє webhook.
`PUBLIC_BASE_URL` у файлі `.env` у цьому режимі можна залишити порожнім. `Ctrl+C` завершує запущений ним ngrok.

Одночасні ngrok agents/endpoints залежать від лімітів вашого облікового запису й доступних доменів.
Роботу з двома тунелями потрібно перевірити на вашому ngrok; при відмові дивіться локальний `data/ngrok.log`.
Скрипт не зупиняє перший бот чи його ngrok. Якщо тариф не дозволяє другий тунель, використайте дозволений окремий endpoint або спільний reverse proxy.

### Ручний ngrok, також Linux/macOS

```sh
ngrok http 8081
```

Вставте HTTPS origin у `.env`: `PUBLIC_BASE_URL=https://your-domain.ngrok-free.app`, без шляху.
В іншому терміналі:

```powershell
.\.venv\Scripts\python.exe -X utf8 -m translator
```

На Linux/macOS: `.venv/bin/python -m translator`. Після зміни URL ngrok перезапустіть бота.
Запуск із `python -m translator` сам реєструє webhook і команди. Черга Telegram при цьому не очищується.
`GET http://127.0.0.1:8081/health` показує роботу сервера та наявність конфігурації перекладу;
це не перевірка доступності зовнішнього API.

## Telegram-група

1. Додайте **нового** бота в групу.
2. Для читання всіх повідомлень вимкніть privacy mode через BotFather (`/setprivacy` → Disable)
   та за потреби приберіть і додайте бота заново. Альтернатива — додати його адміністратором.
3. Дайте право надсилати повідомлення; у каналі потрібне право публікації.
4. За бажанням обмежте чати: `ALLOWED_CHAT_IDS=-1001234567890,123456789`.

Довідка: [Telegram FAQ про privacy mode](https://core.telegram.org/bots/faq#what-messages-will-my-bot-get),
[webhook із secret_token](https://core.telegram.org/bots/api#setwebhook).

## Підключення перекладача

### Власний HTTP-адаптер

Режим `http` — **визначений цим проєктом контракт**, а не універсальний формат будь-якого провайдера.
Для іншого сервісу змініть `HttpTranslator.translate()` у `translator/providers.py` під його запит/відповідь
або поставте перед ним endpoint, що реалізує цей контракт. Решта бота від API не залежить.

```dotenv
TRANSLATION_PROVIDER=http
TRANSLATION_API_URL=https://your-adapter.example/translate
TRANSLATION_API_KEY=YOUR_API_KEY
```

Запит: `POST TRANSLATION_API_URL`, JSON, необов’язковий `Authorization: Bearer <key>`:

```json
{"text":"Hello world!","source_language":"auto","target_language":"uk"}
```

Відповідь:

```json
{"translated_text":"Привіт, світе!","source_language":"en"}
```

`translated_text` обов’язковий. `source_language` — необов’язковий ISO-код джерела, не мови результату.
Невідоме джерело позначайте `unknown`. Автовизначення на боці сервісу не примушується локальною здогадкою.
Порожня/некоректна відповідь відхиляється; вигаданий переклад замість помилки не надсилається.

### LibreTranslate-сумісний endpoint

Це основний адаптер. Він реалізує [офіційний контракт LibreTranslate](https://docs.libretranslate.com/api/operations/translate/):

```dotenv
TRANSLATION_PROVIDER=libretranslate
TRANSLATION_API_URL=http://127.0.0.1:5000/translate
TRANSLATION_API_KEY=
```

Він надсилає `q`, `source=auto`, `target=uk`, `format=text`, необов’язковий `api_key`;
читає `translatedText` та `detectedLanguage.language`. Інстанс має підтримувати українську та потрібні мовні пари.
Перевірено mock-контракт і справжні переклади локальним LibreTranslate 1.9.6.
Також можна вказати URL власного віддаленого LibreTranslate-інстансу та його ключ, якщо він потрібен.
HTTP дозволений лише для localhost, зовнішні API потребують HTTPS. Таймаут — `API_TIMEOUT_SECONDS=60`.

## Жарти

Усі 20 текстів — у `translator/jokes.py`. Можна редагувати їх без зміни коду бота.
Константа набору — `ANTIRUSSIAN_JOKES`.
Набір містить надані користувачем авторські формулювання, збережені дослівно, з ненормативною лексикою.

```dotenv
RUSSIAN_JOKES_ENABLED=true
RUSSIAN_CONFIDENCE=0.75
RUSSIAN_MIN_LETTERS=3
JOKE_COOLDOWN_SECONDS=0
```

## Перевірка й обмеження

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
.\.venv\Scripts\python.exe -m ruff check translator tests scripts/run_libretranslate.py
.\.venv\Scripts\python.exe -m ruff format --check translator tests scripts/run_libretranslate.py
```

55 автоматичних тестів використовують справжній локальний детектор, SQLite і mock HTTP; Telegram, ngrok та платні API не викликаються.
Окремо `translator.check_translation` перевіряє справжній локальний переклад без Telegram.
Бот перевірений на Python 3.14.6 / Windows, LibreTranslate — на Python 3.12.0.
Після зміни коду запущеного бота: `powershell -ExecutionPolicy Bypass -File .\scripts\restart-translator.ps1`.
Цей скрипт робить SQLite-backup і перезапускає тільки перекладач на `8081`, залишаючи планнер, шлюз, ngrok і LibreTranslate.
Для відтворення залежностей бота — `requirements.lock.txt`, окремого сервісу — `requirements-libretranslate.lock.txt`.

Запускайте **один процес і один worker** з конкретною базою. Підтверджені update IDs зберігаються для дедуплікації;
вміст повідомлень очищається після завершення/остаточної помилки. Поки завдання очікує обробки,
оригінал і підготовлена відповідь зберігаються локально в SQLite. SQLite WAL може містити попередні дані;
це не механізм гарантованого стирання. Прямі HTTP URL із токенами й тексти повідомлень у логи бота не пишуться.

Повтор одного `update_id` не створює нове завдання. Проте абсолютна exactly-once доставка неможлива:
якщо Telegram прийняв `sendMessage`, а з’єднання обірвалося до відповіді або процес упав до checkpoint,
повторна спроба може дати дублікат. Остаточні помилки позначаються `failed` і логуються без вмісту.
Редагування повідомлень, голосові, OCR і переклад тексту всередині стікерів не обробляються.
