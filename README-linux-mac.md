# Monitoring Setup (Linux / macOS)

מדריך קצר להקמה של Loki + Prometheus + Grafana ב-`monitor-dev`.

## מה יש בפרויקט

- Helm chart מקומי: `charts/monitor`
- תלויות ב-`Chart.yaml`:
  - `grafana/loki` (כ-subchart)
- קבצי values:
  - `charts/monitor/values-dev.yaml` (ערכי הסביבה להתקנת הצ'ארט המקומי)
  - `charts/monitor/values-loki-dev.yaml` (ערכי Loki להתקנה ישירה של chart חיצוני)

## דרישות מוקדמות

- Kubernetes cluster פעיל
- `kubectl` מחובר לקלאסטר הנכון
- Helm 3 מותקן

בדיקה מהירה:

```bash
kubectl config current-context
helm version
```

## 1) הוספת Helm repos

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## 2) יצירת Namespace

```bash
kubectl create namespace monitor-dev --dry-run=client -o yaml | kubectl apply -f -
```
## 3) התקנת Loki דרך הצ'ארט המקומי (מומלץ)

### 3.1 עדכון dependencies

```bash
helm dependency update ./charts/monitor
```

### 3.2 מילוי `values-dev.yaml` בפורמט של subchart

שים לב: כש-Loki מוגדר כ-dependency, הערכים שלו צריכים להיות תחת המפתח `loki:`.

דוגמה:

```yaml
loki:
  deploymentMode: SingleBinary
  loki:
    auth_enabled: false
    useTestSchema: true
    storage:
      type: filesystem
    commonConfig:
      replication_factor: 1
  singleBinary:
    replicas: 1
  read:
    replicas: 0
  write:
    replicas: 0
  backend:
    replicas: 0
```

### 3.3 התקנה

```bash
helm upgrade --install monitor ./charts/monitor \
  -n monitor-dev \
  -f ./charts/monitor/values-dev.yaml
```
## 4) התקנת Prometheus + Grafana

הדרך הפשוטה: `kube-prometheus-stack` (כולל Prometheus, Grafana, Alertmanager).

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitor-dev
```

## 5) אימות התקנה

```bash
kubectl get pods -n monitor-dev
kubectl get svc -n monitor-dev
helm list -n monitor-dev
```

## 6) גישה ל-Grafana

```bash
kubectl port-forward -n monitor-dev svc/kube-prometheus-stack-grafana 3000:80
```

אחר כך לפתוח בדפדפן: `http://localhost:3000`

סיסמה ראשונית (אם לא שינית values):

```bash
kubectl get secret -n monitor-dev kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

Note for macOS: if `base64 --decode` does not work, use:

```bash
kubectl get secret -n monitor-dev kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -D
```
## 7) חיבור Grafana ל-Loki (Datasource)

1. ב-Grafana: `Connections` -> `Data sources` -> `Add data source` -> `Loki`
2. כתובת Loki מתוך הקלאסטר:

```bash
kubectl get svc -n monitor-dev | grep loki
```

בדרך כלל ה-URL יהיה בסגנון:
- `http://monitor-loki.monitor-dev.svc.cluster.local:3100`
- או שם שירות Loki אחר לפי פלט `kubectl get svc`.

## התקנה ישירה של Loki (אלטרנטיבה)

אם אתה לא רוצה להשתמש ב-dependency של הצ'ארט המקומי, אפשר להתקין ישירות:

```bash
helm upgrade --install loki grafana/loki \
  -n monitor-dev \
  -f ./charts/monitor/values-loki-dev.yaml
```

במצב הזה `values-loki-dev.yaml` מתאים כמו שהוא, כי הוא מועבר ישירות ל-chart של Loki.
