# Étape 07 • Backup — sauvegarder n8n_data et PostgreSQL

## Objectif

Mettre en place une stratégie de sauvegarde fiable pour ne perdre ni les workflows, ni les credentials, ni les données d'exécution.

---

## Ce qu'il faut sauvegarder

| Élément | Emplacement | Contient |
|---|---|---|
| Volume `n8n_data` | `/home/node/.n8n` dans le conteneur | Workflows, credentials (chiffrés), SQLite (si DB_TYPE=sqlite) |
| Base PostgreSQL | Conteneur `kore-n8n-postgres` | Tout (si DB_TYPE=postgresdb) |
| Fichier `.env` | Racine du projet sur le VPS | Clés, configuration — **critique** |

> **La perte de `N8N_ENCRYPTION_KEY` rend les credentials irrécupérables**, même si le volume est intact. Sauvegarder le `.env` est aussi important que le volume.

---

## Backup du volume n8n_data (mode SQLite)

```bash
docker run --rm \
  -v kore-n8n_n8n_data:/data \
  -v $(pwd)/backups:/backup \
  alpine \
  tar czf /backup/n8n_data_$(date +%F_%H%M).tar.gz -C /data .
```

Ce one-liner :
1. Lance un conteneur Alpine temporaire
2. Monte le volume `n8n_data` en lecture
3. Monte le dossier `./backups` de l'hôte
4. Crée une archive `.tar.gz` horodatée

### Restaurer

```bash
# Arrêter n8n
docker compose stop n8n

# Restaurer l'archive dans le volume
docker run --rm \
  -v kore-n8n_n8n_data:/data \
  -v $(pwd)/backups:/backup \
  alpine \
  tar xzf /backup/n8n_data_2026-01-01_0300.tar.gz -C /data

# Redémarrer
docker compose start n8n
```

---

## Backup PostgreSQL (si DB_TYPE=postgresdb)

```bash
mkdir -p backups
docker exec kore-n8n-postgres \
  pg_dump -U n8n n8n | gzip > backups/n8n_pg_$(date +%F_%H%M).sql.gz
```

### Restaurer

```bash
docker compose stop n8n

gunzip -c backups/n8n_pg_2026-01-01_0300.sql.gz | \
  docker exec -i kore-n8n-postgres psql -U n8n -d n8n

docker compose start n8n
```

---

## Automatiser avec cron

Sur le VPS, ajouter une tâche cron pour sauvegarder quotidiennement :

```bash
crontab -e
```

```cron
# Backup n8n chaque nuit à 3h00
0 3 * * * /home/user/kore-n8n/scripts/backup.sh >> /var/log/kore-n8n-backup.log 2>&1
```

Exemple de script `scripts/backup.sh` :

```bash
#!/bin/bash
set -e
BACKUP_DIR="/home/user/kore-n8n/backups"
DATE=$(date +%F_%H%M)

mkdir -p "$BACKUP_DIR"

# Volume n8n_data
docker run --rm \
  -v kore-n8n_n8n_data:/data \
  -v "$BACKUP_DIR":/backup \
  alpine tar czf "/backup/n8n_data_${DATE}.tar.gz" -C /data .

# Garder seulement les 14 derniers backups
find "$BACKUP_DIR" -name "n8n_data_*.tar.gz" -mtime +14 -delete

echo "Backup terminé : n8n_data_${DATE}.tar.gz"
```

---

## Bonnes pratiques

| Pratique | Pourquoi |
|---|---|
| Sauvegarder avant chaque mise à jour | Rollback possible si la montée de version échoue |
| Copier les backups hors du VPS | Un VPS perdu = backups locaux perdus |
| Sauvegarder le `.env` séparément | La `N8N_ENCRYPTION_KEY` est hors du volume |
| Tester la restauration régulièrement | Un backup non testé est un backup non fiable |

---

## Concepts clés

| Concept | Explication |
|---|---|
| **Volume nommé Docker** | Référencé par `<project>_<volume_name>` (ex: `kore-n8n_n8n_data`) |
| **`pg_dump`** | Export logique PostgreSQL — indépendant de la version mineure |
| **`tar czf`** | Créer une archive compressée (`c`=create, `z`=gzip, `f`=fichier) |
| **Rotation de backups** | Supprimer les backups anciens pour contrôler l'espace disque |

---

## Prochaine étape

→ **Étape 08** : Stratégie de mise à jour — comment monter la version de n8n sans interruption ni perte de données.
