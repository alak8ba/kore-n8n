# Étape 05 • Premier démarrage · logs, accès UI, compte admin

## Objectif

Démarrer le socle pour la première fois, vérifier que tout fonctionne, et créer le compte administrateur n8n.

---

## Démarrer le conteneur

```bash
docker compose up -d
```

`-d` = detached, le conteneur tourne en arrière-plan.

---

## Vérifier le démarrage

### 1. État du conteneur

```bash
docker compose ps
```

Le conteneur `kore-n8n` doit afficher le statut `running`.

### 2. Logs en temps réel

```bash
docker compose logs -f n8n
```

À la fin du démarrage, tu dois voir une ligne du type :

```
Editor is now accessible via: https://n8n.yourdomain.com/
```

Si tu vois des erreurs, les causes les plus fréquentes sont listées en bas de ce document.

### 3. Vérifier le certificat TLS

```bash
curl -I https://n8n.yourdomain.com/
```

Réponse attendue : `HTTP/2 200` ou une redirection vers la page de login.

---

## Accès à l'interface

Ouvre `https://n8n.yourdomain.com/` dans ton navigateur.

Si c'est le premier démarrage avec la gestion utilisateurs activée (`N8N_USER_MANAGEMENT_DISABLED=false`), n8n affiche un formulaire de création du compte administrateur :

1. Renseigner prénom, nom, email, mot de passe
2. Ce compte devient le propriétaire de l'instance
3. Il peut ensuite inviter d'autres utilisateurs depuis les paramètres

---

## Commandes utiles

```bash
# Voir les logs
docker compose logs -f n8n

# Arrêter sans supprimer les données
docker compose stop

# Redémarrer
docker compose start

# Arrêter et supprimer le conteneur (les données dans le volume sont conservées)
docker compose down

# Arrêter et supprimer conteneur + volume (tout est perdu)
docker compose down -v
```

---

## Problèmes fréquents

### Le conteneur ne démarre pas

```bash
docker compose logs n8n
```

Vérifier :
- `N8N_ENCRYPTION_KEY` est définie et non vide dans `.env`
- `N8N_HOST` ne contient pas de `https://` (juste le hostname)

### Traefik ne route pas vers n8n

```bash
docker network inspect traefik_public
```

Le conteneur `kore-n8n` doit apparaître dans la liste. Si ce n'est pas le cas, vérifier que le réseau `traefik_public` est bien déclaré comme `external: true` dans `docker-compose.yml`.

### Certificat TLS non obtenu

- Vérifier que le DNS pointe vers le VPS (`dig n8n.yourdomain.com +short`)
- Vérifier que le port 80 est ouvert (Let's Encrypt HTTP-01 challenge)
- Consulter les logs de Traefik : `docker logs traefik`

### Les webhooks ne fonctionnent pas

Vérifier que `WEBHOOK_URL` correspond exactement à l'URL publique de l'instance, avec le slash final :

```env
WEBHOOK_URL=https://n8n.yourdomain.com/
```

---

## Concepts clés

| Concept | Explication |
|---|---|
| **`docker compose up -d`** | Démarre les services en arrière-plan |
| **`docker compose logs -f`** | Suit les logs en temps réel (`-f` = follow) |
| **Compte admin n8n** | Créé au premier démarrage, propriétaire de l'instance |
| **HTTP-01 challenge** | Méthode ACME : Let's Encrypt envoie une requête HTTP sur le domaine pour valider la propriété |

---

## Prochaine étape

→ **Étape 06** : Migrer de SQLite vers PostgreSQL pour un usage en production.
