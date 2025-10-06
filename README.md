# n8n Kubernetes Deployment

Kubernetes konfigurace pro nasazení [n8n](https://n8n.io/) workflow automation platformy na Rancher/k3s cluster s automatickým deploymentem přes GitHub Actions.

## 🚀 Vlastnosti

- ✅ Plně automatizované nasazení přes GitHub Actions
- ✅ PostgreSQL databáze pro persistenci
- ✅ SSL/TLS certifikáty přes Let's Encrypt (cert-manager)
- ✅ Traefik Ingress s automatickým HTTPS redirect
- ✅ Persistent storage pro n8n data
- ✅ Health checks a resource limity
- ✅ Self-hosted GitHub runner pro bezpečný deployment

## 📋 Prerekvizity

- Kubernetes cluster (Rancher/k3s)
- Traefik jako Ingress Controller
- cert-manager pro SSL certifikáty
- ClusterIssuer `letsencrypt-prod` nakonfigurovaný
- DNS záznam `n8n.carpiftw.cz` směřující na cluster
- Self-hosted GitHub Runner s přístupem ke clusteru

## 📁 Struktura projektu

```
.
├── .github/
│   ├── workflows/
│   │   └── deploy.yml          # GitHub Actions workflow pro automatický deployment
│   └── dependabot.yml          # Dependabot konfigurace
├── k8s/
│   ├── namespace.yaml          # Namespace n8n-carpiftw
│   ├── n8n-deployment.yaml     # n8n Deployment s health checks
│   ├── n8n-service.yaml        # n8n ClusterIP Service
│   ├── n8n-configmap.yaml      # Environment proměnné pro n8n
│   ├── pvc-data.yaml           # PVC pro n8n data (5Gi)
│   ├── certificate.yaml        # Let's Encrypt SSL certifikát
│   ├── ingress.yaml            # Traefik IngressRoute s HTTPS
│   ├── postgres-statefulset.yaml  # PostgreSQL 16 databáze
│   ├── postgres-service.yaml   # PostgreSQL Service
│   └── README.md               # Detailní K8s dokumentace
├── GITHUB_SETUP.md             # GitHub Actions setup guide
├── README.md                   # Tento soubor
└── .gitignore
```

## 🔧 Rychlý start

### 1. Nastavení GitHub Secrets

V GitHub repozitáři nastavte následující secrets:

```bash
# Kubernetes přístup
KUBECONFIG              # Obsah kubeconfig souboru

# PostgreSQL
POSTGRES_DATABASE       # Například: n8n
POSTGRES_USER          # Například: n8n
POSTGRES_PASSWORD      # Vygenerujte: openssl rand -base64 32

# n8n konfigurace
N8N_ENCRYPTION_KEY     # Vygenerujte: openssl rand -base64 32 (NIKDY NEMĚNIT!)
```

Detailní instrukce: [GITHUB_SETUP.md](GITHUB_SETUP.md)

### 2. Setup Self-hosted Runner

```bash
# Na serveru s přístupem ke clusteru
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

# Konfigurace (získejte token z GitHub → Settings → Actions → Runners)
./config.sh --url https://github.com/brabijan/n8n-k8s --token YOUR_TOKEN

# Spuštění jako služba
sudo ./svc.sh install
sudo ./svc.sh start
```

### 3. První nasazení

```bash
# Inicializace git repozitáře (pokud ještě není)
git init
git remote add origin git@github.com:brabijan/n8n-k8s.git

# Přidání souborů a push
git add .
git commit -m "Initial n8n k8s configuration"
git branch -M main
git push -u origin main
```

GitHub Actions workflow se automaticky spustí a nasadí n8n do clusteru.

### 4. Ověření nasazení

```bash
# Sledování deploymentu
kubectl get pods -n n8n-carpiftw -w

# Logy n8n
kubectl logs -f deployment/n8n -n n8n-carpiftw

# Stav všech komponent
kubectl get all -n n8n-carpiftw
```

Nebo sledujte progress v GitHub Actions UI: https://github.com/brabijan/n8n-k8s/actions

### 5. Přístup k n8n

Po úspěšném nasazení:

🌐 **https://n8n.carpiftw.cz**

## 🔄 Aktualizace konfigurace

```bash
# Upravte potřebné soubory v k8s/
vim k8s/n8n-deployment.yaml

# Commit a push
git add .
git commit -m "Update n8n configuration"
git push origin main

# Workflow automaticky nasadí změny
```

## 📊 Monitoring

### GitHub Actions

- Všechny deploymenty: https://github.com/brabijan/n8n-k8s/actions
- Sledování běžícího workflow: `gh run watch`
- Logy z workflow: `gh run view --log`

### Kubernetes

```bash
# Stav podů
kubectl get pods -n n8n-carpiftw

# Stav deploymentů
kubectl get deployments -n n8n-carpiftw

# Logy n8n
kubectl logs -f deployment/n8n -n n8n-carpiftw

# Logy PostgreSQL
kubectl logs -f statefulset/postgres -n n8n-carpiftw

# Events
kubectl get events -n n8n-carpiftw --sort-by='.lastTimestamp'
```

## 🛠️ Manuální operace

### Manuální spuštění workflow

GitHub UI: Actions → Deploy n8n → Run workflow

Nebo CLI:
```bash
gh workflow run deploy.yml --ref main
```

### Restart n8n

```bash
kubectl rollout restart deployment/n8n -n n8n-carpiftw
kubectl rollout status deployment/n8n -n n8n-carpiftw
```

### Backup databáze

```bash
kubectl exec -n n8n-carpiftw statefulset/postgres -- \
  pg_dump -U n8n n8n | gzip > n8n-backup-$(date +%Y%m%d).sql.gz
```

### Restore databáze

```bash
gunzip < n8n-backup-20231201.sql.gz | \
  kubectl exec -i -n n8n-carpiftw statefulset/postgres -- \
  psql -U n8n n8n
```

## 🐛 Troubleshooting

### n8n se nespouští

```bash
# Zkontrolovat logy
kubectl logs -f deployment/n8n -n n8n-carpiftw

# Describe pod
kubectl describe pod -l app=n8n -n n8n-carpiftw

# Events
kubectl get events -n n8n-carpiftw --sort-by='.lastTimestamp'
```

### PostgreSQL problémy

```bash
# Logy
kubectl logs -f statefulset/postgres -n n8n-carpiftw

# Připojení k databázi
kubectl exec -it -n n8n-carpiftw statefulset/postgres -- psql -U n8n
```

### SSL certifikát se nevystavuje

```bash
# Stav certifikátu
kubectl describe certificate n8n-carpiftw-tls-prod -n n8n-carpiftw

# cert-manager logy
kubectl logs -n cert-manager deployment/cert-manager
```

### GitHub Actions selhává

1. Zkontrolujte, že runner běží: GitHub → Settings → Actions → Runners
2. Ověřte GitHub Secrets: GitHub → Settings → Secrets
3. Zkontrolujte workflow logy v Actions tabu
4. Detaily viz [GITHUB_SETUP.md](GITHUB_SETUP.md)

## 📚 Dokumentace

- [k8s/README.md](k8s/README.md) - Detailní Kubernetes konfigurace
- [GITHUB_SETUP.md](GITHUB_SETUP.md) - GitHub Actions setup guide
- [n8n dokumentace](https://docs.n8n.io/)
- [Kubernetes dokumentace](https://kubernetes.io/docs/)

## 🔒 Bezpečnost

- ⚠️ **N8N_ENCRYPTION_KEY** je kritický - nikdy ho neměňte a zálohujte si ho!
- ⚠️ Nikdy necommitujte secrets do gitu
- ⚠️ KUBECONFIG by měl mít minimální potřebná oprávnění
- ⚠️ Self-hosted runner by měl běžet na důvěryhodném serveru
- ⚠️ Pravidelně aktualizujte runner na nejnovější verzi

## 📝 Resource konfigurace

### n8n
- Requests: 512Mi RAM, 500m CPU
- Limits: 1024Mi RAM, 1000m CPU
- Storage: 5Gi PVC

### PostgreSQL
- Requests: 256Mi RAM, 250m CPU
- Limits: 512Mi RAM, 500m CPU
- Storage: 10Gi (StatefulSet volumeClaimTemplate)

Upravte podle potřeby v příslušných YAML souborech.

## 🗑️ Odstranění

```bash
# VAROVÁNÍ: Tím se smažou VŠECHNA data včetně databáze!
kubectl delete namespace n8n-carpiftw
```

## 📄 Licence

MIT

## 👤 Autor

Jan Brabec (@brabijan)

## 🤝 Přispívání

Pull requesty jsou vítány! Pro větší změny nejdříve otevřete issue pro diskusi.

---

Made with ❤️ for workflow automation
