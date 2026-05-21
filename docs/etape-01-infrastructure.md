# Étape 01 • Infrastructure — VPS, Docker, réseau Traefik

## Objectif

Préparer le terrain sur le VPS : s'assurer que Docker tourne, que Traefik est opérationnel, et que le réseau partagé entre les conteneurs existe.

---

## Ce qu'on valide ici

```
VPS Linux
├── Docker Engine ≥ 24
├── Docker Compose v2
├── Traefik v2 (conteneur existant)
│   └── réseau : traefik_public
└── DNS : n8n.yourdomain.com → IP du VPS
```

---

## Prérequis système

### Vérifier Docker

```bash
docker --version
# Docker version 24.x.x

docker compose version
# Docker Compose version v2.x.x
```

Si Docker n'est pas installé :

```bash
curl -fsSL https://get.docker.com | sh
```

### Vérifier que Traefik tourne

```bash
docker ps --filter name=traefik
```

Tu dois voir un conteneur Traefik avec le statut `Up`.

---

## Le réseau partagé

Traefik et n8n doivent partager le même réseau Docker pour que Traefik puisse router les requêtes vers n8n.

### Créer le réseau (si inexistant)

```bash
docker network create traefik_public
```

> Si le réseau existe déjà, cette commande retourne une erreur — c'est normal, tu peux l'ignorer.

### Vérifier que Traefik est attaché à ce réseau

```bash
docker network inspect traefik_public
```

Le conteneur Traefik doit apparaître dans la liste `Containers`.

---

## Configuration DNS

Ton domaine doit pointer vers l'IP publique du VPS **avant** de démarrer n8n. Let's Encrypt valide le domaine via HTTP pour émettre le certificat.

```bash
# Vérifier la résolution DNS
nslookup n8n.yourdomain.com
# ou
dig n8n.yourdomain.com +short
```

La réponse doit être l'IP de ton VPS.

---

## Schéma de communication

```
Internet
    │
    ▼
[Traefik :443]  ← termine le TLS, lit le Host header
    │
    │  réseau : traefik_public
    ▼
[n8n :5678]  ← reçoit le trafic déchiffré
    │
    ▼
[Volume n8n_data]  ← persistance des workflows et credentials
```

---

## Concepts clés

| Concept | Explication |
|---|---|
| **Réseau Docker externe** | Un réseau créé manuellement, partagé entre plusieurs stacks Docker Compose |
| **Traefik** | Reverse proxy qui lit les labels des conteneurs pour router automatiquement |
| **Let's Encrypt** | Autorité de certification gratuite • Traefik gère le renouvellement automatiquement |
| **DNS A record** | Entrée DNS qui associe un nom de domaine à une adresse IP |

---

## Prochaine étape

→ **Étape 02** : Écrire le `docker-compose.yml` — service n8n, volumes, réseau.
