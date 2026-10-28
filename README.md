# Label Studio sur Kubernetes

Déploiement de Label Studio via le chart Helm officiel sur **k3s / Rancher Desktop**, en réutilisant le PostgreSQL déjà présent dans le cluster.

- **Base de données** PostgreSQL (projets, tâches, annotations)
- **Médias** PVC de 10Gi (uploads, exports)
- **UI** `http://localhost:8508` (`admin@admin.com` / `admin`)

---

## Installation

### 1. Base PostgreSQL

```bash
psql -h localhost -p 5555 -U admin -d postgres <<'SQL'
CREATE USER labelstudio WITH PASSWORD 'admin';
CREATE DATABASE labelstudio OWNER labelstudio;
SQL

psql -h localhost -p 5555 -U admin -d labelstudio -c 'ALTER SCHEMA public OWNER TO labelstudio;'
```

### 2. Secret Kubernetes

Le chart exige un Secret pour le mot de passe PostgreSQL (les autres sont dans `ls-values.yaml`) :

```bash
kubectl create namespace label-studio
kubectl create secret generic ls-db-secret -n label-studio --from-literal=password=admin
```

### 3. Déploiement

```bash
helm repo add heartex https://charts.heartex.com/
helm install label-studio heartex/label-studio \
  --namespace label-studio --create-namespace \
  -f ls-values.yaml
```

Le premier démarrage prend une à deux minutes (migrations).

---

## Vérification

```bash
kubectl get pods -n label-studio
curl -s http://localhost:8508/health
```

---

## Opérations courantes

```bash
# Appliquer une modification de ls-values.yaml
helm upgrade label-studio heartex/label-studio -n label-studio -f ls-values.yaml

# Logs
kubectl logs -n label-studio deploy/label-studio-ls-app -c app -f --tail=100

# Sauvegarder la base
pg_dump -h localhost -p 5555 -U labelstudio -d labelstudio -Fc -f labelstudio-backup.dump
```

### Repartir de zéro

```bash
helm uninstall label-studio -n label-studio
kubectl delete namespace label-studio
psql -h localhost -p 5555 -U admin -d postgres -c 'DROP DATABASE IF EXISTS labelstudio WITH (FORCE);'
```

Puis reprendre l'installation à l'étape 1 (`CREATE USER` échouera si l'utilisateur existe déjà, ce n'est pas grave).
