# OCP Grafana Standalone — Multi-Cluster Monitoring

Grafana zainstalowana jako usługa systemd na dedykowanym hoście Linux (RHEL).
Każdy klaster OCP = osobny datasource. Klastry dzielone na dwa typy —
**INFRA** i **APP** — każdy typ ma osobny dashboard z własnym dropdownem.

---

## Architektura

```
┌─────────────────────────────────────────────────────────────┐
│  Linux Host — grafana-server (RHEL, systemd, port 3000)     │
│                                                             │
│  /etc/grafana/provisioning/datasources/                     │
│    ├── ocp-prod.yml   "OCP-INFRA - prod" ─────────────────┼──→ Thanos (prod)
│    ├── ocp-dr.yml     "OCP-INFRA - dr"   ─────────────────┼──→ Thanos (dr)
│    ├── ocp-dev.yml    "OCP-APP  - dev"   ─────────────────┼──→ Thanos (dev)
│    └── ocp-test.yml   "OCP-APP  - test"  ─────────────────┼──→ Thanos (test)
│                                                             │
│  /var/lib/grafana/dashboards/                               │
│    ├── ocp-cluster-overview.json  ← dropdown: OCP-INFRA -* │
│    └── ocp-app-overview.json      ← dropdown: OCP-APP  -*  │
└─────────────────────────────────────────────────────────────┘
                        ▲
                        │ scp (brak SSH z bastionu do grafany)
                        │
┌─────────────────────────────────────────────────────────────┐
│  Bastion                                                    │
│  - oc login → aktywny kontekst OCP                         │
│  - ansible-playbook add-clusters.yml                        │
│  - generuje: ./generated/ocp-<name>.yml                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Dwa dashboardy — separacja INFRA vs APP

| Dashboard | Plik | Dla kogo | Regex dropdown |
|---|---|---|---|
| OCP Infra Clusters Overview | `ocp-cluster-overview.json` | klastry infrastrukturalne | `OCP-INFRA - .*` |
| OCP App Clusters Overview | `ocp-app-overview.json` | klastry aplikacyjne | `OCP-APP - .*` |

Klaster trafia do właściwego dashboardu przez prefiks nazwy datasource.
Podajesz go przy dodawaniu klastra przez `-e cluster_type=INFRA` lub `-e cluster_type=APP`.

---

## Zawartość dashboardów

Oba dashboardy zawierają identyczne sekcje:

| Row | Panele |
|---|---|
| **Cluster Health** | Wersja OCP, węzły, pody Running, pody z problemem, operatory Degraded/Progressing |
| **Utilizacja** | CPU per node (%), Memory per node (%) — timeseries |
| **Cluster Operators** | Tabela wszystkich operatorów: Available / Degraded / Progressing |
| **etcd Health** | Leader, Members, WAL fsync P99, DB Size |
| **API Server** | Request rate, Error rate 5xx, Latency P99 |
| **MachineConfigPool** | Tabela poolów: Total / Updated / Degraded / Unavailable |
| **Firing Alerts** | Tabela aktywnych alertów z kolorowaniem severity |
| **Pod Restarts** | Top 15 podów z restartami w ostatniej godzinie |
| **PersistentVolumes** | Wykres % zajętości PV + tabela z wolnym miejscem |
| **Certyfikaty** | Dni do wygaśnięcia certów kubelet per node |

---

## Struktura plików

```
grafana-standalone/
├── inventory/
│   └── hosts.ini                   ← IP/hostname serwera Grafany + bastion
├── vars/
│   └── clusters.yml                ← Konfiguracja Grafany (port, haslo)
├── templates/
│   ├── datasource.yml.j2           ← Template datasource per klaster
│   └── dashboard-provider.yml.j2  ← Konfiguracja providera dashboardow
├── files/
│   ├── ocp-cluster-overview.json  ← Dashboard INFRA (OCP-INFRA - .*)
│   └── ocp-app-overview.json      ← Dashboard APP  (OCP-APP  - .*)
├── generated/                      ← Wygenerowane datasource YAML (gitignore)
├── install-grafana.yml             ← Instalacja Grafany na hoscie Linux
├── add-clusters.yml                ← Dodawanie klastra OCP (z bastionu)
└── README.md
```

---

## Wymagania

### Host Grafany
- RHEL 8/9 z aktywna subskrypcja i repo AppStream (pakiet `grafana`)
- Uzytkownik z `sudo`
- Port 3000 TCP otwarty

### Bastion (skad uruchamiasz `add-clusters.yml`)
- `oc` CLI zainstalowane i zalogowane do klastra
- Python + kubernetes client:
```bash
pip install kubernetes
ansible-galaxy collection install kubernetes.core ansible.posix
```

### Sieć
- Bastion → OCP API (port 6443) ✓
- Bastion → Thanos Querier Route (HTTPS) ✓ — do testu polaczenia
- Bastion → Host Grafany — **nie jest wymagane** (plik generowany lokalnie, kopiujesz rcznie)
- Host Grafany → Thanos Querier Route (HTTPS) ✓ — do odpytywania metryk

---

## Instalacja — krok po kroku

### Krok 0 — Konfiguracja

**`inventory/hosts.ini`** — podaj IP hosta Grafany:
```ini
[grafana]
grafana-server ansible_host=192.168.1.100 ansible_user=marek

[bastion]
bastion ansible_connection=local ansible_python_interpreter=/usr/bin/python3
```

**`vars/clusters.yml`** — zmień hasło admina:
```yaml
grafana_admin_password: "TwojeHasloTutaj!"
grafana_port: 3000
```

### Krok 1 — Instalacja Grafany

Uruchom z maszyny z SSH do hosta Grafany:
```bash
ansible-playbook install-grafana.yml \
  -i inventory/hosts.ini \
  --ask-become-pass
```

Co robi:
- Weryfikuje dostępność pakietu `grafana` w RHEL AppStream
- Instaluje pakiet `grafana` (bez dostępu do internetu)
- Konfiguruje `grafana.ini` (port, hasło, logi, brak alertingu)
- Tworzy katalogi provisioning
- Kopiuje **oba dashboardy** JSON
- Otwiera port w firewalld
- Uruchamia `grafana-server.service`

### Krok 2 — Dodawanie klastrów OCP

Uruchamiasz **z bastionu** dla każdego klastra osobno:

```bash
# Klaster infrastrukturalny
oc login https://api.prod.example.com -u admin -p haslo
ansible-playbook add-clusters.yml \
  -i inventory/hosts.ini \
  -e cluster_name=prod \
  -e cluster_type=INFRA

# Klaster aplikacyjny
oc login https://api.dev.example.com -u admin -p haslo
ansible-playbook add-clusters.yml \
  -i inventory/hosts.ini \
  -e cluster_name=dev \
  -e cluster_type=APP
```

Playbook:
1. Sprawdza `oc whoami` — czy jesteś zalogowany i do jakiego klastra
2. Tworzy `ServiceAccount grafana-monitoring-reader` w `openshift-monitoring`
3. Nadaje `ClusterRoleBinding cluster-monitoring-view` (read-only)
4. Tworzy nie-wygasający token (`kubernetes.io/service-account-token`)
5. Pobiera Route `thanos-querier`
6. Testuje połączenie (HTTP 200)
7. Generuje plik `./generated/ocp-<name>.yml`
8. Wypisuje instrukcję co zrobić dalej

### Krok 3 — Skopiuj datasource na hosta Grafany

Playbook nie ma SSH do hosta Grafany — kopiujesz ręcznie:

```bash
# Z maszyny z dostępem SSH do grafany
scp generated/ocp-prod.yml marek@grafana:/tmp/

# Na hoscie Grafany
sudo mv /tmp/ocp-prod.yml /etc/grafana/provisioning/datasources/
sudo chown grafana:grafana /etc/grafana/provisioning/datasources/ocp-prod.yml
sudo systemctl reload grafana-server
```

### Krok 4 — Otwórz dashboardy

```
http://<grafana-host>:3000/d/ocp-cluster-overview   ← klastry INFRA
http://<grafana-host>:3000/d/ocp-app-overview        ← klastry APP
```

Zaloguj jako `admin` / hasło z `vars/clusters.yml`.
Wybierz klaster z dropdown — dashboard przełącza dane.

---

## Aktualizacja dashboardów

Gdy zmienisz pliki JSON (nowe panele, poprawki):

**Opcja A — ponownie install-grafana.yml (zalecane)**
```bash
ansible-playbook install-grafana.yml -i inventory/hosts.ini --ask-become-pass
```
Ansible wykryje zmienione pliki JSON, skopiuje je i zrestartuje Grafanę. Reszta tasków bez zmian.

**Opcja B — ręcznie**
```bash
scp files/ocp-cluster-overview.json marek@grafana:/tmp/
scp files/ocp-app-overview.json     marek@grafana:/tmp/

# Na hoscie Grafany
sudo cp /tmp/ocp-*.json /var/lib/grafana/dashboards/
sudo chown grafana:grafana /var/lib/grafana/dashboards/ocp-*.json
sudo systemctl reload grafana-server
```

---

## Migracja istniejących klastrów (stara nazwa datasource)

Klastry dodane przed wprowadzeniem podziału INFRA/APP mają nazwę `OCP - <name>`
i nie pasują do żadnego dashboardu. Żeby je przenieść:

```bash
# 1. Zaloguj się do klastra
oc login https://api.prod.example.com ...

# 2. Ponownie odpal add-clusters.yml z cluster_type
ansible-playbook add-clusters.yml -i inventory/hosts.ini \
  -e cluster_name=prod -e cluster_type=INFRA

# 3. Na hoscie Grafany — usuń stary datasource, wgraj nowy
sudo rm /etc/grafana/provisioning/datasources/ocp-prod.yml
scp generated/ocp-prod.yml marek@grafana:/tmp/
sudo mv /tmp/ocp-prod.yml /etc/grafana/provisioning/datasources/
sudo chown grafana:grafana /etc/grafana/provisioning/datasources/ocp-prod.yml
sudo systemctl reload grafana-server
```

Grafana po reload usunie `OCP - prod` i doda `OCP-INFRA - prod`.

---

## Troubleshooting

### Grafana nie startuje
```bash
sudo systemctl status grafana-server
sudo journalctl -u grafana-server -n 50
```

### Pakiet grafana niedostępny w repo
```bash
subscription-manager repos --list-enabled | grep -i appstream
dnf repolist | grep appstream
dnf info grafana
```

### Dropdown jest pusty (brak klastrów)
```bash
# Sprawdz pliki datasource i ich nazwy
ls /etc/grafana/provisioning/datasources/
sudo cat /etc/grafana/provisioning/datasources/ocp-prod.yml | grep "name:"
# Nazwa musi pasowac do regex dashboardu:
#   OCP Infra: "OCP-INFRA - prod"
#   OCP App:   "OCP-APP - dev"
sudo systemctl reload grafana-server
```

### No data w panelach
```bash
# Test z bastionu — sprawdz token i URL
TOKEN=$(oc get secret grafana-monitoring-reader-token \
  -n openshift-monitoring -o jsonpath='{.data.token}' | base64 -d)
THANOS=$(oc get route thanos-querier \
  -n openshift-monitoring -o jsonpath='{.spec.host}')

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS/api/v1/query?query=cluster_operator_conditions" | jq .status
# Oczekiwane: "success"
```

### Grafana nie dociera do Thanos (sieć)
```bash
# Na hoscie Grafany — sprawdz czy widzi Route
curl -sk https://<thanos-route>/api/v1/query?query=up
# Powinno zwrocic 401 — siec dziala, brak tokenu to normalne
```

---

## Usuwanie klastra

```bash
# Na hoscie Grafany
sudo rm /etc/grafana/provisioning/datasources/ocp-prod.yml
sudo systemctl reload grafana-server

# W OCP
oc delete clusterrolebinding grafana-monitoring-view
oc delete secret grafana-monitoring-reader-token -n openshift-monitoring
oc delete serviceaccount grafana-monitoring-reader -n openshift-monitoring
```

---

## Backup Grafany

```bash
sudo tar czf grafana-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/grafana \
  /etc/grafana
```
