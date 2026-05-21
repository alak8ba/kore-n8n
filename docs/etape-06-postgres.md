# Étape 06 • PostgreSQL · migration depuis SQLite

## Objectif

Comprendre quand et comment passer de SQLite (défaut) à PostgreSQL, et comment ajouter un service Postgres colocalisé dans la stack Docker Compose.

---

## SQLite vs PostgreSQL : quand migrer ?

| Critère | SQLite | PostgreSQL |
|---|---|---|
| Usage solo, faible volume | ✅ Suffisant | Surdimensionné |
| Équipe, usage intensif | ⚠️ Limites | ✅ Recommandé |
| Backup granulaire | Copie de fichier | `pg_dump` |
| Haute disponibilité | ❌ | ✅ |
| Exécutions concurrentes | Limité | ✅ |

> **Règle simple :** tant que tu es seul et que le volume d'exécutions reste bas, SQLite tient. Dès qu'une équipe utilise l'instance ou que les exécutions deviennent intensives, migrer vers PostgreSQL.

---

## Ajouter PostgreSQL à la stack

### 1. Modifier `docker-compose.yml`

Ajouter le service `postgres` et une dépendance dans `n8n` :

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: kore-n8n-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=${DB_POSTGRESDB_DATABASE}
      - POSTGRES_USER=${DB_POSTGRESDB_USER}
      - POSTGRES_PASSWORD=${DB_POSTGRESDB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - traefik_public
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_POSTGRESDB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    ...
    depends_on:
      postgres:
        condition: service_healthy
    ...

volumes:
  n8n_data:
    driver: local
  postgres_data:      # ← ajouter ce volume
    driver: local
```

### 2. Configurer le `.env`

```env
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres         # nom du service Docker
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=change_me_strong_password
```

> `DB_POSTGRESDB_HOST=postgres` : dans Docker Compose, les services se joignent par leur nom de service, pas par `localhost`.

---

## Migration des données existantes

n8n ne migre pas automatiquement les données de SQLite vers PostgreSQL. Les workflows et credentials doivent être exportés manuellement.

### Exporter depuis SQLite

1. Ouvrir l'interface n8n
2. **Settings → Import/Export → Export all workflows**
3. Noter les credentials manuellement (ils ne s'exportent pas pour des raisons de sécurité)

### Démarrer avec PostgreSQL

```bash
# Arrêter l'instance SQLite
docker compose down

# Mettre à jour .env (DB_TYPE=postgresdb + variables Postgres)
# Mettre à jour docker-compose.yml (ajouter service postgres)

# Redémarrer
docker compose up -d

# Vérifier
docker compose logs -f n8n
```

### Importer dans la nouvelle instance

1. Se connecter à la nouvelle instance (nouveau compte admin à créer)
2. **Settings → Import/Export → Import workflows**
3. Recréer les credentials manuellement

---

## Commandes utiles PostgreSQL

```bash
# Entrer dans le conteneur Postgres
docker exec -it kore-n8n-postgres psql -U n8n -d n8n

# Vérifier les tables créées par n8n
\dt

# Backup de la base
docker exec kore-n8n-postgres pg_dump -U n8n n8n > backup_$(date +%F).sql

# Restaurer un backup
docker exec -i kore-n8n-postgres psql -U n8n -d n8n < backup_2026-01-01.sql
```

---

## Concepts clés

| Concept | Explication |
|---|---|
| **`depends_on` + `healthcheck`** | n8n attend que Postgres soit prêt avant de démarrer |
| **Nom de service comme hostname** | Dans Compose, chaque service est joignable par son nom depuis les autres services |
| **`postgres:16-alpine`** | Image légère, suffisante pour n8n |
| **`pg_dump`** | Outil de backup PostgreSQL · produit un fichier SQL restaurable |

---

## Prochaine étape

→ **Étape 07** : Stratégie de backup · sauvegarder le volume `n8n_data` et la base PostgreSQL.
