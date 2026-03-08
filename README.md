# 🔗 HH.ru Data Pipeline — Chain of Responsibility

> Пайплайн предобработки резюме HeadHunter с применением паттерна проектирования **«Цепочка ответственности»**.  
> На выходе — готовые к анализу `X.npy` и `y.npy`.

---

## 📋 Содержание

- [Описание задачи](#-описание-задачи)
- [Паттерн проектирования](#-паттерн-проектирования)
- [Структура пайплайна](#-структура-пайплайна)
- [Описание обработчиков](#-описание-обработчиков)
- [Установка](#-установка)
- [Быстрый старт](#-быстрый-старт)
- [Технические решения](#-технические-решения)

---

## 📌 Описание задачи

Предобработка данных о соискателях с HeadHunter для задачи регрессии зарплат.

| Параметр | Значение |
|---|---|
| Источник данных | HH.ru (CSV) |
| Тип задачи | Предобработка → Регрессия |
| Целевая переменная | Зарплата (`ЗП`), руб. |
| Формат вывода | `X.npy` — признаки, `y.npy` — таргет |
| Интерфейс | `python app.py path/to/hh.csv` |

---

## 🧩 Паттерн проектирования

Проект реализует классический паттерн **«Цепочка ответственности»** (Chain of Responsibility).

```
Запрос (DataFrame)
      │
      ▼
┌──────────────┐
│  Handler 1   │──── если не может обработать ────►  Handler 2  ──► ...  ──► Handler N
│  _run(df)    │
└──────────────┘
      │
   результат передаётся следующему обработчику через execute()
```

**Базовый абстрактный класс `HandlerBase`:**

```python
class HandlerBase(ABC):
    def link_next(self, handler) -> HandlerBase   # соединить следующий
    def execute(self, df) -> DataFrame            # запустить цепочку
    def _run(self, df) -> DataFrame               # логика конкретного обработчика
```

Каждый обработчик:
- Реализует единственный метод `_run(df)` — никаких других зависимостей
- Не знает о существовании других обработчиков
- Возвращает `DataFrame` и передаёт его дальше через `execute()`
- Может быть переставлен, заменён или отключён без изменения остальных

---

## 🏗 Структура пайплайна

```
CSVSource          ← загрузка CSV (поддержка utf-8 / cp1251 / latin1)
    │
    ▼
TextSanitizer      ← очистка BOM, NBSP, управляющих символов
    │
    ▼
SalaryConverter    ← парсинг зарплаты + конвертация валют в рубли
    │
    ▼
SalaryOutlierHandler  ← обработка выбросов в таргете через IQR
    │
    ▼
DataCompleteness   ← дубликаты, NaN, плохие колонки
    │
    ▼
ProfileEncoder     ← feature engineering: возраст, пол, город, опыт, график...
    │
    ▼
TargetSeparator    ← выделение X / y
    │
    ▼
NumpyDatasetWriter ← сохранение X.npy и y.npy
```

---

## 📦 Описание обработчиков

### `CSVSource`
Загружает CSV, перебирая кодировки (`utf-8` → `cp1251` → `latin1`).  
Использует `engine='python'` для корректной обработки полей с переносами строк.

### `TextSanitizer`
Очищает все текстовые колонки:
- Удаляет BOM (`\ufeff`) и неразрывные пробелы (`\xa0`)
- Заменяет табы и переводы строк на пробел
- Убирает непечатаемые символы
- Сжимает последовательности пробелов

### `SalaryConverter`
Извлекает числовое значение зарплаты и конвертирует в рубли.

| Валюта | Курс к рублю |
|---|---|
| RUB / руб | 1.0 |
| USD | 77.83 |
| EUR | 90.54 |
| GBP | 104.20 |
| KZT | 0.15 |
| BYN | 26.84 |
| CNY | 11.15 |

### `SalaryOutlierHandler`
Обрабатывает выбросы в таргете методом IQR.

| Стратегия | Поведение |
|---|---|
| `clip` | Обрезает до границ `[Q1 - k·IQR, Q3 + k·IQR]` |
| `remove` | Удаляет строки с выбросами |
| `nan` | Заменяет на `NaN` |

По умолчанию: `clip` с `factor=1.5`.

### `DataCompleteness`
- Удаляет дубликаты (если строк > 100)
- Дропает колонки, где > 50% значений — NaN
- Числовые NaN → медиана колонки
- Категориальные NaN → `"__missing__"`

### `ProfileEncoder`
Главный feature engineering блок. Обрабатывает каждое поле резюме:

| Исходная колонка | Результирующие признаки |
|---|---|
| `Пол, возраст` | `gender` (0/1/-1), `age` (18–75, median impute) |
| `Ищет работу на должность:` | `role_group_*` (one-hot: dev, sys, mgr, analyst, support, marketing, engineer) |
| `Город` | `city_tier_*` (moscow / spb / large / medium / small), `trip_readiness_*` |
| `Занятость` | `is_full_time`, `is_part_time`, `is_project` |
| `График` | `sch_full_day`, `sch_flexible`, `sch_rotational`, `sch_remote` |
| `Опыт` | `years_exp` (лет + месяцы/12, clip 0–45) |
| `Авто` | `has_car` (0/1) |
| `Последняя/нынешняя должность` | `prev_role_group_*` (one-hot) |
| `Образование и ВУЗ` | `has_higher_edu` (0/1) |

### `TargetSeparator`
Разделяет `DataFrame` на `X` и `y`.  
Автоматически находит целевую колонку по списку стандартных имён: `target`, `y`, `label`, `salary`, `ЗП`, `Зарплата`.

### `NumpyDatasetWriter`
Сохраняет `X` и `y` как `.npy` рядом с исходным CSV-файлом.

---

## ⚙️ Установка

```bash
pip install numpy pandas
```

**Требования:**

```
python  >= 3.10
numpy   >= 1.24
pandas  >= 2.0
```

---

## 🚀 Быстрый старт

```bash
python app.py path/to/hh.csv
```

После запуска рядом с CSV появятся два файла:

```
path/to/X.npy   ← матрица признаков (float64)
path/to/y.npy   ← вектор зарплат в рублях (float64)
```

**Использование из кода:**

```python
from pathlib import Path
from hw4_hh_parse import run_pipeline

df = run_pipeline(Path("hh.csv"), target_col="ЗП")
```

**Или построение кастомной цепочки:**

```python
from hw4_hh_parse import (
    CSVSource, TextSanitizer, SalaryConverter,
    SalaryOutlierHandler, DataCompleteness,
    ProfileEncoder, TargetSeparator, NumpyDatasetWriter
)

loader = CSVSource("hh.csv")
loader \
    .link_next(TextSanitizer()) \
    .link_next(SalaryConverter(salary_col="ЗП")) \
    .link_next(SalaryOutlierHandler(strategy="clip")) \
    .link_next(DataCompleteness()) \
    .link_next(ProfileEncoder()) \
    .link_next(TargetSeparator("ЗП")) \
    .link_next(NumpyDatasetWriter("hh.csv"))

result = loader.execute(None)
```

---

## 🔬 Технические решения

### Почему «Цепочка ответственности»?

Каждый этап обработки данных независим и легко тестируется изолированно. Добавление нового шага (например, нормализации) требует написания одного класса и одной строки `.link_next(...)` — без изменения существующего кода. Это соответствует принципу **Open/Closed** из SOLID.

### Обработка кодировок

```
utf-8 → cp1251 → latin1
```

Последовательный перебор позволяет корректно прочитать файлы из разных источников — Windows, Linux, выгрузки HH.

### IQR для выбросов зарплат

```
lower = Q1 - 1.5 × IQR
upper = Q3 + 1.5 × IQR
```

Стратегия `clip` сохраняет все строки, не удаляя потенциально ценные данные, но ограничивает влияние экстремальных значений на модель.

### Feature Engineering города

```
Москва / МО  →  "moscow"
Санкт-Петербург  →  "spb"
Новосибирск, Екатеринбург...  →  "large_city"
Волгоград, Тюмень...  →  "medium_city"
Остальные  →  "small_city"
```

Тиеризация снижает количество dummy-переменных и учитывает региональные зарплатные различия.

### Опыт работы

```
"3 года 6 месяцев"  →  3 + 6/12 = 3.5
clip(0, 45)  →  защита от некорректных значений
median impute  →  для пропусков
```

---