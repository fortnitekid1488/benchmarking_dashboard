# Кастомизация чистого финансового дашборда

Эта инструкция нужна для generic-версии, где `config/company_source_registry.json` пустой и ни одна компания еще не внесена. Базовый принцип: Excel-шаблон задает вид таблицы, реестр задает компании и источники, Python-скрипты готовят источники и переносят проверенный JSON в копию workbook.

## Откуда брать отчетность

Ищите источники в таком порядке:

1. Официальные структурированные источники: регуляторные API, XBRL/iXBRL, биржевые раскрытия, SEC EDGAR для США, OpenDART для Кореи, CNINFO или биржевые раскрытия для Китая.
2. Официальные investor relations страницы компании: `Reports`, `Results`, `Financial statements`, `Presentations`, `Quarterly results`, `Annual reports`.
3. Официальные PDF, XLSX и HTML-таблицы с отчетностью. XLSX особенно удобны: скрипт конвертирует их в upload-friendly `.txt`, а оригинальный файл остается в пакете для provenance.
4. `StockAnalysis` как читаемый агрегатор-фолбэк, если официальная страница неудобна или нужна быстрая сверка. Для годовых данных обычно подходит страница вида `https://stockanalysis.com/stocks/TICKER/financials/`, для квартальных - та же страница с `?p=quarterly`. Для баланса и cash flow используйте соседние страницы `balance-sheet/` и `cash-flow-statement/`. В реестре помечайте такой источник как `readable_aggregator_fallback`.
5. Платные или low-cost financial data API только если проект осознанно принимает зависимость от цены, ключей и лимитов.

Не стройте основной workflow на старых ссылках из Excel, Investing, Yahoo Finance или MarketScreener без проверки: такие страницы часто ломаются из-за cookies, Cloudflare, paywall или нестабильной разметки. Их можно использовать как подсказку для названий метрик или ручную сверку, но не как основной источник истины.

## Что менять для нового набора компаний

### 1. Excel-шаблон

Файл: `financial_benchmark_template.xlsx`.

На листе `Свод` укажите компании в видимых колонках таблицы. Сейчас импортер ищет названия компаний в строках `1` и `31`, в колонках `D:R`. Если в JSON придет компания, которой нет в этих заголовках, скрипт пропустит ее и запишет причину в provenance.

Если нужно больше или меньше колонок, проверьте код в `scripts/apply_aistudio_json.py`:

- `OUTPUT_MAX_ROW` и `OUTPUT_MAX_COL` задают копируемую область итогового листа.
- `build_company_column_map()` сейчас сканирует `range(4, 19)`, то есть `D:R`.
- `TOP_METRIC_ROWS` связывает названия метрик из JSON со строками на листе `Свод`.
- `recalc_company_column()` пересчитывает производные строки и USD-блок.

### 2. Реестр компаний и источников

Файл: `config/company_source_registry.json`.

Минимальный пример:

```json
{
  "batches": [
    {
      "batch_id": "batch_01_main",
      "title": "Main batch",
      "companies": ["company_a", "company_b"]
    }
  ],
  "companies": [
    {
      "key": "company_a",
      "display_name": "Company A",
      "country": "United States",
      "currency": "USD",
      "preferred_period": "latest reported period for selected mode",
      "sources": [
        {
          "type": "official_ir_page",
          "url": "https://example.com/investors/results",
          "note": "Official results and reports page."
        },
        {
          "type": "readable_aggregator_fallback",
          "url": "https://stockanalysis.com/stocks/EXAMPLE/financials/?p=quarterly",
          "note": "Readable fallback for quarterly financial tables."
        }
      ]
    }
  ]
}
```

Правила:

- `key` - короткий стабильный ASCII-идентификатор. Он используется в батчах, именах файлов и JSON.
- `display_name` должен совпадать с заголовком компании в Excel или иметь алиас в `COMPANY_ALIASES`.
- `currency` - валюта отчетности компании. Модель должна отдавать значения в миллионах этой валюты, без конвертации.
- `batches[].companies` содержит только `key` из блока `companies`.
- В один батч обычно кладите до 6 компаний. Скрипт дополнительно считает примерный размер промпта и файлов и авто-сплитит слишком тяжелые батчи.

Типы источников, которые уже понимает промпт:

- `official_api`
- `official_pdf`
- `official_excel`
- `official_ir_page`
- `official_regulatory_portal`
- `readable_aggregator_fallback`

Дополнительные типы можно использовать, если они понятны человеку и модели, но для строгого workflow лучше держаться этого словаря или расширять схему осознанно.

### 3. Метрики

Файлы:

- `scripts/prepare_aistudio_sources.py`
- `scripts/apply_aistudio_json.py`
- `templates/aistudio_financial_schema.json`

Если меняете список метрик, обновите его в обоих Python-файлах. Если меняете строки в Excel, обновите `TOP_METRIC_ROWS` в `scripts/apply_aistudio_json.py`. Если добавляете новые поля в JSON, проверьте схему и функцию `provenance_comment()`.

### 4. Промпт и правила периода

Файл: `templates/aistudio_prompt_template.txt`.

Обычно его не нужно менять для нового набора компаний. Он уже просит:

- брать последний самостоятельный квартал в режиме `Квартал`;
- брать последний полный финансовый год в режиме `Год`;
- не смешивать периоды внутри одной компании;
- отдавать источник, страницу/таблицу и короткое evidence для каждого значения;
- помечать непроверенные значения как `review_required`.

Если отрасль требует специальных метрик или терминов, добавляйте это в промпт, но не убирайте требования к JSON, периоду и source evidence.

### 5. Алиасы компаний

Файл: `scripts/apply_aistudio_json.py`.

Если в Excel компания написана иначе, чем в JSON, добавьте алиас в `COMPANY_ALIASES`:

```python
COMPANY_ALIASES = {
    "company_a": "company a plc",
}
```

Ключ слева - нормализованный `company_key` из JSON, значение справа - нормализованный заголовок из Excel.

### 6. Вид и поведение дашборда

Для простого добавления компаний UI менять не нужно.

Если нужно менять саму панель:

- `web/dashboard.html` - структура экрана и кнопки;
- `web/dashboard.css` - стили;
- `web/dashboard.js` - логика фронтенда, список LLM-провайдеров и тексты статусов;
- `scripts/dashboard_server.py` - локальный API, запуск подготовки источников, сохранение JSON и сборка Excel.

Дашборд должен оставаться тонким интерфейсом над Python-скриптами. Не переносите парсинг источников или запись Excel во frontend.

### 7. AGENTS.md

`AGENTS.md` не нужен пользователю для запуска дашборда. Это инструкция для Codex и других coding agents: какие файлы важны, какие команды запускать, что нельзя коммитить и как считать задачу готовой.

Обновляйте `AGENTS.md`, когда меняются:

- структура репозитория;
- команды установки, запуска, тестов или сборки;
- правила безопасности и Git hygiene;
- договоренности о workbook, источниках, provenance или dashboard workflow.

Не используйте `AGENTS.md` как пользовательский README или продуктовую документацию. Для людей пишите `README.md` и файлы в `docs/`.

## Проверка после кастомизации

1. Проверьте JSON-реестр:

```bash
python3 -m json.tool config/company_source_registry.json >/tmp/company_source_registry_checked.json
```

2. Установите зависимости, если еще не установлены:

```bash
./install_macos_requirements.command
```

На Windows используйте `install_windows_requirements.cmd`.

3. Подготовьте источники:

```bash
.venv/bin/python scripts/prepare_aistudio_sources.py --mode quarterly --download-mode full
.venv/bin/python scripts/prepare_aistudio_sources.py --mode annual --download-mode full
```

4. Откройте последний пакет в `outputs/` и проверьте для каждого батча:

- есть `prompt_for_aistudio.txt`;
- есть `source_manifest.json`;
- есть папка `FILES_FOR_AI_STUDIO`;
- в `FILES_FOR_AI_STUDIO` лежат только файлы, которые нужно загрузить в LLM;
- у источников нет массовых `failed` без понятной причины.

5. Запустите дашборд:

```bash
.venv/bin/python scripts/dashboard_server.py --open
```

6. Проведите один тестовый батч через выбранный LLM и сохраните JSON в дашборде.

7. После сохранения всех батчей соберите Excel. Скрипт создаст копию в `outputs/` и не перезапишет `financial_benchmark_template.xlsx`.

8. Откройте sidecar `*_provenance.json` и проверьте:

- нет неожиданных `skipped_companies`;
- значения без источника не попали в workbook;
- период соответствует режиму `Квартал` или `Год`;
- `source_url`, `source_file`, `table_label` и `evidence_quote` заполнены для внесенных значений.

## Короткий чеклист

- Excel `Свод` содержит нужные компании в строках `1` и/или `31`.
- `config/company_source_registry.json` содержит те же компании через `key` и `display_name`.
- Для каждой компании есть минимум один официальный источник.
- `StockAnalysis` добавлен только как `readable_aggregator_fallback`, а не как главный источник.
- Метрики в Python совпадают с метриками в Excel.
- Промпт требует source evidence и строгий JSON.
- Итоговый workbook создается в `outputs/`, исходный шаблон не меняется.
