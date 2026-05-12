# Fraud Detection Feature Store — PaySim

PaySim dataset-i üzərində Feast feature store qurulması və ML training üçün feature-ların hazırlanması.

---

## Nə edir bu notebook?

6.3 milyon tranzaksiya, cəmi 8213 fraud hadisəsi. Klassik imbalanced problem. Bu notebook həmin xamdən başlayıb ML-ə hazır, versiyonlanmış feature store-a qədər gəlir.

İki hissəyə bölünür: feature engineering (xam datadan mənalı siqnallar çıxarmaq) və feature store quraşdırma (həmin siqnalları sonra yenidən istifadə etmək üçün saxlamaq).

---

## Dataset

**PaySim** — sintetik mobil ödəniş tranzaksiya dataseti. Kaggle-dan `kagglehub` ilə birbaşa çəkilir.

```
ealaxi/paysim1
```

6.3M sətir, 11 sütun. Fraud yalnız `CASH_OUT` və `TRANSFER` tipli tranzaksiyalarda baş verir — bu əsas siqnaldır.

---

## Feature Engineering

### Müştəri səviyyəsi (customer-level)

Hər müştəri üçün bütün tranzaksiyalar üzərindən aggregation:

| Feature | Nədir |
|---|---|
| `txn_count` | Ümumi tranzaksiya sayı |
| `avg_amount` / `max_amount` | Orta və maksimum məbləğ |
| `cashout_ratio` | CASH_OUT tranzaksiyalarının payı |
| `transfer_ratio` | TRANSFER tranzaksiyalarının payı |
| `balance_drain_ratio` | Balance-ın sıfıra düşdüyü tranzaksiyaların payı |
| `fraud_rate` | Bu müştərinin keçmiş fraud nisbəti |
| `was_flagged` | Sistem tərəfindən flaglənib-flaglənmədiyi |

`balance_drain_ratio` praktikada ən güclü siqnallardan biridir — müştərinin hesabı tamamilə boşaldılırsa, bu demək olar hər zaman fraud ilə əlaqəlidir.

> ⚠️ `past_fraud_count` / `fraud_rate` — data leakage riski daşıyır. Modeli deploy etməzdən əvvəl bu feature-ların training/test split-inə necə düşdüyünü yoxla.

### Tranzaksiya səviyyəsi (transaction-level)

Hər individual tranzaksiya üçün:

- `balance_delta_orig` / `balance_delta_dest` — göndərənin və alanın balans dəyişikliyi
- `amount_to_balance_ratio` — məbləğin mövcud balansa nisbəti
- `dest_unchanged` — alıcı balansı dəyişməyibsə (şübhəli pattern)
- `is_cashout` / `is_transfer` — tip binary flagları

---

## Feature Store Arxitekturası

```
feature_repo/
├── feature_store.yaml     ← backend konfiq
└── features.py            ← entity, view, service definitionları

data/
├── agg.parquet            ← customer-level features
├── aggt.parquet           ← transaction-level features
└── registry.db            ← Feast metadata (SQLite)
```

### Feature Views

**`customer_behavior`** — müştərinin davranış profili
- txn_count, avg_amount, max_amount, cashout_ratio, transfer_ratio

**`customer_risk`** — müştərinin risk göstəriciləri
- balance_drain_ratio, fraud_rate, was_flagged

**`fraud_detection_v1`** (Feature Service) — hər iki view-dan seçilmiş feature-ları birləşdirir

### Online / Offline Store

| | Development | Production |
|---|---|---|
| Offline | Parquet (file) | — |
| Online | SQLite | Redis |
| Registry | SQLite | PostgreSQL |

---

## İşə salma sırası

```python
# 1. Parquet fayllarını yaz
agg.to_parquet("data/agg.parquet")
aggt.to_parquet("data/aggt.parquet")

# 2. Feast repo-nu apply et
subprocess.run(["feast", "apply"], cwd="feature_repo")

# 3. Store-u init et
from feast import FeatureStore
store = FeatureStore(repo_path="feature_repo/")

# 4. Historical features çək (training üçün)
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "customer_behavior:txn_count",
        "customer_behavior:cashout_ratio",
        "customer_risk:balance_drain_ratio",
        ...
    ]
).to_df()
```

> Hər dəfə `feast apply` işlətdikdən sonra `FeatureStore(...)` obyektini yenidən init et. Köhnə `store` obyekti əvvəlki registry-ni yaddaşda saxlayır.

---

## Texniki stack

- **Python** — pandas, pyarrow, scikit-learn
- **Feast** — feature store framework
- **KaggleHub** — dataset download
- **SQLite** — development-də online store və registry
- **Parquet** — offline store format

---

## Məlumat

Dataset PaySim-in simulyasiya datıdır — real bank məlumatı deyil. Production-a aparmadan əvvəl real Azerbaijani ödəniş datası üzərində feature distribution-ları yenidən yoxlanılmalıdır.
