# Étape 02 • Docker Compose — service n8n, volumes, réseau

## Objectif

Décrire le `docker-compose.yml` du socle : comment le service n8n est défini, pourquoi chaque section est là, et ce que chaque directive produit.

---

## Ce qu'on a créé

```
kore-n8n/
└── docker-compose.yml
```

---

## Le fichier complet

```yaml
services:
  n8n:
    image: n8nio/n8n:${N8N_VERSION:-latest}
    container_name: kore-n8n
    restart: unless-stopped
    environment:
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-https}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      ...
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - traefik_public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${N8N_HOST}`)"
      ...

volumes:
  n8n_data:
    driver: local

networks:
  traefik_public:
    external: true
```

---

## Explication section par section

### `image`

```yaml
image: n8nio/n8n:${N8N_VERSION:-latest}
```

`${N8N_VERSION:-latest}` : lit la variable `N8N_VERSION` du `.env`. Si elle est absente, utilise `latest`. En production, toujours pincer une version précise pour éviter les upgrades surprises.

### `restart: unless-stopped`

Le conteneur redémarre automatiquement après un crash ou un reboot du serveur, sauf si on l'arrête manuellement avec `docker compose stop`.

### `environment`

Les variables d'environnement injectées dans n8n au démarrage. Elles viennent toutes du `.env` — jamais de valeurs en dur dans ce fichier.

| Variable | Rôle |
|---|---|
| `N8N_HOST` | Hostname public de l'instance |
| `N8N_PORT` | Port interne du processus n8n (toujours 5678) |
| `N8N_PROTOCOL` | Protocole exposé aux utilisateurs (https) |
| `WEBHOOK_URL` | URL complète utilisée par n8n pour construire les URLs de webhooks |
| `N8N_ENCRYPTION_KEY` | Clé de chiffrement des credentials stockés |

### `volumes`

```yaml
volumes:
  - n8n_data:/home/node/.n8n
```

`/home/node/.n8n` est le répertoire où n8n stocke tout : workflows, credentials, logs d'exécution, et la base SQLite. Le volume Docker `n8n_data` le rend persistant entre les redémarrages.

### `networks`

```yaml
networks:
  traefik_public:
    external: true
```

`external: true` signifie que ce réseau **n'est pas créé** par ce Compose — il doit exister au préalable (voir Étape 01). C'est ce qui permet à Traefik de voir le conteneur n8n.

---

## Volume nommé vs bind mount

On utilise un **volume nommé** (`n8n_data`) plutôt qu'un bind mount (`./data:/home/node/.n8n`) pour deux raisons :

1. Docker gère les permissions automatiquement (n8n tourne sous l'utilisateur `node`, pas `root`)
2. Le volume est portable — il survit à un `docker compose down` sans options

---

## Concepts clés

| Concept | Explication |
|---|---|
| **Volume nommé** | Espace de stockage géré par Docker, référencé par un nom, indépendant du chemin hôte |
| **`external: true`** | Le réseau n'est pas créé par ce Compose — il doit exister avant le `up` |
| **Variable avec défaut** | `${VAR:-default}` : si VAR est vide ou absente, utilise `default` |
| **`restart: unless-stopped`** | Politique de redémarrage : automatique sauf arrêt explicite |

---

## Prochaine étape

→ **Étape 03** : Configurer les labels Traefik pour router le trafic et activer Let's Encrypt.
