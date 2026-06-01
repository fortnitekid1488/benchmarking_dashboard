# Generic Financial Benchmark Template

Локальный шаблон для обновления Excel-бенча финансовой отчетности по выбранному набору компаний. Проект готовит пакеты официальных источников, проводит пользователя через Qwen / Google AI Studio / другой LLM-сервис, принимает JSON-ответы модели и собирает новую Excel-книгу с одним листом `Свод`.

Исходный workbook не перезаписывается. Все результаты создаются как новые файлы в `outputs/`.

## Что внутри

- `financial_benchmark_template.xlsx` — одно-листовый Excel-шаблон `Свод` с сохраненными метриками, формулами, ячейками и оформлением.
- `config/company_source_registry.json` — реестр компаний, батчей и источников. В generic-версии он пустой: компании и URL добавляются под конкретный проект.
- `scripts/prepare_aistudio_sources.py` — подготовка source-пакетов для ручной LLM-экстракции.
- `scripts/apply_aistudio_json.py` — перенос проверенного JSON в копию Excel.
- `web/` и `scripts/dashboard_server.py` — локальная панель управления.
- `templates/` — JSON-схема и шаблон промпта.

## Установка на Windows

Подходит для Windows 10/11.

1. Скачайте или скопируйте проект в обычную папку, например:

```text
C:\FinancialBenchmarkTemplate
```

2. Установите Python `3.11` или `3.12` с [python.org/downloads/windows](https://www.python.org/downloads/windows/). На первом экране включите `Add python.exe to PATH`.

3. Проверьте, что в корне проекта есть workbook:

```text
financial_benchmark_template.xlsx
```

4. Дважды щелкните:

```text
install_windows_requirements.cmd
```

5. Запустите панель:

```text
start_dashboard.cmd
```

Обычно откроется `http://127.0.0.1:8787/`. Если порт занят, сервер попробует следующий свободный порт.

## Установка на macOS

1. Скачайте или скопируйте проект, например в:

```text
~/Documents/FinancialBenchmarkTemplate
```

2. Установите Python `3.11` или `3.12` с [python.org/downloads/macos](https://www.python.org/downloads/macos/), если он еще не установлен.

3. Проверьте, что в корне проекта есть:

```text
financial_benchmark_template.xlsx
```

4. Дважды щелкните:

```text
install_macos_requirements.command
```

Если macOS не разрешает открыть файл:

```bash
cd ~/Documents/FinancialBenchmarkTemplate
chmod +x *.command scripts/*.sh
xattr -dr com.apple.quarantine .
./install_macos_requirements.command
```

5. Запустите панель:

```text
start_dashboard.command
```

## Настройка под конкретный проект

Generic-шаблон поставляется без компаний и источников. Для нового проекта нужно заполнить `config/company_source_registry.json`.

Минимальный формат:

```json
{
  "batches": [
    {
      "batch_id": "batch_01",
      "title": "Main batch",
      "companies": ["company_key"]
    }
  ],
  "companies": [
    {
      "key": "company_key",
      "display_name": "Company Name",
      "country": "Country",
      "currency": "USD",
      "preferred_period": "latest reported period for selected mode",
      "sources": [
        {
          "type": "official_ir_page",
          "url": "https://example.com/investors/reports",
          "note": "Official reports page."
        }
      ]
    }
  ]
}
```

После добавления компаний их названия должны совпадать с заголовками в `financial_benchmark_template.xlsx` на листе `Свод`, иначе скрипт не найдет колонку для записи.

## Метрики

Список метрик сохранен из исходного шаблона:

- `Total Revenues`
- `Cost Of Revenues`
- `Gross Profit`
- `Gross Profit Margin %`
- `Selling General & Admin Expenses`
- `Other Operating Expenses`
- `Operating Income`
- `EBIT Margin %`
- `EBITDA`
- `EBITDA Margin %`
- `Net Income`
- `Accounts Receivable, Total`
- `Inventory`
- `Total Current Liabilities`
- `Total Assets`
- `Total Equity`
- `Total Debt`
- `Capital Expenditure`
- `Levered Free Cash Flow`
- `Cash from Operations`

## Режимы

`Квартал` ищет последний самостоятельный опубликованный квартал по каждой компании. Это не YTD, не 9M и не full year.

`Год` ищет последний полный финансовый год. Годовой режим блокирует квартальные, interim, H1, 9M, YTD и TTM ответы.

Если компания отчиталась позже остальных, provenance может содержать предупреждения о периоде. Такие значения стоит проверять вручную.

## Сценарий через дашборд

1. Запустите `start_dashboard.cmd` на Windows или `start_dashboard.command` на Mac.
2. Выберите режим: `Квартал` или `Год`.
3. Выберите провайдера. Если не знаете, оставьте `Qwen Studio`.
4. Нажмите `Подготовить источники`.
5. Дождитесь появления батчей.
6. Нажмите `Начать батч`.
7. Панель откроет папку `FILES_FOR_AI_STUDIO` и скопирует промпт в буфер обмена.
8. В LLM-сервисе отключите web search / grounding, если такой переключатель есть.
9. Загрузите все файлы из открытой папки `FILES_FOR_AI_STUDIO`.
10. Вставьте промпт из буфера обмена.
11. Скопируйте полный ответ модели и вставьте его в поле ответа в дашборде.
12. Нажмите `Сохранить и дальше`.
13. Повторите шаги для всех батчей.
14. Когда счетчик JSON станет `N/N`, нажмите `Собрать Excel`.
15. Нажмите `Показать файл` и откройте готовый workbook.

Важно: загружайте в LLM только файлы из `FILES_FOR_AI_STUDIO`. Не загружайте `source_manifest.json`, `aistudio_financial_schema.json`, `FILES_TO_UPLOAD.txt` и `prompt_for_aistudio.txt`; нужная структура уже встроена в промпт.

## CLI

После установки зависимостей можно запускать Python напрямую.

macOS/Linux:

```bash
.venv/bin/python scripts/dashboard_server.py --open
.venv/bin/python scripts/prepare_aistudio_sources.py --mode quarterly --download-mode full
.venv/bin/python scripts/prepare_aistudio_sources.py --mode annual --download-mode full
.venv/bin/python scripts/apply_aistudio_json.py --mode quarterly
.venv/bin/python scripts/apply_aistudio_json.py --mode annual
```

Windows:

```bat
.venv\Scripts\python.exe scripts\dashboard_server.py --open
.venv\Scripts\python.exe scripts\prepare_aistudio_sources.py --mode quarterly --download-mode full
.venv\Scripts\python.exe scripts\prepare_aistudio_sources.py --mode annual --download-mode full
.venv\Scripts\python.exe scripts\apply_aistudio_json.py --mode quarterly
.venv\Scripts\python.exe scripts\apply_aistudio_json.py --mode annual
```

Если провайдер не справляется с большими батчами:

```bash
.venv/bin/python scripts/prepare_aistudio_sources.py --mode quarterly --download-mode full --max-companies-per-batch 1
```

## Результаты

Итоговые файлы появляются в `outputs/`.

Для квартального режима:

```text
outputs/aistudio_quarterly_excel_update_*/financial_benchmark_template_aistudio_quarterly_update_*.xlsx
outputs/aistudio_quarterly_excel_update_*/*_provenance.json
```

Для годового режима:

```text
outputs/aistudio_annual_excel_update_*/financial_benchmark_template_aistudio_annual_update_*.xlsx
outputs/aistudio_annual_excel_update_*/*_provenance.json
```

Excel-файл содержит один лист `Свод` в формате исходного шаблона. Обновленные значения получают комментарии с источниками, рядом сохраняется machine-readable provenance JSON.

## Что коммитить

Коммитить можно:

- код в `scripts/`, `web/`, `templates/`, `config/`;
- `README.md`, `AGENTS.md`, `LICENSE`, `requirements.txt`;
- исходный workbook `financial_benchmark_template.xlsx`.

Не коммитьте без отдельной проверки:

- `.env`, API-ключи, токены, пароли;
- `outputs/` и все сгенерированные Excel/JSON/PDF/PNG;
- другие `.xlsx`/`.xlsm`/`.xls` файлы, если это не осознанное обновление исходного шаблона;
- локальные кэши `.venv/`, `.playwright-*`, `__pycache__/`;
- внутренние рабочие материалы `.Codex/`.

Код распространяется по MIT License. Скачанные отчеты компаний и сторонние документы не становятся частью лицензии репозитория; для них действуют права соответствующих владельцев и источников.

## Частые проблемы

### Python не найден

Переустановите Python с [python.org](https://www.python.org/downloads/) и включите добавление в `PATH` на Windows.

### Панель не открылась

Посмотрите окно Terminal или Command Prompt. Там должен быть адрес вида:

```text
http://127.0.0.1:8787/
```

Скопируйте его в браузер.

### Нет батчей

В generic-шаблоне реестр компаний пустой. Заполните `config/company_source_registry.json`, затем снова нажмите `Подготовить источники`.

### Кнопка `Собрать Excel` неактивна

Не все батчи сохранены. Счетчик JSON должен стать `N/N`.

### Итоговый Excel не появился

Проверьте:

1. исходный workbook лежит в корне проекта;
2. JSON сохранен для всех батчей;
3. папка `outputs/` доступна для записи;
4. Excel-файл не открыт в другом приложении во время сборки.

## Для разработчиков

Проверить Python-синтаксис:

```bash
python3 -m compileall -q scripts
```

Запустить локальную панель без автозапуска браузера:

```bash
.venv/bin/python scripts/dashboard_server.py
```

Открыть статус API:

```text
http://127.0.0.1:8787/api/status
```
