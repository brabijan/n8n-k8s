# n8n Kubernetes Deployment - n8n.carpiftw.cz

Kubernetes konfigurace pro n8n provozované na Rancher/k3s clusteru s doménou `n8n.carpiftw.cz`.

## Prerekvizity

- Kubernetes cluster (Rancher/k3s)
- Nainstalovaný Traefik jako Ingress Controller
- Nainstalovaný cert-manager pro SSL certifikáty
- ClusterIssuer `letsencrypt-prod` pro Let's Encrypt certifikáty
- DNS záznam `n8n.carpiftw.cz` směřující na cluster

## Struktura souborů

```
k8s/
├── namespace.yaml              # Namespace n8n-carpiftw
├── n8n-deployment.yaml         # n8n Deployment
├── n8n-service.yaml            # n8n Service
├── n8n-configmap.yaml          # Konfigurace n8n
├── pvc-data.yaml               # PVC pro n8n data (5Gi)
├── certificate.yaml            # SSL certifikát
└── ingress.yaml                # Traefik IngressRoute s HTTPS
```

## Před nasazením

### 1. Vytvořit secrets

Před nasazením je nutné vytvořit dva Kubernetes secrets:

#### Database Secret (External PostgreSQL)

```bash
kubectl create secret generic database-secret \
  --from-literal=host=YOUR_DB_HOST \
  --from-literal=port=5432 \
  --from-literal=dbname=n8n \
  --from-literal=user=n8n \
  --from-literal=password=VASE_HESLO_PRO_POSTGRES \
  -n n8n-carpiftw
```

**Poznámka:** Používá se externí PostgreSQL server, ne PostgreSQL v clusteru.

#### n8n Encryption Key Secret

Vygenerujte náhodný encryption key:

```bash
openssl rand -base64 32
```

A vytvořte secret:

```bash
kubectl create secret generic n8n-secret \
  --from-literal=encryption-key=VAS_VYGENEROVANY_KLIC \
  -n n8n-carpiftw
```

**DŮLEŽITÉ:** Encryption key použitý v produkci NIKDY neměňte, jinak ztratíte přístup k zašifrovaným datům!

## Nasazení

### Krok 1: Vytvořit namespace

```bash
kubectl apply -f namespace.yaml
```

### Krok 2: Vytvořit secrets (viz výše)

### Krok 3: Nasadit n8n

```bash
kubectl apply -f pvc-data.yaml
kubectl apply -f n8n-configmap.yaml
kubectl apply -f n8n-deployment.yaml
kubectl apply -f n8n-service.yaml
```

### Krok 4: Nastavit SSL a Ingress

```bash
kubectl apply -f certificate.yaml
kubectl apply -f ingress.yaml
```

## Ověření nasazení

```bash
# Zkontrolovat pody
kubectl get pods -n n8n-carpiftw

# Zkontrolovat services
kubectl get svc -n n8n-carpiftw

# Zkontrolovat ingress
kubectl get ingressroute -n n8n-carpiftw

# Zkontrolovat certifikát
kubectl get certificate -n n8n-carpiftw

# Zkontrolovat logy n8n
kubectl logs -f deployment/n8n -n n8n-carpiftw

# Zkontrolovat logy PostgreSQL
kubectl logs -f statefulset/postgres -n n8n-carpiftw
```

## Přístup k aplikaci

Po úspěšném nasazení bude n8n dostupné na:

```
https://n8n.carpiftw.cz
```

## Konfigurace

### Environment proměnné

Většina konfigurace je v `n8n-configmap.yaml`. Pokud potřebujete změnit konfiguraci:

1. Upravte `n8n-configmap.yaml`
2. Aplikujte změny: `kubectl apply -f n8n-configmap.yaml`
3. Restartujte n8n: `kubectl rollout restart deployment/n8n -n n8n-carpiftw`

### Resource limity

Aktuální nastavení:

**n8n:**
- Requests: 512Mi RAM, 500m CPU
- Limits: 1024Mi RAM, 1000m CPU

**PostgreSQL:**
- Requests: 256Mi RAM, 250m CPU
- Limits: 512Mi RAM, 500m CPU

Upravte v příslušných YAML souborech podle potřeby.

### Úložiště

- **n8n data:** 5Gi PVC (`pvc-data.yaml`)
- **PostgreSQL data:** 10Gi (v `postgres-statefulset.yaml`)

## Zálohování

### Databáze

Zálohování PostgreSQL databáze:

```bash
kubectl exec -n n8n-carpiftw statefulset/postgres -- \
  pg_dump -U n8n n8n | gzip > n8n-backup-$(date +%Y%m%d).sql.gz
```

### n8n data

```bash
kubectl exec -n n8n-carpiftw deployment/n8n -- \
  tar czf - /home/node/.n8n > n8n-data-backup-$(date +%Y%m%d).tar.gz
```

## Troubleshooting

### n8n se nespouští

```bash
# Zkontrolovat logy
kubectl logs -f deployment/n8n -n n8n-carpiftw

# Zkontrolovat events
kubectl get events -n n8n-carpiftw --sort-by='.lastTimestamp'

# Zkontrolovat popis podu
kubectl describe pod -l app=n8n -n n8n-carpiftw
```

### Problémy s databází

```bash
# Zkontrolovat PostgreSQL logy
kubectl logs -f statefulset/postgres -n n8n-carpiftw

# Připojit se k PostgreSQL
kubectl exec -it -n n8n-carpiftw statefulset/postgres -- psql -U n8n
```

### SSL certifikát se nevystavuje

```bash
# Zkontrolovat stav certifikátu
kubectl describe certificate n8n-carpiftw-tls-prod -n n8n-carpiftw

# Zkontrolovat cert-manager logy
kubectl logs -n cert-manager deployment/cert-manager
```

## Update n8n

```bash
# Stáhnout nejnovější image
kubectl set image deployment/n8n n8n=n8nio/n8n:latest -n n8n-carpiftw

# Nebo restartovat s pull latest
kubectl rollout restart deployment/n8n -n n8n-carpiftw
```

## Odstranění

```bash
kubectl delete namespace n8n-carpiftw
```

**VAROVÁNÍ:** Tím se smažou všechna data včetně databáze!

## Poznámky

- Encryption key je kritický pro bezpečnost - nikdy ho neztratit a nezveřejňovat
- Pro produkční provoz doporučuji nastavit pravidelné zálohování
- Sledujte resource usage a podle potřeby upravte limity
- Pro vyšší dostupnost zvažte zvýšení `replicas` a použití PostgreSQL clusteru
