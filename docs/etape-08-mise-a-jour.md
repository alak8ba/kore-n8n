# Étape 08 • Mise à jour · stratégie de montée de version n8n

## Objectif

Monter la version de n8n de façon contrôlée, sans perte de données, avec possibilité de rollback.

---

## Principe

n8n publie des mises à jour fréquentes (parfois plusieurs par semaine). La stratégie du socle :

1. **Pincer la version** dans `.env` (`N8N_VERSION=1.90.0`)
2. **Lire le changelog** avant de monter
3. **Sauvegarder** avant de toucher quoi que ce soit
4. **Monter**, vérifier, valider

---

## Étapes de mise à jour

### 1. Identifier la version actuelle

```bash
docker inspect kore-n8n --format='{{.Config.Image}}'
# n8nio/n8n:1.90.0
```

Ou consulter l'UI n8n → Settings → About.

### 2. Consulter le changelog

Toujours lire les notes de version avant de mettre à jour :
- [github.com/n8n-io/n8n/releases](https://github.com/n8n-io/n8n/releases)
- Chercher les mentions **breaking changes** ou **migration required**

### 3. Sauvegarder

Avant toute mise à jour :

```bash
# Backup du volume
docker run --rm \
  -v kore-n8n_n8n_data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/n8n_data_before_upgrade_$(date +%F).tar.gz -C /data .
```

### 4. Mettre à jour la version dans `.env`

```env
# Avant
N8N_VERSION=1.90.0

# Après
N8N_VERSION=1.91.0
```

### 5. Puller la nouvelle image et redémarrer

```bash
docker compose pull
docker compose up -d
```

`docker compose pull` télécharge la nouvelle image sans couper le service. `up -d` recrée le conteneur avec la nouvelle image.

### 6. Vérifier

```bash
docker compose logs -f n8n
```

Attendre la ligne :

```
Editor is now accessible via: https://n8n.yourdomain.com/
```

Puis vérifier dans l'UI que les workflows sont intacts.

---

## Rollback

Si la mise à jour pose problème :

```bash
# Revenir à l'ancienne version dans .env
N8N_VERSION=1.90.0

# Restaurer le backup si nécessaire
docker compose stop n8n
docker run --rm \
  -v kore-n8n_n8n_data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/n8n_data_before_upgrade_2026-01-01.tar.gz -C /data

# Redémarrer avec l'ancienne image
docker compose pull
docker compose up -d
```

> ⚠️ n8n peut appliquer des migrations de base de données lors d'une montée de version. Un rollback après migration peut laisser la base dans un état incohérent. C'est pourquoi le backup **avant** la mise à jour est non négociable.

---

## Trouver les dernières versions disponibles

```bash
# Lister les tags récents via Docker Hub API
curl -s "https://registry.hub.docker.com/v2/repositories/n8nio/n8n/tags?page_size=10" \
  | grep -o '"name":"[^"]*"' | head -10
```

Ou consulter directement : [hub.docker.com/r/n8nio/n8n/tags](https://hub.docker.com/r/n8nio/n8n/tags)

---

## Cadence recommandée

| Contexte | Fréquence |
|---|---|
| Usage personnel | Toutes les 2-4 semaines, après lecture du changelog |
| Usage équipe / production | Mensuel, sur environnement de test d'abord |
| Patch de sécurité critique | Immédiatement |

---

## Concepts clés

| Concept | Explication |
|---|---|
| **Version pinnée** | Utiliser un tag précis (`1.90.0`) plutôt que `latest` pour contrôler les upgrades |
| **`docker compose pull`** | Télécharge les nouvelles images sans arrêter les conteneurs |
| **Migration de BDD** | n8n peut modifier son schéma à chaque version · d'où l'importance du backup pre-upgrade |
| **Rollback** | Revenir à l'état précédent en combinant ancienne image + restauration du backup |
