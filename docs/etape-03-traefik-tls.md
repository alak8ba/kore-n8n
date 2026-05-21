# Étape 03 • Traefik & TLS · labels, entrypoints, Let's Encrypt

## Objectif

Comprendre comment Traefik découvre n8n via les labels Docker et lui assigne automatiquement un certificat TLS Let's Encrypt.

---

## Les labels Traefik dans docker-compose.yml

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.n8n.rule=Host(`${N8N_HOST}`)"
  - "traefik.http.routers.n8n.entrypoints=${TRAEFIK_ENTRYPOINT:-websecure}"
  - "traefik.http.routers.n8n.tls=true"
  - "traefik.http.routers.n8n.tls.certresolver=${TRAEFIK_CERT_RESOLVER:-letsencrypt}"
  - "traefik.http.services.n8n.loadbalancer.server.port=5678"
  - "traefik.docker.network=traefik_public"
```

---

## Explication label par label

| Label | Rôle |
|---|---|
| `traefik.enable=true` | Active la découverte de ce conteneur par Traefik |
| `routers.n8n.rule=Host(...)` | Traefik route les requêtes dont le `Host` header correspond à `N8N_HOST` |
| `routers.n8n.entrypoints=websecure` | Écoute sur l'entrypoint HTTPS de Traefik (port 443) |
| `routers.n8n.tls=true` | Active TLS sur ce router |
| `routers.n8n.tls.certresolver=letsencrypt` | Utilise le resolver Let's Encrypt pour obtenir le certificat |
| `services.n8n.loadbalancer.server.port=5678` | Indique à Traefik sur quel port interne n8n écoute |
| `traefik.docker.network=traefik_public` | Précise sur quel réseau Traefik doit joindre le conteneur |

---

## Pourquoi préciser le réseau ?

Si un conteneur est attaché à plusieurs réseaux Docker, Traefik ne sait pas lequel utiliser pour le joindre. Le label `traefik.docker.network` lève l'ambiguïté.

---

## Comment Traefik obtient le certificat

```
1. Tu démarre n8n (docker compose up -d)
2. Traefik détecte le conteneur via l'API Docker
3. Traefik lit les labels → crée un router pour n8n.yourdomain.com
4. Traefik contacte Let's Encrypt (ACME challenge HTTP-01 ou DNS-01)
5. Let's Encrypt valide que n8n.yourdomain.com pointe vers ce serveur
6. Traefik reçoit le certificat, le stocke (acme.json), le renouvelle automatiquement
```

> Le certificat est renouvelé automatiquement par Traefik avant expiration (tous les 90 jours).

---

## Variables à adapter à votre Traefik

Deux variables du `.env` dépendent de **votre configuration Traefik** :

### `TRAEFIK_ENTRYPOINT`

Le nom de l'entrypoint HTTPS dans votre config statique Traefik. Exemple typique :

```yaml
# traefik.yml (config statique de votre Traefik)
entryPoints:
  websecure:       ← c'est ce nom qu'on met dans TRAEFIK_ENTRYPOINT
    address: ":443"
```

### `TRAEFIK_CERT_RESOLVER`

Le nom du resolver dans votre config statique Traefik. Exemple typique :

```yaml
# traefik.yml
certificatesResolvers:
  letsencrypt:     ← c'est ce nom qu'on met dans TRAEFIK_CERT_RESOLVER
    acme:
      email: you@yourdomain.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web
```

---

## Schéma du routage

```
Client (HTTPS)
    │
    ▼
[Traefik :443]
    │  lit Host: n8n.yourdomain.com
    │  router: n8n → TLS ON → certresolver: letsencrypt
    │
    │  réseau traefik_public
    ▼
[n8n :5678]
```

---

## Concepts clés

| Concept | Explication |
|---|---|
| **Label Docker** | Métadonnée attachée à un conteneur, lue par Traefik à chaud |
| **Router Traefik** | Règle qui associe un critère (Host, Path) à un service et une config TLS |
| **Entrypoint** | Point d'entrée réseau de Traefik (ex : port 443 nommé `websecure`) |
| **Certificate resolver** | Configuration ACME pour obtenir un certificat Let's Encrypt |
| **ACME** | Protocole standard pour l'automatisation des certificats TLS |

---

## Prochaine étape

→ **Étape 04** : Gérer les variables d'environnement et les secrets · `.env`, clé de chiffrement.
