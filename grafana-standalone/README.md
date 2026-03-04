# OCP Grafana Standalone — Multi-Cluster Monitoring

Grafana zainstalowana jako usługa systemd na dedykowanym hoście Linux.
Każdy klaster OCP = osobny datasource. Wybierasz klaster z dropdown
i widzisz ten sam dashboard dla wybranego środowiska.

---

## Architektura

```
┌─────────────────────────────────────────────────────┐
│  Linux Host (grafana-server)                        │
│                                                     │
│  grafana-server.service (systemd, port 3000)        │
│  /etc/grafana/provisioning/datasources/             │
│    ├── ocp-prod.yml  ──────────────────────────────┼──→  Thanos Querier (prod)
│    ├── ocp-dev.yml   ──────────────────────────────┼──→  Thanos Querier (dev)
│    └── ocp-staging.yml ───────────────────────────┼──→  Thanos Querier (staging)
│  /var/lib/grafana/dashboards/                       │
│    └── ocp-cluster-overview.json                   │
└─────────────────────────────────────────────────────┘
                          ▲
                          │  ansible-playbook (SSH)
                          │
┌─────────────────────────────────────────────────────┐
│  Maszyna Ansible (Twój laptop / bastion)            │
│  vars/clusters.yml  ←─ edytujesz tutaj             │
└─────────────────────────────────────────────────────┘
```

---

## Struktura plików

```
grafana-standalone/
├── inventory/
│   └── hosts.ini                  ← IP/hostname serwera Grafany
├── vars/
│   └── clusters.yml               ← Lista klastrow + haslo Grafany
├── templates/
│   ├── datasource.yml.j2          ← Template datasource per klaster
│   └── dashboard-provider.yml.j2 ← Konfiguracja providera dashboardow
├── files/
│   └── ocp-cluster-overview.json ← Dashboard JSON (multi-cluster)
├── install-grafana.yml            ← Krok 1: instalacja Grafany
├── add-clusters.yml               ← Krok 2: dodawanie klastrow OCP
└── README.md
```

---

## Wymagania

### Host Grafany
- RHEL 8/9, CentOS Stream, Fedora
- Dostep SSH (ansible user z sudo)
- Dostep do internetu lub repo wewnetrznego (Grafana RPM)
- Port 3000 TCP wolny (lub inny w `grafana_port`)
- Sieciowy dostep do Thanos Querier Route kazdego klastra OCP

### Maszyna Ansible
```bash
pip install kubernetes
ansible-galaxy collection install kubernetes.core ansible.posix
```

### Pliki kubeconfig
Kazdy klaster musi miec dostepny plik kubeconfig na maszynie Ansible:
```
~/.kube/prod     ← klaster produkcyjny
~/.kube/dev      ← klaster deweloperski
```

---

## Instalacja krok po kroku

### Krok 0 — Przygotuj konfigurację

**1. Edytuj `inventory/hosts.ini`** — podaj IP/hostname serwera Grafany:
```ini
[grafana]
grafana-server ansible_host=192.168.1.100 ansible_user=marek
```

**2. Edytuj `vars/clusters.yml`** — podaj klastry i haslo:
```yaml
grafana_admin_password: "TwojeHasloTutaj!"

ocp_clusters:
  - name: prod
    kubeconfig: "~/.kube/prod"
  - name: dev
    kubeconfig: "~/.kube/dev"
```

### Krok 1 — Zainstaluj Grafanę

```bash
ansible-playbook install-grafana.yml \
  -i inventory/hosts.ini \
  --ask-become-pass
```

Co robi:
- Dodaje repo Grafana RPM
- Instaluje pakiet `grafana`
- Konfiguruje `grafana.ini` (port, haslo, logi)
- Tworzy katalogi provisioning
- Wgrywa dashboard JSON
- Otwiera port w firewalld
- Uruchamia `grafana-server.service`

Po zakonczeniu Grafana dostepna pod: `http://<host>:3000`

### Krok 2 — Dodaj klastry OCP

```bash
ansible-playbook add-clusters.yml \
  -i inventory/hosts.ini \
  --ask-become-pass
```

Co robi dla każdego klastra z `vars/clusters.yml`:
- Tworzy `ServiceAccount` + nie-wygasający token w `openshift-monitoring`
- Pobiera Route `thanos-querier`
- Testuje połączenie (HTTP 200)
- Zapisuje plik datasource w `/etc/grafana/provisioning/datasources/`
- Przeładowuje Grafanę

### Krok 3 — Otwórz dashboard

```
http://<host-grafany>:3000/d/ocp-cluster-overview
```

1. Zaloguj jako `admin` / hasło z `clusters.yml`
2. W górnym dropdownie **"Klaster OCP"** wybierz środowisko
3. Dashboard automatycznie przełącza dane

---

## Dodawanie kolejnego klastra

1. Dodaj wpis w `vars/clusters.yml`:
```yaml
ocp_clusters:
  - name: staging
    kubeconfig: "~/.kube/staging"
```

2. Uruchom tylko `add-clusters.yml` (instalacja Grafany nie jest potrzebna):
```bash
ansible-playbook add-clusters.yml -i inventory/hosts.ini --ask-become-pass
```

Nowy datasource `OCP - staging` pojawi się automatycznie w dropdown.

---

## Troubleshooting

### Grafana nie startuje

```bash
# Na hoscie Grafany
sudo systemctl status grafana-server
sudo journalctl -u grafana-server -n 50
```

### Datasource nie widzi metryk (No data)

```bash
# Na maszynie Ansible — sprawdz token i URL
TOKEN=$(oc get secret grafana-monitoring-reader-token \
  -n openshift-monitoring \
  -o jsonpath='{.data.token}' --kubeconfig=~/.kube/prod | base64 -d)

THANOS=$(oc get route thanos-querier \
  -n openshift-monitoring \
  -o jsonpath='{.spec.host}' --kubeconfig=~/.kube/prod)

curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$THANOS/api/v1/query?query=cluster_operator_conditions" | jq .status
```

### Dropdown "Klaster OCP" jest pusty

Sprawdz czy datasource'y zostaly zapisane i maja prefix `OCP - `:
```bash
ls /etc/grafana/provisioning/datasources/
sudo cat /etc/grafana/provisioning/datasources/ocp-prod.yml
sudo systemctl reload grafana-server
```

### Grafana nie ma dostępu do klastra (sieć)

Serwer Grafany musi miec dostep do Route thanos-querier.
Sprawdz z hosta Grafany:
```bash
curl -sk https://<thanos-route>/api/v1/query?query=up
# Powinno zwrocic 401 (Unauthorized) — to znaczy ze siec dziala
```

---

## Sprzątanie jednego klastra

```bash
# Usun datasource z Grafany
sudo rm /etc/grafana/provisioning/datasources/ocp-dev.yml
sudo systemctl reload grafana-server

# Usun SA i token z OCP
oc delete clusterrolebinding grafana-monitoring-view --kubeconfig=~/.kube/dev
oc delete secret grafana-monitoring-reader-token -n openshift-monitoring --kubeconfig=~/.kube/dev
oc delete serviceaccount grafana-monitoring-reader -n openshift-monitoring --kubeconfig=~/.kube/dev
```

## Backup Grafany

Dane Grafany (dashboardy edytowane w UI, uzytkownicy, alerty):
```bash
sudo tar czf grafana-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/grafana \
  /etc/grafana
```
