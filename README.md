# Purchase Propensity Modeling

Прогнозирование вероятности покупки пользователя интернет-магазина в течение 7 дней
после его первой сессии — на основе поведения только в этой первой сессии.

$$
P(\text{purchase within 7 days} \mid \text{first session behavior})
$$

## Бизнес-задача

Не все пользователи одинаково склонны к покупке: одни активно просматривают и добавляют
товары в корзину, другие ограничиваются парой просмотров. Модель ранжирует пользователей
по вероятности покупки сразу после окончания первой сессии — это можно использовать для:

- приоритизации маркетинговых активностей;
- персонализации коммуникаций;
- отбора аудитории для дальнейших A/B-экспериментов.

## Датасет

[eCommerce events history in electronics store](https://www.kaggle.com/datasets/mkechinov/ecommerce-events-history-in-electronics-store)
(Kaggle) — события `view` / `cart` / `purchase` пользователей интернет-магазина электроники.

Датасет не включён в репозиторий из-за размера. Чтобы запустить ноутбук локально:

1. Скачайте `events.csv` (или архив с ним) с Kaggle.
2. Положите файл в `data/` в корне проекта, либо укажите свой путь через переменную
   окружения `DATA_DIR`.

## Pipeline

```text
Raw Events
    ↓
Data Preparation
    ↓
Session Construction
    ↓
First Session Identification
    ↓
Feature Engineering
    ↓
Future Purchase Target
    ↓
Leakage Check
    ↓
Time-Based Train/Test Split
    ↓
Baseline Model (Logistic Regression)
    ↓
Random Forest
    ↓
Model Evaluation
    ↓
Top-K Business Analysis
```

Ключевые методологические решения:

- **Prediction point** — момент окончания первой сессии; все признаки строятся только
  на данных, доступных до этого момента (явная проверка на data leakage).
- **Временное разделение train/test** вместо случайного — модель оценивается так, как
  она бы работала в проде: обучение на прошлом, проверка на будущем.
- **Class-weighted модели** — целевая переменная сильно несбалансирована (~1.4% положительного класса).

## Результаты

Сравнение моделей на отложенной по времени тестовой выборке (75 185 пользователей):

| Метрика | Logistic Regression | Random Forest |
|---|---|---|
| ROC AUC | 0.652 | **0.678** |
| PR AUC | **0.040** | 0.033 |
| Precision (порог 0.5) | 0.061 | 0.033 |
| Recall (порог 0.5) | 0.248 | **0.404** |
| F1 (порог 0.5) | **0.098** | 0.061 |

**Top-K анализ** (бизнес-метрика): отбор топ-10% пользователей по предсказанной
вероятности даёт конверсию **4.0%** против **1.4%** в среднем по выборке — **lift ×2.89**.

| Top-K | Пользователей | Покупок | Conversion Rate | Lift |
|---|---|---|---|---|
| 10% | 7 518 | 300 | 4.0% | ×2.89 |
| 20% | 15 037 | 457 | 3.0% | ×2.20 |
| 30% | 22 555 | 587 | 2.6% | ×1.88 |
| 50% | 37 592 | 777 | 2.1% | ×1.50 |
| 100% | 75 185 | 1 039 | 1.4% | ×1.00 |

Полный разбор результатов и графики — в ноутбуке.

## Ограничения

- Порог классификации 0.5 не откалиброван; для практического использования модель
  лучше применять как ранкер (см. Top-K) или подбирать порог под целевой Precision/Recall.
- Оценка проведена на единственном временном разбиении, без rolling-кросс-валидации.
- Top-K lift — predictive/ranking lift, а не causal uplift; для оценки эффекта
  маркетингового воздействия нужен A/B-тест.

## Структура репозитория

```text
.
├── README.md
├── requirements.txt
├── notebooks/
│   └── pet-project.ipynb      # полный pipeline: EDA → фичи → модели → бизнес-анализ
├── data/                      # датасет (не в git, см. .gitignore)
└── models/                    # сохранённая модель и список признаков (не в git)
```

## Как запустить

```bash
git clone https://github.com/<your-username>/purchase-propensity-modeling.git
cd purchase-propensity-modeling

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install -r requirements.txt

# скачать events.csv с Kaggle и положить в data/
jupyter notebook notebooks/pet-project.ipynb
```

## Возможные направления развития

- Подбор гиперпараметров (GridSearch/Optuna) и сравнение с градиентным бустингом
  (XGBoost/LightGBM/CatBoost).
- Калибровка вероятностей (Platt scaling / isotonic regression).
- Кросс-валидация по времени (rolling window).
- Дополнительные признаки: топ-категория/бренд первой сессии, час/день недели.
- A/B-тест на пользователях из топ-K сегмента — переход от ranking lift к causal uplift.

## Лицензия

[MIT](LICENSE)
