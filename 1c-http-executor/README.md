# 1c-http-executor

Минимальный HTTP-сервис для 1С 8.3, который позволяет внешнему клиенту (включая LLM-агента) выполнять **произвольный код встроенного языка 1С** на сервере: запросы, чтение/создание/изменение/удаление данных, прогон обработки inline, инспекция метаданных, песочница для будущего кода.

Состоит из двух слоёв:

1. **Server-side patch** — два файла в формате выгрузки конфигурации (`onec/`), готовых к `1cv8.exe DESIGNER /LoadConfigFromFiles`. Подкладываешь поверх дампа своей конфигурации, обновляешь БД — сервис работает.
2. **AI subagent** — markdown-описание для Claude Code (`claude/agents/1c-httpExecutor.md`). Любой другой LLM-агент или человек может читать его как system-prompt и работать с тем же контрактом.

URL endpoint'а **не зашит** в агента. Адрес базы пробрасывается через переменную окружения `ONEC_DEV_URL` — благодаря этому один и тот же агент работает с любой базой, где развёрнут серверный слой.

## Что внутри

```
1c-http-executor/
├── LICENSE                                # MIT
├── README.md                              # этот файл
├── onec/                                  # серверный слой (1С dump format)
│   ├── HTTPServices/
│   │   ├── dev.xml                        # метаданные HTTPService.dev
│   │   └── dev/Ext/Module.bsl             # код обработчиков read/write
│   └── Configuration.snippet.xml          # для первой установки в новую конфу
└── claude/agents/1c-httpExecutor.md       # subagent для Claude Code
```

## Установка серверной части (5 минут)

**Требуется:** 1С 8.3 (тестировано на 8.3.22), серверный режим (или файловая БД), любой веб-сервер с публикацией 1С (Apache 2.4 или IIS).

### Если в твоей конфигурации ещё **нет** объекта `HTTPService.dev`:

1. Выгрузи конфигурацию в файлы (Конфигуратор → Конфигурация → Выгрузить конфигурацию в файлы) или возьми существующий дамп.
2. Открой её `Configuration.xml`, найди тег `<ChildObjects>`, добавь строку:
   ```xml
   <HTTPService>dev</HTTPService>
   ```
   Подсказка лежит в [`onec/Configuration.snippet.xml`](onec/Configuration.snippet.xml).
3. Скопируй из репо в дамп два файла:
   - `onec/HTTPServices/dev.xml` → `<твой дамп>/HTTPServices/dev.xml`
   - `onec/HTTPServices/dev/Ext/Module.bsl` → `<твой дамп>/HTTPServices/dev/Ext/Module.bsl`
4. Залей дамп в БД:
   ```
   "C:\Program Files\1cv8\<version>\bin\1cv8.exe" DESIGNER /S "<server>\<base>" ^
       /LoadConfigFromFiles "<путь к дампу>" /DisableStartupDialogs /DisableStartupMessages
   "C:\Program Files\1cv8\<version>\bin\1cv8.exe" DESIGNER /S "<server>\<base>" ^
       /UpdateDBCfg -Server /DisableStartupDialogs /DisableStartupMessages
   ```

### Если объект `HTTPService.dev` уже есть в твоей конфигурации (drop-in update):

Просто перезапиши `dev.xml` и `dev/Ext/Module.bsl` поверх дампа и запусти те же две команды — изменения подхватятся динамически.

### Публикация на веб-сервере

Apache:
```
"C:\Program Files\1cv8\<version>\bin\webinst.exe" -publish -apache24 ^
    -wsdir <publication-name> -dir "<публикационная папка>" ^
    -connstr "Srvr=<server>;Ref=<base>;" ^
    -confPath "<path-to-httpd.conf>"
```

После рестарта Apache endpoint доступен по адресу:
```
http://<host>/<publication-name>/hs/dev/read
http://<host>/<publication-name>/hs/dev/write
```

Запомни этот адрес — он становится значением `ONEC_DEV_URL` (без хвостового `/read|/write`):
```
ONEC_DEV_URL = http://<host>/<publication-name>/hs/dev
```

## Контракт

**Запрос:** `POST $ONEC_DEV_URL/read` (или `/write` — оба эндпойнта вызывают один и тот же обработчик, разделение чисто семантическое).

```json
{
  "code":   "<встроенный язык 1С, последовательность операторов>",
  "params": { ... любая JSON-структура ... }
}
```

Внутри `code` доступны переменные:
- `Параметры` — твой `params` как `Структура` 1С.
- `Результат` — переменная, в которую кладёшь то, что хочешь вернуть. Сериализуется в JSON автоматически: примитивы — нативно, `Дата` → ISO-строка, `ТаблицаЗначений` → массив объектов по колонкам, `Структура`/`Соответствие`/`Массив`/`ДеревоЗначений`/`СписокЗначений` рекурсивно, ссылки на объекты БД — строковым представлением.

**Ответ всегда `200`** (даже при ошибке исполнения кода):
```json
{
  "ok": true | false,
  "result": <сериализованный Результат | null>,
  "error": "<полный текст ошибки или ''>",
  "errorInfo": { "description": "...", "detailed": "..." } | null,
  "executionMs":     <number>,   // длительность Выполнить(Код)
  "serializationMs": <number>    // конвертация Результат → JSON-структура
}
```

`executionMs` и `serializationMs` присутствуют **всегда**: и при успехе, и когда `Выполнить(Код)` бросил исключение. На ранних ошибках (плохой JSON в теле, пустой `code`) оба поля = 0.

HTTP `500` от сервиса означает фатальный сбой в самой обвязке (не в пользовательском коде) — теоретически не должен происходить.

## Быстрая проверка после установки

```powershell
$env:ONEC_DEV_URL = 'http://localhost/yourBase/hs/dev'
$body = '{"code":"Результат = 40 + 2;"}'
$bytes = [Text.Encoding]::UTF8.GetBytes($body)
$req = [System.Net.HttpWebRequest]::Create("$env:ONEC_DEV_URL/read")
$req.Method = 'POST'; $req.ContentType = 'application/json; charset=utf-8'
$req.ContentLength = $bytes.Length
$rs = $req.GetRequestStream(); $rs.Write($bytes,0,$bytes.Length); $rs.Close()
$resp = $req.GetResponse()
(New-Object System.IO.StreamReader($resp.GetResponseStream(),[Text.Encoding]::UTF8)).ReadToEnd()
```

Ожидаемый ответ: `{"ok": true, "result": 42, "error": "", "errorInfo": null, "executionMs": 0, "serializationMs": 0}`.

## Подключение к Claude Code

```
mkdir -p .claude/agents
cp claude/agents/1c-httpExecutor.md .claude/agents/
# выставь ONEC_DEV_URL в окружении или в .claude/settings.json env
```

После рестарта Claude Code subagent появится в picker'е и будет автоматически предлагаться к спавну под задачи вида «прочитай/измени/создай данные в 1С», «прогони код 1С», «верни метаданные конфигурации».

## Подключение к другим LLM-агентам / человеку

Файл [`claude/agents/1c-httpExecutor.md`](claude/agents/1c-httpExecutor.md) — обычный markdown с YAML-frontmatter. Тело файла (всё после второго `---`) можно использовать как **system-prompt** для любого LLM-агента, который умеет POST'ить HTTP. Контракт описан в этом README в разделе «Контракт» — этого достаточно человеку, чтобы написать клиента самому.

## Границы (что нельзя)

1. Внутри `code` нельзя объявлять `Функция`/`Процедура` — компилятор отдаёт «Ожидается последовательность операторов». Хелперы клади в `Module.bsl`, через POST зови их.
2. Только серверный контекст: `&НаКлиенте`, формы, `ОткрытьФорму()`, UI-`Сообщить()` не работают.
3. Имена переменных не должны совпадать с boolean-операторами: `и`, `или`, `не` зарезервированы.
4. Транзакции не переходят между HTTP-вызовами — для атомарности собирай шаги в один `code` с `НачатьТранзакцию()`/`ЗафиксироватьТранзакцию()`/`ОтменитьТранзакцию()`.
5. Длинные расчёты режутся таймаутами rphost/веб-сервера. Для тяжёлых миграций пиши настоящую обработку и зови её через тот же сервис.
6. Внешние обработки (.epf) через `ВнешниеОбработки.Создать()` теоретически работают на сервере, но могут резаться безопасным режимом и доступом сервера к файлу — не тестировалось.

## Безопасность

Сервис **по дизайну** позволяет выполнять произвольный код 1С на сервере. Это **сознательный выбор** — ради того, чтобы сервис был самодостаточным универсальным каналом для отладки и тестирования. Поэтому:

- **Никогда не публикуй этот сервис в продакшен-базе** или базе с боевыми данными. Этот инструмент — для тестовых стендов и dev-окружений.
- Если веб-сервер слушает не только loopback — закрой публикацию авторизацией на уровне HTTPService (роли пользователей 1С) или реверс-прокси.
- Любой, у кого есть `ONEC_DEV_URL` и сетевой доступ к нему, может делать с базой что угодно. Это эквивалент шелла на сервере 1С.

## Лицензия

[MIT](LICENSE).
