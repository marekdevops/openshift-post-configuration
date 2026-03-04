# OCP Grafana Monitoring Dashboard

Instaluje Grafanę na klastrze OpenShift przez Operator (OLM) i wgrywa
gotowy dashboard do monitorowania stanu klastra — operatory, węzły, etcd
i API Server. Dane pochodzą z klastrowego Thanos Querier (bez instalowania
dodatkowego Prometheusa).

---

## Co dostaniesz po uruchomieniu

```
Klaster OCP
├── Namespace: grafana-monitoring
│   ├── Grafana Operator (OLM, community, channel v5)
│   ├── Grafana Instance  ocp-grafana  ← pod + service + route (HTTPS)
│   ├── GrafanaDatasource → Thanos Querier OCP  (token z openshift-monitoring)
│   └── GrafanaDashboard  ocp-cluster-overview
│
└── Namespace: openshift-monitoring  (istniejący)
    ├── ServiceAccount: grafana-monitoring-reader
    ├── Secret:         grafana-monitoring-reader-token  (nie wygasa)
    └── ClusterRoleBinding: grafana-monitoring-view      (read-only)
```

---

## Dashboard — zawartość

### Row: Cluster Health  *(stat panels)*
| Panel | Metryka |
|---|---|
| Wersja OCP | `cluster_version{type="completed"}` |
| Węzły klastra | `count(kube_node_info)` |
| Pody Running | `count(kube_pod_status_phase{phase="Running"})` |
| Pody z problemem | `count(kube_pod_status_phase{phase!~"Running|Succeeded"})` |
| Operatory Degraded | `count(cluster_operator_conditions{condition="Degraded",...})` |
| Operatory Progressing | `count(cluster_operator_conditions{condition="Progressing",...})` |

### Row: Utilizacja Klastra  *(timeseries)*
- CPU Usage per Node (%)
- Memory Usage per Node (%)

### Row: Cluster Operators  *(table)*
- Tabela wszystkich operatorów z kolumnami: **Available / Degraded / Progressing**
- Kolorowanie: zielony=OK, czerwony=problem, żółty=aktualizacja

### Row: etcd Health
- etcd Leader (stat — zielony/czerwony)
- etcd Members (stat)
- WAL fsync latency P99 (timeseries)
- DB Size (timeseries)

### Row: API Server
- Request Rate by verb (timeseries)
- Error Rate 5xx (timeseries)
- Latency P99 by verb (timeseries)

---

## Wymagania

### Uprawnienia w OCP
Zalogowany użytkownik musi mieć `cluster-admin` (potrzebne do:
tworzenia ClusterRoleBinding, instalacji Operatora przez OLM).

### Na maszynie roboczej

```bash
pip install kubernetes
ansible-galaxy collection install kubernetes.core
```

### Weryfikacja logowania do klastra

```bash
oc whoami
oc cluster-info
```

---

## Instalacja krok po kroku

### 1. Sklonuj / przejdź do katalogu

```bash
cd openshift-post-configuration/grafana-monitoring
```

### 2. (Opcjonalnie) Zmień hasło admina

Domyślne hasło to `OCP-Grafana-2025!`. Możesz je zmienić przed uruchomieniem:

```bash
ansible-playbook playbook.yml -e grafana_admin_password="TwojeHaslo123!"
```

### 3. Uruchom playbook

```bash
# Domyślnie (validate_certs=false — dla self-signed certów OCP)
ansible-playbook playbook.yml

# Jeśli klaster ma certyfikat od zaufanego CA
ansible-playbook playbook.yml -e validate_certs=true

# Verbose (debug)
ansible-playbook playbook.yml -vvv
```

Instalacja trwa **3–6 minut** (głównie oczekiwanie na pobranie obrazu Grafany).

### 4. Sprawdź output

Na końcu zobaczysz:
```
"URL     : https://ocp-grafana-grafana-monitoring.apps.cluster.example.com"
"Login   : admin"
"Haslo   : OCP-Grafana-2025!"
"Dashboard: https://ocp-grafana-....apps.../d/ocp-cluster-overview"
```

Credentials są też zapisane w pliku `grafana-ocp-credentials.env` (chmod 600).

### 5. Zaloguj się i zmień hasło

1. Otwórz URL Grafany w przeglądarce
2. Zaloguj jako `admin` / hasło z outputu
3. **Zmień hasło**: User menu → Change password

---

## Troubleshooting

### Operator utknął w "InstallPlan" — nie osiąga Succeeded

```bash
# Sprawdz stan CSV
oc get csv -n grafana-monitoring

# Sprawdz SubscriptionStatus
oc describe subscription grafana-operator -n grafana-monitoring

# Czy community-operators catalog dziala?
oc get catalogsource -n openshift-marketplace
```

### Grafana pod nie startuje

```bash
oc get pods -n grafana-monitoring
oc describe pod <grafana-pod> -n grafana-monitoring
oc logs <grafana-pod> -n grafana-monitoring
```

### Dashboard nie widzi metryk (No Data)

```bash
# Sprawdz datasource — token i URL
oc get grafanadatasource -n grafana-monitoring -o yaml

# Test reczny
TOKEN=$(oc get secret grafana-monitoring-reader-token \
  -n openshift-monitoring -o jsonpath='{.data.token}' | base64 -d)
THANOS=$(oc get route thanos-querier -n openshift-monitoring \
  -o jsonpath='{.spec.host}')

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS/api/v1/query?query=cluster_operator_conditions" | jq .status
```

### Route nie zostala utworzona

```bash
oc get route -n grafana-monitoring

# Jesli brak — sprawdz czy Grafana CR ma spec.route
oc describe grafana ocp-grafana -n grafana-monitoring
```

---

## Sprzątanie (rollback)

```bash
# Usun wszystko z namespace grafana-monitoring
oc delete namespace grafana-monitoring

# Usun ClusterRoleBinding (jest cluster-scoped)
oc delete clusterrolebinding grafana-monitoring-view

# Usun SA i Secret z openshift-monitoring
oc delete serviceaccount grafana-monitoring-reader -n openshift-monitoring
oc delete secret grafana-monitoring-reader-token -n openshift-monitoring
```

---

## Rozszerzenie dashboardu

Plik `files/ocp-cluster-overview.json` to standardowy Grafana JSON.
Możesz go edytować w Grafana UI (Edit dashboard) i eksportować z powrotem:

1. W Grafanie: Dashboard → Share → Export → Save to file
2. Podmień plik `files/ocp-cluster-overview.json`
3. Uruchom ponownie playbook — zaktualizuje dashboard w klastrze

Aby dodać kolejny dashboard, stwórz nowy JSON w `files/` i dodaj
kolejne zadanie `GrafanaDashboard` w `playbook.yml`.
