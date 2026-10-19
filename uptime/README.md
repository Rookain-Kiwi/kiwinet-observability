# Uptime Kuma : page de statut et surveillance de disponibilité

Page de statut publique de Kiwinet et alertes de disponibilité vers Discord.
Uptime Kuma 2, déployé sur le **VPS Scaleway**, derrière Traefik.

Répond à : **« Mon service est-il en ligne ? »**
Complète la supervision interne (dossier `monitoring/`), qui répond à « Pourquoi ça a planté ? ».

---

## Pourquoi sur le VPS

Uptime Kuma tourne hors du serveur qu'il surveille. Si le serveur ou la connexion Internet de la
maison tombe, l'alerte part quand même et la page de statut reste en ligne. Une seule instance
suffit : une seconde doublerait les sauvegardes, les webhooks et les mises à jour sans gain.

```
        Internet
           │
           ▼
   ┌───────────────────────────── VPS ──────────────┐
   │  Traefik ──▶ Uptime Kuma ──┬──▶ services publics du serveur (HTTPS)
   │              (réseau proxy)├──▶ Minecraft, site (VPS)
   │                            └──▶ alertes ──▶ webhook Discord
   └────────────────────────────────────────────────┘
        ▲
        └── signaux « push » du serveur (services du réseau local, voir plus bas)
```

Décisions liées : séparation statut public et observabilité interne, page publique sans
authentification, alertes Discord uniquement. Détail dans « Décisions (Stack) », second brain Notion.

---

## Ce que ce dossier déploie

| Élément | Choix |
|---------|-------|
| Image | `louislam/uptime-kuma`, version `2.5.6`, variante `slim-rootless`, épinglée par l'empreinte de l'index multi-architecture |
| Base | SQLite, dans le volume nommé `uptime-kuma-data` (SSD système du VPS) |
| Réseau | `proxy` seulement ; aucun port publié |
| Route Traefik | `rate-limit` puis `secure-headers` ; pas d'`auth-basic` (page publique) |
| Durcissement | `no-new-privileges`, `cap_drop: ALL`, système de fichiers en lecture seule (+ `tmpfs` sur `/tmp`), utilisateur non root, limite de mémoire, rotation des journaux (10 Mo, 3 fichiers) |

**Slim et rootless.** La variante slim n'embarque ni Chromium ni MariaDB : seuls le type de
moniteur « Browser Engine » et la base MariaDB embarquée sont concernés (la documentation
officielle, page « Docker Tags », ne liste pas d'autre différence). Un moniteur « Browser Engine »
imposerait la variante complète. La variante rootless fait tourner Kuma sous l'utilisateur `node`
de l'image ; le moniteur de type Docker n'y fonctionne pas, ce qui est sans effet ici (aucun socket
Docker n'est monté).

**Volume nommé.** Un volume neuf recopie le propriétaire du dossier préparé par l'image (`node`) :
aucun `chown`, aucune ligne `user:`. Un dossier lié au dépôt aurait appartenu à root et provoqué
« permission denied » avec `cap_drop: ALL`.

Aucun secret dans ce dossier : moniteurs, page de statut, webhook Discord et jetons de push
sont dans la base de Kuma. Pas de `.env`.

---

## Sécurité : limites connues

- **La connexion à l'administration est servie sur la même adresse publique que la page de
  statut.** Protections : mot de passe long, **double authentification TOTP** (à activer dès la
  création du compte), limiteurs internes de Kuma.
- **Les tentatives de connexion passent par une WebSocket** : la limitation de débit de Traefik
  ne les voit pas une à une. Les freins sont ceux de Kuma (code de la 2.5.6, `server/rate-limiter.js`) :
  20 connexions par minute et 30 vérifications TOTP par minute, **compteurs globaux, non par
  adresse**. Conséquence : un tiers peut verrouiller l'accès à l'administration en envoyant de
  mauvaises tentatives (la surveillance et la page de statut continuent). Remède prévu : réserver
  l'administration au réseau de confiance (WireGuard), comme pour les autres interfaces.
- **Création du compte à l'installation.** Tant qu'aucun compte n'existe, le premier visiteur peut
  le créer : le faire immédiatement après le premier démarrage (voir « Premier démarrage »).
- **Les URL de push sont des secrets** : le jeton qu'elles contiennent est leur seule protection.
  Elles ne vont jamais dans Git.
- **Le VPS lui-même n'est surveillé par personne** : s'il tombe, aucune alerte ne part et la page
  de statut disparaît. Limite assumée.
- **Pas de sauvegarde du volume pour l'instant** : moniteurs, page de statut et webhook seraient à
  ressaisir en cas de perte.
- **Mise à jour manuelle** : voir « Mettre à jour l'image ».

---

## Prérequis

- Réseau Docker `proxy` existant (créé par le Traefik du VPS).
- Traefik actif, avec les middlewares `rate-limit` et `secure-headers` (fichier dynamique de
  `kiwinet-services`), le point d'entrée `websecure` et le résolveur `letsencrypt`.
- Enregistrement DNS de `status.kiwinet.me` vers le VPS.
- Place disque : l'image slim fait environ 180 Mo compressée ; le disque du VPS est petit, à
  surveiller.

---

## Déploiement

Le dépôt est cloné sur le VPS, en HTTPS, sans clé (dépôt public, lecture seule). Aucune
modification locale sur le serveur : tout passe par commit, push, puis :

```bash
git pull --ff-only
docker compose config -q          # valide sans rien afficher
docker compose up -d
docker compose logs --tail 50     # contrôle du démarrage
```

Ne pas utiliser `docker compose config` sans `-q` : il peut afficher des secrets.

### Premier démarrage

1. Choisir **SQLite** à l'écran de choix de la base.
2. Créer immédiatement le compte administrateur (mot de passe long).
3. Activer la **double authentification TOTP** dans les réglages du compte.
4. Régler la durée de conservation de l'historique (disque limité).
5. Créer le point de contact Discord (webhook propre à Uptime Kuma, distinct de celui de Grafana).

### Vérifications

- `docker ps` : le conteneur passe à « healthy » (délai de démarrage de 180 s).
- La page de statut répond en HTTPS avec un certificat valide ; l'administration se charge
  (WebSocket comprise).
- Un moniteur volontairement faux déclenche une alerte Discord, puis le retour à la normale.
- Si le système de fichiers en lecture seule provoque une erreur d'écriture, retirer `read_only`
  et noter la raison ici.

---

## Moniteurs

La liste vit dans la base de Kuma (aucun fichier à versionner). Cibles prévues :

| Service | Type | Note |
|---------|------|------|
| Jellyfin, Music Assistant, Kiwix, pgAdmin, Grafana, tableau de bord Traefik, WebDAV, Home Assistant, site principal | HTTP(S) | Un code `401` doit être décidé moniteur par moniteur (voir ci-dessous) |
| Minecraft | TCP | Port brut, hors Traefik |
| Seedbox (quand elle sera déployée) | HTTP(S) | Exposée par Traefik |
| Pi-hole | Push | Non exposé ; signal envoyé depuis le serveur à la bascule du DNS du réseau local |

**Le code `401` n'a pas le même sens partout.** Quand il vient de l'authentification de base de
Traefik, il répond même si le conteneur derrière est tombé : le moniteur resterait « en ligne » à
tort. Quand il vient du service lui-même (WebDAV), il prouve que le service répond. À trancher
et à tester (arrêt d'un conteneur) pour chaque cible.

Réglages recommandés :

| Paramètre | Valeur | Raison |
|-----------|--------|--------|
| Essais | 2 | Tolère un accroc ponctuel (le défaut des nouveaux moniteurs en v2 est 0) |
| Timeout | 10 s | Détecte une vraie panne sans faux positif |
| Méthode HTTP | GET | Home Assistant rejette HEAD avec un 405 |

Un moniteur de l'instance sur elle-même n'est pas créé : si Kuma tombe, il ne peut pas l'indiquer.

**Pas de directive `dns:`** : le VPS résout les noms publics normalement. Si de fausses alertes
`ENOTFOUND` apparaissent, revoir ce point.

---

## Alertes

Webhook Discord propre à Uptime Kuma (révocation indépendante de celui de Grafana, source
identifiable). Notification à chaque changement d'état (panne, retour). La page de statut
publique est servie par `status.kiwinet.me`, sans authentification.

---

## Mettre à jour l'image

Même méthode que `monitoring/README.md` (section « Mettre à jour une image ») : version stable
couvrant amd64, lecture des notes de version, puis relève de l'empreinte de l'**index
multi-architecture** (champ `digest` de
`hub.docker.com/v2/repositories/louislam/uptime-kuma/tags/<tag>`). Garder la variante
`slim-rootless`. Après la mise à jour : `docker compose pull`, `docker compose up -d`, contrôle du
statut « healthy ». Lire les notes de version avant chaque mise à jour (une version peut modifier
la base) : la copie du volume est la seule sortie en cas de problème, il n'y a pas de retour arrière
automatique.
