# Étape 04 • Variables & secrets — .env, encryption key, sécurité

## Objectif

Comprendre le rôle de chaque variable du `.env`, savoir lesquelles sont critiques, et ne jamais commettre d'erreur de sécurité en versionant des secrets.

---

## Ce qu'on a créé

```
kore-n8n/
├── .env.example   ← versionné, sans valeurs sensibles
└── .env           ← ignoré par git, contient les vraies valeurs
```

---

## Variables critiques

### `N8N_ENCRYPTION_KEY`

C'est la variable **la plus importante** du socle. n8n chiffre tous les credentials (clés API, mots de passe) stockés dans ses workflows avec cette clé.

```bash
# Générer une clé sécurisée
openssl rand -hex 32
```

> **Si cette clé change, tous les credentials chiffrés deviennent illisibles.** Stocker cette valeur dans un gestionnaire de secrets (Bitwarden, Vault, 1Password) et ne jamais la perdre.

### `N8N_HOST` et `WEBHOOK_URL`

Ces deux variables doivent être cohérentes :

```env
N8N_HOST=n8n.yourdomain.com
WEBHOOK_URL=https://n8n.yourdomain.com/
```

`WEBHOOK_URL` est utilisée par n8n pour construire les URLs publiques de ses webhooks. Une incohérence entre les deux brise les intégrations entrantes.

---

## Variables par domaine

### Authentification

```env
N8N_BASIC_AUTH_ACTIVE=false
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=change_me
```

À activer avant toute exposition publique si la gestion utilisateurs n8n n'est pas encore configurée. La basic auth ajoute une couche de protection au niveau du proxy.

```env
N8N_USER_MANAGEMENT_DISABLED=false
```

Laisser à `false` pour bénéficier du système multi-utilisateurs natif de n8n (invitations, rôles). Passer à `true` uniquement pour un usage solo sans interface de connexion.

### Base de données

```env
DB_TYPE=sqlite
```

SQLite est suffisant pour un usage personnel ou de faible charge. Pour la production à volume, voir **Étape 06** (migration PostgreSQL).

### Exécutions

```env
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=336
```

336 heures = 14 jours. n8n supprime automatiquement les logs d'exécution plus anciens. Réduire cette valeur si le volume d'exécutions est élevé.

### Timezone

```env
GENERIC_TIMEZONE=Europe/Paris
```

Affecte les triggers cron et les timestamps dans l'UI. Utiliser un identifiant IANA valide.

---

## Ce que le .gitignore protège

```gitignore
.env
.env.local
.env.*.local
```

Le fichier `.env` contient des valeurs réelles — il ne doit **jamais** apparaître dans un commit. `.env.example` est la version sans secrets, versionnée comme référence.

---

## Bonnes pratiques

| Pratique | Pourquoi |
|---|---|
| Générer `N8N_ENCRYPTION_KEY` avec `openssl rand -hex 32` | Entropie maximale, non devinable |
| Stocker la clé dans un gestionnaire de secrets | Survie au-delà du serveur |
| Versionner `.env.example` à jour | Onboarding sans appel téléphonique |
| Ne jamais mettre de secrets dans `docker-compose.yml` | Ce fichier est versionné |
| Restreindre les permissions du `.env` sur le serveur | `chmod 600 .env` |

---

## Concepts clés

| Concept | Explication |
|---|---|
| **Chiffrement at rest** | Les credentials n8n sont chiffrés sur disque avec `N8N_ENCRYPTION_KEY` |
| **Variable avec défaut** | `${VAR:-default}` dans Compose : valeur de fallback si la variable est absente |
| **`.env.example`** | Fichier modèle versionné — liste les variables sans leurs valeurs sensibles |
| **Webhook URL** | URL publique que les services externes appellent pour déclencher un workflow |

---

## Prochaine étape

→ **Étape 05** : Premier démarrage — vérification des logs, accès à l'interface, création du compte admin.
