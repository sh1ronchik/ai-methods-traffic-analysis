# 🎯 HH.ru Developer Level Classification — PoC

> Proof of Concept: автоматическое определение уровня IT-разработчика (junior / middle / senior) по резюме HeadHunter с помощью Random Forest.

---

## 📋 Содержание

- [Описание задачи](#-описание-задачи)
- [Формирование целевой переменной](#-формирование-целевой-переменной)
- [Архитектура модели](#-архитектура-модели)
- [Установка](#-установка)
- [Быстрый старт](#-быстрый-старт)
- [Визуализации](#-визуализации)
- [Результаты и выводы](#-результаты-и-выводы)
- [Технические решения](#-технические-решения)

---

## 📌 Описание задачи

PoC-эксперимент: можно ли по данным резюме HH.ru автоматически разделить разработчиков на уровни junior / middle / senior с разумным качеством?

| Параметр | Значение |
|---|---|
| Источник данных | `X.xlsx` + `y.xlsx` (выход пайплайна HW4) |
| Фильтр | Только IT-разработчики (`position_group_dev == True`) |
| Тип задачи | Многоклассовая классификация |
| Классы | `junior`, `middle`, `senior` |
| Модель | `RandomForestClassifier` |

---

## 🏷 Формирование целевой переменной

Уровень разработчика определяется по **опыту работы** (`years_experience`):

```
years_experience < 3          →  junior
3 ≤ years_experience ≤ 6      →  middle
years_experience > 6           →  senior
```

Пороги основаны на общепринятой отраслевой практике и позволяют однозначно разметить выборку без ручной разметки.

---

## 🏗 Архитектура модели

Модель — **Random Forest Classifier** с балансировкой классов.

```
X (признаки резюме)
    │
    ▼
┌──────────────────────────────────────┐
│  RandomForestClassifier               │
│  n_estimators     = 100              │
│  max_depth        = 10               │
│  class_weight     = 'balanced'       │  ← компенсация дисбаланса классов
│  min_samples_split = 5               │
│  min_samples_leaf  = 2               │
│  n_jobs           = -1               │
└──────────────────────────────────────┘
    │
    ▼
ŷ ∈ {junior, middle, senior}
```

**Разбивка данных:** `train_test_split(test_size=0.2, stratify=y)` — стратификация сохраняет пропорции классов в обоих сплитах.

**Признаки для обучения** (исключены из X):
- `years_experience` — использован для разметки таргета, утечка данных
- `level` — сам таргет
- `position_group_last_dev` — константа после фильтрации

---

## ⚙️ Установка

```bash
pip install scikit-learn pandas numpy matplotlib seaborn openpyxl
```

**Требования:**

```
python       >= 3.10
scikit-learn >= 1.3
pandas       >= 2.0
numpy        >= 1.24
matplotlib   >= 3.7
seaborn      >= 0.12
openpyxl     >= 3.1
```

---

## 🚀 Быстрый старт

```python
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

# загрузка данных
df_features = pd.read_excel('X.xlsx')
df_target   = pd.read_excel('y.xlsx')
data = pd.concat([df_features, df_target], axis=1)

# фильтрация разработчиков
data_dev = data[data['position_group_dev'] == True].copy()
data_dev['level'] = data_dev['years_experience'].apply(
    lambda y: 'junior' if y < 3 else ('middle' if y <= 6 else 'senior')
)

# train/test split
X = data_dev.drop(['years_experience', 'level', 'position_group_last_dev'], axis=1)
y = data_dev['level']

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# обучение
clf = RandomForestClassifier(
    n_estimators=100, max_depth=10,
    class_weight='balanced', random_state=42, n_jobs=-1
)
clf.fit(X_train, y_train)

from sklearn.metrics import classification_report
print(classification_report(y_test, clf.predict(X_test)))
```

---

## 📊 Визуализации

В ноутбуке строятся следующие графики:

**1. Баланс классов**
- Столбчатая диаграмма с количеством резюме по уровням
- Круговая диаграмма с процентным соотношением

**2. Матрица ошибок**
- Абсолютные значения (heatmap, Blues)
- Нормализованная по строкам (heatmap, Greens, %)

**3. Важность признаков**
- Горизонтальный bar chart топ-15 признаков
- Кривая совокупной важности с отметками 80% / 90%

---

## 📈 Результаты и выводы

### Автоматический анализ

Ноутбук автоматически диагностирует:

**Дисбаланс классов**
```
imbalance_ratio = max_class / min_class

> 3.0  →  сильный дисбаланс, рекомендуется SMOTE или undersampling
1.5–3  →  умеренный, частично компенсирован class_weight='balanced'
≤ 1.5  →  классы сбалансированы
```

**Переобучение**
```
gap = accuracy_train − accuracy_test

> 0.10  →  значительное переобучение
0.05–0.10  →  небольшое, в пределах нормы
≤ 0.05  →  переобучения нет
```

**Зависимость от малого числа признаков**
```
top-5 importance > 60%  →  модель слабо использует остальные признаки
```

### Возможные причины ошибок

- **Граничные случаи** — разработчик с 2.9 года опыта и разработчик с 3.1 года формально в разных классах, но фактически неотличимы
- **Дисбаланс классов** — middle-разработчиков традиционно больше всего в датасете
- **Качество разметки** — таргет выведен из опыта, а не из реального грейда; в реальности senior может иметь 2 года опыта в стартапе
- **Утечка признаков исключена** — `years_experience` намеренно удалён из X, модель учится на прочих сигналах

---

## 🔬 Технические решения

### Почему Random Forest?

Random Forest хорошо подходит для PoC на табличных данных:

- Не требует нормализации признаков
- Устойчив к выбросам и пропускам
- Встроенная важность признаков `feature_importances_`
- Параметр `class_weight='balanced'` автоматически компенсирует дисбаланс классов

### Стратифицированный сплит

```python
train_test_split(..., stratify=y)
```

Гарантирует, что пропорции junior/middle/senior одинаковы в train и test. Без этого в тестовой выборке может оказаться мало объектов одного класса, что исказит метрики.

### class_weight='balanced'

```
w_class = n_samples / (n_classes × n_samples_in_class)
```

Редкие классы получают более высокий вес в функции потерь — модель не игнорирует их в угоду мажоритарному классу.

### Совокупная важность признаков

```python
cumulative_importance = feature_importances_.cumsum()
```

Позволяет понять, сколько признаков несут 80% и 90% информации. Если N признаков дают 80% важности, остальные можно рассматривать как кандидатов на удаление.