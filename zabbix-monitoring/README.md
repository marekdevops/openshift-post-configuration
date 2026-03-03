# OCP → Zabbix Monitoring Integration

Playbook konfiguruje dostep do klastrowego Prometheusa (przez Thanos Querier)
dla zewnetrznego Zabbixa. Na koncu wypluwa gotowy token i URL do wklejenia w Zabbixa.

---

## Co robi playbook

```
[1] Tworzy ServiceAccount       zabbix-monitoring-reader  (openshift-monitoring)
[2] Nadaje ClusterRoleBinding   cluster-monitoring-view   (tylko odczyt metryk)
[3] Tworzy Secret SA            kubernetes.io/service-account-token  (NIE wygasa)
[4] Czeka az OCP wypelni token
[5] Pobiera Route Thanos Querier
[6] Robi testowe query          cluster_operator_conditions{condition="Available"}
[7] Zapisuje credentials do     zabbix-ocp-credentials.env
```

---

## Wymagania

### Na maszynie z ktorej uruchamiasz playbook

```bash
# Python kubernetes client
pip install kubernetes

# Ansible collections
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install community.okd   # opcjonalnie, dla Route
```

### Uprawnienia w OCP

Zalogowany uzytkownik musi miec mozliwosc tworzenia `ClusterRoleBinding`.
Wystarczy rola `cluster-admin` lub customowa rola z uprawnieniami:

```
rbac.authorization.k8s.io/clusterrolebindings  create,get,list
core/serviceaccounts                            create,get
core/secrets                                    create,get,watch
```

### Kubeconfig

```bash
# Sprawdz ze jestes zalogowany do wlasciwego klastra
oc whoami
oc cluster-info
```

---

## Uruchomienie

```bash
cd openshift-tools/zabbix-monitoring

# Domyslnie (validate_certs=false - przydatne dla self-signed)
ansible-playbook playbook.yml

# Jesli klaster ma zaufany certyfikat (publiczny CA)
ansible-playbook playbook.yml -e validate_certs=true

# Verbose - jesli cos nie dziala
ansible-playbook playbook.yml -vvv
```

Po zakonczeniu zobaczysz output podobny do:

```
"======================================================"
"  KONFIGURACJA ZAKONCZONA POMYSLNIE"
"======================================================"
"Thanos Querier URL : https://thanos-querier-openshift-monitoring.apps.cluster.example.com"
"Credentials zapisano: /path/to/zabbix-ocp-credentials.env"
"Test query HTTP    : 200"
"Operatorow w odpowiedzi: 32"
"------------------------------------------------------"
"Operatory (Available=1):"
"authentication, baremetal, cloud-controller-manager, ..."
"======================================================"
```

Credentials sa w pliku `zabbix-ocp-credentials.env` (chmod 600).

---

## Co wkleic w Zabbixa

Po uruchomieniu playbooka otwórz plik `zabbix-ocp-credentials.env` i weź:
- `THANOS_URL` - adres endpointu
- `BEARER_TOKEN` - token do naglowka

### Konfiguracja Macro na hoscie w Zabbix

| Macro | Wartosc |
|---|---|
| `{$OCP_THANOS_URL}` | wartosc `THANOS_URL` z pliku env |
| `{$OCP_TOKEN}` | wartosc `BEARER_TOKEN` z pliku env (type: Secret text) |

### Item - przyklad (HTTP Agent)

| Pole | Wartosc |
|---|---|
| Name | OCP Cluster Operator: {#NAME} Available |
| Type | HTTP agent |
| URL | `{$OCP_THANOS_URL}/api/v1/query` |
| Query fields | `query` = `cluster_operator_conditions{condition="Available",name="{#NAME}"}` |
| Request method | GET |
| Headers | `Authorization` = `Bearer {$OCP_TOKEN}` |
| Preprocessing | JSONPath: `$.data.result[0].value[1]` |
| Value mapping | 1 → Available, 0 → Degraded |

### Przykladowe query dla Zabbix

```
# Stan operatora (1=ok, 0=degraded)
cluster_operator_conditions{condition="Available",name="authentication"}
cluster_operator_conditions{condition="Available",name="ingress"}
cluster_operator_conditions{condition="Available",name="dns"}
cluster_operator_conditions{condition="Available",name="etcd"}

# Wszystkie operatory naraz (dla LLD - Low Level Discovery)
cluster_operator_conditions{condition="Available"}

# Degraded operatory (powinny byc 0)
cluster_operator_conditions{condition="Degraded"}

# Progressing (aktualizacja w toku)
cluster_operator_conditions{condition="Progressing"}
```

### Test z curl (weryfikacja przed konfiguracja Zabbix)

```bash
source zabbix-ocp-credentials.env

curl -sk \
  -H "Authorization: Bearer $BEARER_TOKEN" \
  "$THANOS_URL/api/v1/query?query=cluster_operator_conditions{condition=\"Available\"}" \
  | python3 -m json.tool | grep -E '"name"|"value"'
```

---

## Architektura

```
Zabbix HTTP Agent
      |
      |  HTTPS GET + Bearer Token
      v
Route: thanos-querier-openshift-monitoring.apps.<cluster>
      |
      v
Thanos Querier (openshift-monitoring)
      |
   [deduplikacja]
      |
   +--+--+
   |     |
   v     v
prometheus-k8s-0   prometheus-k8s-1
(cluster metrics)  (cluster metrics)
```

Thanos Querier deduplikuje dane z obu replik Prometheusa - dlatego pytamy
jego endpoint, nie bezposrednio Prometheusa.

---

## Co to jest ten token i dlaczego nie wygasa

Playbook tworzy Secret typu `kubernetes.io/service-account-token`.
To jest "legacy" metoda z Kubernetes < 1.24, ktora OCP nadal wspiera.
Token tego typu:
- **nie ma daty waznosci** (w przeciwienstwie do `oc create token`)
- jest przechowywany w Secretcie w namespace `openshift-monitoring`
- mozna go w kazdej chwili uniewa_znic usuwajac Secret

Aby sprawdzic token po fakcie:

```bash
oc get secret zabbix-monitoring-reader-token \
  -n openshift-monitoring \
  -o jsonpath='{.data.token}' | base64 -d
```

Aby go uniewa_znic (bez usuwania SA):

```bash
oc delete secret zabbix-monitoring-reader-token -n openshift-monitoring
```

---

## Sprzatanie (jesli chcesz cofnac zmiany)

```bash
oc delete clusterrolebinding zabbix-monitoring-view
oc delete secret zabbix-monitoring-reader-token -n openshift-monitoring
oc delete serviceaccount zabbix-monitoring-reader -n openshift-monitoring
```

---

## Troubleshooting

| Problem | Przyczyna | Rozwiazanie |
|---|---|---|
| `401 Unauthorized` | Zly token lub wygasl | Sprawdz token z curl, regeneruj Secret |
| `403 Forbidden` | Brak ClusterRoleBinding | Sprawdz: `oc get clusterrolebinding zabbix-monitoring-view` |
| `Route not found` | Monitoring wylaczony | `oc get route -n openshift-monitoring` |
| SSL error | Self-signed cert | Uzyj `-e validate_certs=false` lub dodaj CA do Zabbix |
| Token pusty po 60s | Problem z kontrolerem | Sprawdz: `oc describe secret zabbix-monitoring-reader-token -n openshift-monitoring` |
| `No result` w Zabbix | Zly JSONPath | Sprawdz odpowiedz API curl - moze brak danych dla danego operatora |
