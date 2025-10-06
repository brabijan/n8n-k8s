# GitHub Actions Setup

Tento dokument popisuje nastavení GitHub Actions pro automatické nasazení n8n do Kubernetes clusteru.

## Přehled

Repozitář obsahuje GitHub Actions workflow, který automaticky nasazuje n8n do Kubernetes clusteru při každém push do `main` větve nebo při manuálním spuštění.

## Požadavky

1. **Self-hosted GitHub Runner** na serveru s přístupem ke Kubernetes clusteru
2. **kubectl** nainstalovaný na runneru
3. Přístup ke Kubernetes clusteru (kubeconfig)
4. Nakonfigurované GitHub Secrets

## GitHub Secrets

V repozitáři `brabijan/n8n-k8s` je potřeba nastavit následující secrets:

### Kubernetes přístup

#### `KUBECONFIG`
Celý obsah kubeconfig souboru pro přístup do Kubernetes clusteru.

```bash
# Získání kubeconfig (například z Rancher)
# Nebo z lokálního systému:
cat ~/.kube/config
```

### External PostgreSQL Database

#### `DATABASE_HOST`
Hostname nebo IP adresa externího PostgreSQL serveru.

```
Příklad: postgres.example.com
```

#### `DATABASE_PORT`
Port PostgreSQL serveru (volitelné, výchozí 5432).

```
Příklad: 5432
```

#### `DATABASE_NAME`
Název databáze pro n8n.

```
Příklad: n8n
```

#### `DATABASE_USER`
Uživatelské jméno pro PostgreSQL.

```
Příklad: n8n
```

#### `DATABASE_PASSWORD`
Heslo pro PostgreSQL uživatele.

### n8n konfigurace

#### `N8N_ENCRYPTION_KEY`
Encryption key pro n8n - kritický pro zabezpečení dat!

```bash
# Vygenerování encryption key
openssl rand -base64 32
```

**VAROVÁNÍ:** Tento klíč nikdy neměňte po prvním nasazení! Ztratili byste přístup k zašifrovaným datům (credentials, atd.).

## Nastavení secrets v GitHub

1. Jděte do repozitáře na GitHubu: `https://github.com/brabijan/n8n-k8s`
2. Klikněte na **Settings**
3. V levém menu vyberte **Secrets and variables** → **Actions**
4. Klikněte na **New repository secret**
5. Přidejte každý secret jednotlivě:
   - Name: `KUBECONFIG`
   - Secret: (obsah kubeconfig souboru)
   - Klikněte **Add secret**
6. Opakujte pro všechny ostatní secrets

## Self-hosted Runner

### Instalace runneru

1. V repozitáři jděte do **Settings** → **Actions** → **Runners**
2. Klikněte **New self-hosted runner**
3. Vyberte OS (Linux) a architekturu
4. Postupujte podle zobrazených instrukcí na serveru:

```bash
# Stažení
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Konfigurace
./config.sh --url https://github.com/brabijan/n8n-k8s --token YOUR_TOKEN

# Spuštění jako služba
sudo ./svc.sh install
sudo ./svc.sh start
```

### Ověření kubectl na runneru

```bash
# Přihlaste se na server kde běží runner
kubectl version --client
kubectl cluster-info
```

## Workflow soubory

### `.github/workflows/deploy.yml`

Hlavní workflow pro deployment. Spouští se:
- Automaticky při push do `main` větve
- Manuálně přes GitHub UI (Actions → Deploy n8n → Run workflow)

**Kroky workflow:**
1. Checkout kódu
2. Setup kubectl a kubeconfig
3. Vytvoření namespace `n8n-carpiftw`
4. Vytvoření/aktualizace Kubernetes secrets
5. Nasazení PostgreSQL
6. Čekání na PostgreSQL ready stav
7. Nasazení n8n (PVC, ConfigMap, Deployment, Service)
8. Nasazení SSL certifikátu a Ingress
9. Rollout restart deploymentu
10. Ověření, že vše běží
11. Výpis stavu a logů

### `.github/dependabot.yml`

Automatické aktualizace GitHub Actions dependencies.

## Použití

### Automatické nasazení

Při každém push do `main` větve se automaticky spustí deployment:

```bash
git add .
git commit -m "Update k8s configuration"
git push origin main
```

### Manuální nasazení

1. Jděte na GitHub: `https://github.com/brabijan/n8n-k8s/actions`
2. Vyberte workflow **Deploy n8n to Kubernetes**
3. Klikněte **Run workflow**
4. Volitelně zadejte důvod nasazení
5. Klikněte **Run workflow**

## Monitoring deploymentu

### V GitHub UI

1. Jděte do **Actions** tabu
2. Klikněte na běžící/poslední workflow run
3. Sledujte průběh jednotlivých kroků
4. V kroku "Verify deployment" uvidíte stav všech komponent

### Z příkazové řádky

```bash
# Sledovat stav podů
kubectl get pods -n n8n-carpiftw -w

# Sledovat logy n8n
kubectl logs -f deployment/n8n -n n8n-carpiftw

# Sledovat logy PostgreSQL
kubectl logs -f statefulset/postgres -n n8n-carpiftw

# Stav deploymentu
kubectl rollout status deployment/n8n -n n8n-carpiftw
```

## Troubleshooting

### Workflow selhává na "Setup kubectl"

**Problém:** kubectl není dostupný na runneru.

**Řešení:**
```bash
# Na serveru s runnerem
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mkdir -p $HOME/bin
mv kubectl $HOME/bin/
```

### Workflow selhává na "Create PostgreSQL secret"

**Problém:** Secrets nejsou nastaveny v GitHubu.

**Řešení:** Zkontrolujte, že všechny secrets jsou správně nastaveny v GitHub Settings → Secrets.

### Workflow selhává na "Wait for PostgreSQL to be ready"

**Problém:** PostgreSQL se nespouští.

**Řešení:** Zkontrolujte logy PostgreSQL:
```bash
kubectl logs -l app=postgres -n n8n-carpiftw
kubectl describe pod -l app=postgres -n n8n-carpiftw
```

### Workflow selhává na "Wait for n8n deployment to be ready"

**Problém:** n8n se nespouští.

**Řešení:** Zkontrolujte logy n8n:
```bash
kubectl logs -l app=n8n -n n8n-carpiftw
kubectl describe pod -l app=n8n -n n8n-carpiftw
```

### Secret nesprávně nastaven

**Problém:** Secret má špatnou hodnotu.

**Řešení:** Secrets můžete aktualizovat v GitHub UI a znovu spustit workflow.

## Bezpečnost

- **Nikdy** necommitujte secrets do gitu
- **KUBECONFIG** by měl mít minimální potřebná oprávnění
- **N8N_ENCRYPTION_KEY** je kritický - zálohujte si ho bezpečně
- Self-hosted runner by měl běžet na důvěryhodném serveru
- Pravidelně aktualizujte runner na nejnovější verzi

## První nasazení

1. **Nastavte všechny GitHub Secrets** (viz výše)
2. **Nainstalujte a spusťte self-hosted runner**
3. **Pushněte kód do main větve** nebo **spusťte workflow manuálně**
4. **Sledujte progress** v GitHub Actions
5. **Ověřte nasazení** na `https://n8n.carpiftw.cz`

## Aktualizace konfigurace

Pro aktualizaci Kubernetes manifestů:

1. Upravte soubory v `k8s/` adresáři
2. Commitněte a pushněte do main větve
3. Workflow automaticky aplikuje změny
4. n8n deployment se automaticky restartuje s novou konfigurací

## Užitečné příkazy

```bash
# Manuální trigger workflow z CLI (vyžaduje gh CLI)
gh workflow run deploy.yml --ref main

# Sledování posledního workflow runu
gh run watch

# Zobrazení logů z posledního runu
gh run view --log

# Seznam všech runů
gh run list --workflow=deploy.yml
```

## Podpora

Pokud narazíte na problémy:

1. Zkontrolujte logy v GitHub Actions UI
2. Zkontrolujte stav komponent v Kubernetes
3. Ověřte, že všechny secrets jsou správně nastaveny
4. Zkontrolujte, že runner běží a má přístup ke clusteru

## Reference

- [GitHub Actions dokumentace](https://docs.github.com/en/actions)
- [Self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [kubectl dokumentace](https://kubernetes.io/docs/reference/kubectl/)
- [n8n dokumentace](https://docs.n8n.io/)
