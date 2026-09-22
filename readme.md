# Автоматизация кабинета Ozon через AdsPower Local API

Автоматизация кабинета Ozon через AdsPower строится на связке локального REST API антидетекта для управления жизненным циклом профилей и протокола Chrome DevTools (CDP) через Playwright для выполнения целевых сценариев. Вся цепочка включает смену IP мобильного прокси, запуск изолированного окружения с уникальным отпечатком, управление страницей и корректное закрытие сессии.
  Есть готовый реализованный проект по автоматизации действий в кабмнетах ЯндексБизнес, но для LinkenSphere+Selenium. С синхронизацией списков сессий с внутренней ERP, получение текста ответов на отзывы через ERP/AI генерацию. Могу продемонстрировать и показать
  Также реализовывал парсинг и взаимодействие с OZON через внктренниe POST/GET запросы озон (не офциальное API с документацией для разработчиков, а клиентские обращения к API).
## Архитектура и логика работы

Архитектура решения состоит из трех основных уровней: сервисного слоя управления сетевой идентичностью, оркестратора сессий AdsPower и исполнительного модуля автоматизации на Playwright.

Взаимодействие компонентов происходит по следующему сценарию:

- **Смена сетевого адреса**: перед стартом профиля скрипт отправляет HTTP-запрос на endpoint ротации мобильного прокси (change-IP URL) и валидирует доступность соединения (при необходимости).
- **Инициализация профиля**: скрипт выполняет GET-запрос к Local API AdsPower (`/api/v1/browser/start?user_id=...`), после чего браузер запускается с заранее сконфигурированным отпечатком и прокси.
- **Подключение к процессу**: в ответе API возвращается WebSocket-ссылка (`ws.puppeteer`), через которую библиотека Playwright подключается напрямую к запущенному экземпляру браузера по протоколу CDP.
- **Работа с контекстом**: Playwright перехватывает уже открытую вкладку профиля или создает новую страницу, сохраняя куки, авторизацию и локальное хранилище сессии Ozon.
- **Выполнение целевых действий**: скрипт имитирует действия пользователя (сбор цен, остатков, выгрузка отчетов или клики по элементам интерфейса) с рандомизированными задержками и плавным перемещением курсора.
- **Завершение сессии**: после выполнения задачи Playwright закрывает контекст, а оркестратор отправляет запрос к `/api/v1/browser/stop`, высвобождая ресурсы оперативной памяти.

## Стратегия обработки ошибок

Для отказоустойчивой работы с кабинетом Ozon сценарии оборачиваются в логику повторных попыток с контролем состояния страниц и защитных механизмов.

Стратегия охватывает ключевые риски:

- **Зависание страницы**: используются явные таймауты загрузки (`timeout=30000`) с режимом ожидания `domcontentloaded` вместо `networkidle`, поскольку фоновые аналитические запросы маркетплейса могут блокировать событие полной загрузки сети. При превышении лимита выполняется перезагрузка страницы или повторная инициализация вкладки.
- **Непрогруженные элементы**: исключаются жесткие паузы `time.sleep()`, вместо них применяются динамические ожидания `locator.wait_for(state="visible")` и цепочки fallback-селекторов на случай обновления фронтенд-разметки Ozon.
- **Капчи и антифрод-экраны**: скрипт проверяет появление контейнеров проверок (Yandex SmartCaptcha или встроенных защитных экранов маркетплейса). При обнаружении триггерится отправка уведомления в Telegram-бот либо вызов внешнего сервиса распознавания капчи (antigate. rucaptcha и др).
- **Сетевые сбои прокси**: если страница выдает статус `ERR_PROXY_CONNECTION_FAILED`, сессия прерывается, вызывается смена адреса по ссылке ротации, после чего браузер перезапускается заново.
- **Гарантированная очистка**: взаимодействие с браузером всегда заключается в блок `try...finally`, чтобы при любых исключениях вызывался API-метод остановки профиля в AdsPower, исключая появление зависших процессов в операционной системе.

## Реализация на Python

Фрагмент кода ниже демонстрирует открытие профиля через Local API AdsPower, подключение Playwright через CDP и безопасное завершение работы.

```python
import time
import requests
from playwright.sync_api import sync_playwright

ADS_API_HOST = "http://127.0.0.1:50325"
PROFILE_ID = "k123456"  # Укажите реальный ID профиля AdsPower


def start_adspower_profile(profile_id: str) -> str:
    url = f"{ADS_API_HOST}/api/v1/browser/start"
    params = {"user_id": profile_id, "open_tabs": "1"}
    response = requests.get(url, params=params, timeout=15)
    payload = response.json()

    if payload.get("code") != 0:
        raise RuntimeError(f"Ошибка запуска профиля: {payload.get('msg')}")

    return payload["data"]["ws"]["puppeteer"]


def stop_adspower_profile(profile_id: str) -> None:
    url = f"{ADS_API_HOST}/api/v1/browser/stop"
    params = {"user_id": profile_id}
    try:
        requests.get(url, params=params, timeout=10)
    except requests.RequestException:
        pass


def run_ozon_task():
    ws_endpoint = start_adspower_profile(PROFILE_ID)

    with sync_playwright() as playwright:
        browser = playwright.chromium.connect_over_cdp(ws_endpoint)
        try:
            context = browser.contexts[0] if browser.contexts else browser.new_context()
            page = context.pages[0] if context.pages else context.new_page()

            # Переход в кабинет Ozon с контролем таймаута
            page.goto("https://seller.ozon.ru", timeout=45000, wait_until="domcontentloaded")

            # Ожидание селектора целевого элемента
            target_selector = "button[data-widget='submit-button']"
            target_element = page.locator(target_selector)
            
            target_element.wait_for(state="visible", timeout=10000)
            target_element.click()

            time.sleep(2)
        finally:
            browser.close()
            stop_adspower_profile(PROFILE_ID)


if __name__ == "__main__":
    run_ozon_task()
```






# Оптимизация «раздутой» таблицы: SQL + Python + Google Sheets (view-only)

Для таблиц с десятками тысяч строк, тяжёлыми формулами и дубликатами наиболее устойчивое решение — перенести хранение и обработку в SQL-базу (MySQL/PostgreSQL), а Google Sheets оставить тонким слоем отображения через Apps Script. Это исключает вычисления в интерфейсе таблицы, позволяет контролировать дубликаты на уровне БД и отдаёт сотрудникам готовые агрегированные данные.

## 1. Архитектура решения

### Слои системы

| Слой | Технология | Роль |
|------|------------|------|
| **Storage** | MySQL / PostgreSQL / MariaDB | Единое хранилище истины: сырые данные, индексы, уникальные ключи, история загрузок и защита от дубликатов. |
| **Processing** | Python (`pandas`, SQLAlchemy) | ETL-пайплайн: чтение файлов и API, нормализация, валидация, дедупликация, агрегация и расчёт бизнес-метрик. |
| **Presentation** | Google Sheets + Apps Script | Отображение готовых результатов для сотрудников. Лист получает значения, а не формулы; обновление запускается по расписанию или вручную. |

### Поток данных

1. **Загрузка**: Python читает CSV/XLSX/API, приводит типы, нормализует ключевые поля и загружает данные в staging-таблицу SQL.
2. **Защита от дублей**: БД применяет `PRIMARY KEY` или `UNIQUE INDEX` по бизнес-ключу — например, `source_id`, `article`, `order_id` или комбинации полей. Для PostgreSQL используются `INSERT ... ON CONFLICT`, для MySQL — `INSERT ... ON DUPLICATE KEY UPDATE`.
3. **Обработка**: тяжёлые операции — `JOIN`, `GROUP BY`, оконные функции, расчёт остатков, продаж, статусов и KPI — выполняются в SQL или Python, а не формулами на десятках тысяч строк в Sheets.
4. **Агрегированный слой**: итоговые данные сохраняются в таблицах/материализованных представлениях наподобие `aggregated_sales` или `report_daily`.
5. **Выгрузка**: Apps Script по расписанию выполняет ограниченный `SELECT` к готовой витрине и записывает результат в лист пакетами через `setValues()`.
6. **Отображение**: сотрудники используют Google Sheets как интерфейс просмотра, фильтрации, сортировки и ручных комментариев, но не как вычислительный движок.

### Почему Google Sheets станет быстрее

Если в Google Sheets не будет тяжёлых формул, массивных ссылок на диапазоны и постоянных пересчётов, таблица обычно будет открываться и обновляться заметно быстрее. Браузеру пользователя не нужно ждать пересчёта тысяч ячеек и зависимых формул — он получает уже рассчитанные значения из SQL-витрины.

Это также уменьшает нагрузку на ПК сотрудников: снижается потребление CPU и памяти браузером, исчезают задержки при вводе, фильтрации, прокрутке и открытии листа. Нагрузка переносится на серверный контур — базу данных и ETL-процесс, где её можно контролировать индексами, расписанием, логированием и ресурсами сервера.

Важно сохранить в Google Sheets только лёгкие операции интерфейса: фильтры, сортировку, условное форматирование в разумном объёме и несколько простых пользовательских формул при необходимости. Крупные вычисления, `IMPORTRANGE`, цепочки `VLOOKUP/XLOOKUP`, `SUMIFS` по полным колонкам, `ARRAYFORMULA` по десяткам тысяч строк и volatile-функции (`NOW`, `RAND`, `INDIRECT`, `OFFSET`) можно убрать из пользовательского листа

### Практические меры оптимизации

- **SQL как master storage**: хранить полную историю и сырые данные в БД, а не в рабочих листах.
- **Уникальные ключи**: создать `UNIQUE`-ограничения по реальному бизнес-ключу; не использовать только `article + order_date`, если за день по артикулу может быть несколько независимых операций.
- **Staging и витрины**: разделить таблицы `raw_*`, `staging_*`, `report_*`; пользователи читают только `report_*`.
- **Инкрементальная обработка**: загружать и пересчитывать только новые/изменённые записи по `updated_at`, ID источника или watermark, а не пересобирать всю историю при каждом запуске.
- **Индексы**: индексировать ключи `JOIN`, фильтры по датам, идентификаторы товаров/заказов и поля дедупликации.
- **Пакетная запись в Sheets**: использовать `setValues()` крупными блоками или Google Sheets API `batchUpdate`, а не записывать клетки или строки в цикле.
- **Ограниченная выгрузка**: в Sheets выводить только нужные сотрудникам срезы, последние периоды и агрегаты; детализацию хранить в SQL и отдавать по запросу.
- **Архивация**: старые raw-данные переносить в архивные таблицы/партиции, сохраняя в активных витринах только актуальный период.

## 2. Практический код

Ниже функция на `pandas` для локальной оптимизации входного файла. Она читает CSV/XLSX, нормализует значения, удаляет полные дубликаты и дубликаты по бизнес-ключу, агрегирует продажи по артикулам и сохраняет компактный результат в Parquet или CSV. Предпочтителен для внутреннего хранения и повторной обработки; CSV удобен для последующей выгрузки в Google Sheets.

```python
from pathlib import Path
from typing import Iterable

import pandas as pd


def optimize_sales_file(
    input_path: str,
    output_path: str,
    sku_column: str = "article",
    sales_column: str = "sales_qty",
    date_column: str | None = "order_date",
    dedupe_columns: Iterable[str] | None = None,
) -> Path:
    """Очищает файл, удаляет дубликаты и агрегирует продажи по SKU.

    Parameters
    ----------
    input_path:
        Исходный CSV/XLS/XLSX с сырыми данными.
    output_path:
        Файл результата: .parquet для компактного хранения или .csv для выгрузки.
    sku_column:
        Колонка с артикулом/SKU.
    sales_column:
        Колонка с количеством продаж.
    date_column:
        Опциональная колонка даты. Используется для расчёта последней продажи.
    dedupe_columns:
        Бизнес-ключ для удаления повторных записей, например
        ("source_order_id",) или ("article", "order_date", "operation_id").
        Если не передан, сначала удаляются только полные дубликаты строк.
    """
    source = Path(input_path)
    target = Path(output_path)

    if source.suffix.lower() in {".xlsx", ".xls"}:
        df = pd.read_excel(source)
    elif source.suffix.lower() == ".csv":
        df = pd.read_csv(source, low_memory=False)
    else:
        raise ValueError("Поддерживаются только CSV, XLS и XLSX")

    required = {sku_column, sales_column}
    missing = required.difference(df.columns)
    if missing:
        raise ValueError(f"В файле нет обязательных колонок: {', '.join(sorted(missing))}")

    # Нормализация ключевых полей.
    df[sku_column] = df[sku_column].astype("string").str.strip()
    df = df[df[sku_column].notna() & df[sku_column].ne("")].copy()
    df[sales_column] = pd.to_numeric(df[sales_column], errors="coerce").fillna(0)

    if date_column and date_column in df.columns:
        df[date_column] = pd.to_datetime(df[date_column], errors="coerce")

    # Полные дубли и, при наличии, дубли по реальному бизнес-ключу.
    df = df.drop_duplicates()
    if dedupe_columns:
        dedupe_columns = list(dedupe_columns)
        absent = set(dedupe_columns).difference(df.columns)
        if absent:
            raise ValueError(f"Колонки для дедупликации не найдены: {', '.join(sorted(absent))}")
        df = df.drop_duplicates(subset=dedupe_columns, keep="last")

    aggregations: dict[str, str] = {sales_column: "sum"}
    if date_column and date_column in df.columns:
        aggregations[date_column] = "max"

    result = (
        df.groupby(sku_column, as_index=False, dropna=False)
        .agg(aggregations)
        .rename(columns={sales_column: "total_sales_qty", date_column: "last_order_date"})
        .sort_values(sku_column, kind="stable")
    )

    target.parent.mkdir(parents=True, exist_ok=True)

    if target.suffix.lower() == ".parquet":
        result.to_parquet(target, index=False, compression="snappy")
    elif target.suffix.lower() == ".csv":
        result.to_csv(target, index=False, encoding="utf-8-sig")
    else:
        raise ValueError("Для результата используйте расширение .parquet или .csv")

    return target


if __name__ == "__main__":
    output_file = optimize_sales_file(
        input_path="raw_sales.csv",
        output_path="optimized/sales_by_article.parquet",
        sku_column="article",
        sales_column="sales_qty",
        date_column="order_date",
        dedupe_columns=("source_order_id",),
    )
    print(f"Готово: {output_file}")
```

## 3. Production-вариант: SQL + Apps Script

Для регулярной работы функцию выше используют на входе ETL-пайплайна, после чего нормализованные данные загружаются в SQL. Защита от дублей должна оставаться в БД как финальная гарантия, даже если дубли уже отсечены через `pandas`.

### SQL-схема PostgreSQL

```sql
CREATE TABLE raw_sales (
    id BIGSERIAL PRIMARY KEY,
    source_order_id TEXT NOT NULL,
    article TEXT NOT NULL,
    sales_qty NUMERIC(14, 2) NOT NULL,
    order_date DATE,
    loaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (source_order_id)
);

CREATE TABLE aggregated_sales (
    article TEXT PRIMARY KEY,
    total_sales_qty NUMERIC(14, 2) NOT NULL,
    last_order_date DATE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_raw_sales_article ON raw_sales (article);
CREATE INDEX idx_raw_sales_order_date ON raw_sales (order_date);
```

### Обновление агрегированной витрины

```sql
INSERT INTO aggregated_sales (
    article,
    total_sales_qty,
    last_order_date,
    updated_at
)
SELECT
    article,
    SUM(sales_qty) AS total_sales_qty,
    MAX(order_date) AS last_order_date,
    NOW() AS updated_at
FROM raw_sales
GROUP BY article
ON CONFLICT (article) DO UPDATE SET
    total_sales_qty = EXCLUDED.total_sales_qty,
    last_order_date = EXCLUDED.last_order_date,
    updated_at = EXCLUDED.updated_at;
```

### Apps Script: SQL → Google Sheets

Apps Script JDBC поддерживает подключения к Cloud SQL for MySQL, MySQL, SQL Server, Oracle и PostgreSQL.

```javascript
const DB_CONFIG = {
  url: 'jdbc:postgresql://HOST:5432/DATABASE?ssl=true&sslmode=require',
  user: PropertiesService.getScriptProperties().getProperty('DB_USER'),
  password: PropertiesService.getScriptProperties().getProperty('DB_PASSWORD'),
  query: `
    SELECT article, total_sales_qty, last_order_date, updated_at
    FROM aggregated_sales
    ORDER BY article
    LIMIT 10000
  `
};

function importFromSQL() {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = spreadsheet.getSheetByName('Data');
  if (!sheet) sheet = spreadsheet.insertSheet('Data');

  const conn = Jdbc.getConnection(DB_CONFIG.url, DB_CONFIG.user, DB_CONFIG.password);
  const stmt = conn.createStatement();
  const rs = stmt.executeQuery(DB_CONFIG.query);

  try {
    const meta = rs.getMetaData();
    const columnCount = meta.getColumnCount();
    const headers = [];
    const rows = [];

    for (let col = 1; col <= columnCount; col++) {
      headers.push(meta.getColumnName(col));
    }

    while (rs.next()) {
      const row = [];
      for (let col = 1; col <= columnCount; col++) {
        row.push(rs.getString(col));
      }
      rows.push(row);
    }

    // Один пакет записи вместо обращения к ячейкам внутри цикла.
    sheet.clearContents();
    sheet.getRange(1, 1, 1, columnCount).setValues([headers]);
    if (rows.length > 0) {
      sheet.getRange(2, 1, rows.length, columnCount).setValues(rows);
    }
  } finally {
    rs.close();
    stmt.close();
    conn.close();
  }
}

function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('SQL Sync')
    .addItem('Обновить данные', 'importFromSQL')
    .addToUi();
}
```


## Результат

Архитектура **SQL (источник истины и расчёты) → Python (ETL) → Apps Script/Sheets (отображение)** убирает тяжёлые формулы из клиентского слоя. В результате Google Sheets быстрее открывается и обновляется, а ПК сотрудников не тратит ресурсы на массовый пересчёт — пользователи работают с уже подготовленной витриной данных.
