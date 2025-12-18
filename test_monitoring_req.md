# Test de charge pour le monitoring:

✅ envoie **des requêtes en batch**

✅ **en parallèle**

✅ sur `/predict`, `/drift/check`, et une **alerte manuelle**

✅ pour **remplir Application Insights** et tester le monitoring réel

👉 Tu peux l’utiliser **en local OU contre Azure** sans modification lourde.

---

# 🧪 Script : `monitoring_load_test.py`

```python
import requests
import random
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

# =========================================
# CONFIGURATION
# =========================================
API_BASE_URL = "https://bank-churn.ashybay-fc2e9f26.westeurope.azurecontainerapps.io"
# Pour local :
# API_BASE_URL = "http://localhost:8000"

N_PREDICTIONS = 50
N_DRIFT_CHECKS = 5
N_MANUAL_ALERTS = 3
MAX_WORKERS = 10

# =========================================
# PAYLOADS
# =========================================
def random_customer():
    return {
        "CreditScore": random.randint(350, 850),
        "Age": random.randint(18, 80),
        "Tenure": random.randint(0, 10),
        "Balance": round(random.uniform(0, 250000), 2),
        "NumOfProducts": random.randint(1, 4),
        "HasCrCard": random.randint(0, 1),
        "IsActiveMember": random.randint(0, 1),
        "EstimatedSalary": round(random.uniform(15000, 200000), 2),
        "Geography_Germany": random.randint(0, 1),
        "Geography_Spain": random.randint(0, 1)
    }

# =========================================
# TASKS
# =========================================
def call_predict(i):
    try:
        r = requests.post(
            f"{API_BASE_URL}/predict",
            json=random_customer(),
            timeout=10
        )
        return f"[PREDICT {i}] {r.status_code}"
    except Exception as e:
        return f"[PREDICT {i}] ERROR {e}"

def call_drift(i):
    try:
        r = requests.post(
            f"{API_BASE_URL}/drift/check",
            params={"threshold": 0.05},
            timeout=30
        )
        return f"[DRIFT {i}] {r.status_code}"
    except Exception as e:
        return f"[DRIFT {i}] ERROR {e}"

def call_manual_alert(i):
    try:
        r = requests.post(
            f"{API_BASE_URL}/drift/alert",
            timeout=10
        )
        return f"[ALERT {i}] {r.status_code}"
    except Exception as e:
        return f"[ALERT {i}] ERROR {e}"

# =========================================
# MAIN
# =========================================
def run_load_test():
    tasks = []

    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        # Predictions
        for i in range(N_PREDICTIONS):
            tasks.append(executor.submit(call_predict, i))

        # Drift checks
        for i in range(N_DRIFT_CHECKS):
            tasks.append(executor.submit(call_drift, i))

        # Manual alerts
        for i in range(N_MANUAL_ALERTS):
            tasks.append(executor.submit(call_manual_alert, i))

        for future in as_completed(tasks):
            print(future.result())

    print("\n✅ Load test terminé")

if __name__ == "__main__":
    start = time.time()
    run_load_test()
    print(f"⏱️ Durée totale : {time.time() - start:.2f}s")
```

---

# ▶️ Comment l’utiliser

### 1️⃣ Installer les dépendances

```bash
pip install requests
```

### 2️⃣ Lancer le script

```bash
python monitoring_load_test.py
```

---

# 🔍 Ce que tu verras dans Azure Application Insights

Après **1–3 minutes**, lance ces requêtes 👇

---

## 📊 Toutes les requêtes API

```kusto
traces
| project timestamp, message, customDimensions
| order by timestamp desc

```

---

## 📈 Prédictions

```kusto
traces
| where customDimensions.event_type == "prediction"
| project
    timestamp,
    probability = todouble(customDimensions.probability),
    prediction = toint(customDimensions.prediction),
    risk = tostring(customDimensions.risk_level)
| order by timestamp desc

```

---

## 🚨 Drift détecté

```kusto
traces
| where customDimensions.event_type == "drift_detection"
| project timestamp,
          drift_percentage=todouble(customDimensions.drift_percentage),
          risk_level=tostring(customDimensions.risk_level)
| order by timestamp desc
```

---

## 🔔 Alertes manuelles

```kusto
traces
| where customDimensions.event_type == "manual_drift_alert"
| order by timestamp desc
```

---
