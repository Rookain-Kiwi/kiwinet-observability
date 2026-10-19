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

**nscd.** Kuma tente de lancer `nscd` (cache DNS embarqué) au démarrage avec `sudo service nscd
start`. La commande échoue ici, par construction : `sudo` a besoin d'élever ses privilèges, ce
qu'interdisent `no-new-privileges` et `cap_drop: ALL`. L'échec est sans effet (une ligne de
journal). Désactiver le réglage « NSCD » dans l'interface supprime le message ; il n'y a de toute
façon pas de cache DNS local.

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
- **Disque du VPS petit et sans alerte** : la supervision de `monitoring/` ne couvre que le
  serveur. Un disque plein affecterait tous les services du VPS. Une mise à jour d'image demande
  de télécharger la nouvelle version (environ 0,8 Go) tant que l'ancienne est présente : prévoir
  de la place ou retirer l'ancienne image avant.
- **Le vert de certains moniteurs ne prouve pas le service** (codes `401` de Traefik, moniteurs
  sur la machine de Kuma) : voir « Moniteurs ».
- **La clé d'API de l'intégration Home Assistant est sensible** : voir « Intégration Home
  Assistant ».
- **Mise à jour manuelle** : voir « Mettre à jour l'image ».

---

## Prérequis

- Réseau Docker `proxy` existant (créé par le Traefik du VPS).
- Traefik actif, avec les middlewares `rate-limit` et `secure-headers` (fichier dynamique de
  `kiwinet-services`), le point d'entrée `websecure` et le résolveur `letsencrypt`.
- Enregistrement DNS de `status.kiwinet.me` vers le VPS.
- Place disque : l'image slim fait environ 180 Mo compressée et environ 0,8 Go sur disque
  (mesuré au déploiement : 79 % puis 88 % d'occupation sur 9 Go). Le disque du VPS est petit et
  rien ne l'alerte (voir « Sécurité : limites connues »).

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

La liste vit dans la base de Kuma (aucun fichier à versionner). Moniteurs en place :

| Service | Type | Codes acceptés | Sur la page publique | Ce que le vert prouve |
|---------|------|----------------|----------------------|------------------------|
| Site principal | HTTP(S) | `200-299` | oui | Le site répond (même machine que Kuma, voir plus bas) |
| Jellyfin | HTTP(S) | `200-299` | oui | Le service répond (la redirection est suivie) |
| Home Assistant | HTTP(S), GET | `200-299` | oui | Le service répond |
| Music Assistant | HTTP(S) | `200-299` | oui | Le service répond |
| WebDAV | HTTP(S), chemin du service | `401` seulement | oui | Le service répond : le `401` vient du service lui-même (royaume propre à WebDAV), pas de Traefik |
| Kiwix | HTTP(S) | `401` seulement | oui (démonstration) | **La chaîne d'accès seulement** (voir ci-dessous) |
| Minecraft | GameDig | n/a | oui | Un serveur Minecraft répond à travers Traefik (même machine que Kuma) |
| pgAdmin, Grafana | HTTP(S) | `401` seulement | non | **La chaîne d'accès seulement** (voir ci-dessous) |
| Tableau de bord Traefik | HTTP(S) | `401` seulement | non | Traefik répond : le `401` est celui de Traefik, c'est bien lui que l'on teste |
| WireGuard Easy | HTTP(S) | `200-299` | non | La page de connexion répond ; rien sur l'état du tunnel VPN |
| Seedbox (quand elle sera déployée) | HTTP(S) | à définir | à définir | Exposée par Traefik |
| Pi-hole | Push | n/a | non | Non exposé ; signal envoyé depuis le serveur à la bascule du DNS du réseau local |

**Limites des codes `401`.** Quand le `401` vient de l'authentification de base de Traefik
(pgAdmin, Kiwix, Grafana), il est renvoyé avant que la requête n'atteigne le conteneur : un
conteneur tombé répondrait quand même `401`, et le moniteur resterait « en ligne ». Ces moniteurs
prouvent que la chaîne d'accès (DNS, box, Traefik) fonctionne, pas le service. Décision : accepter
cette limite ; Grafana alerte déjà sur les conteneurs du serveur. Aucun identifiant de Traefik
n'est donné à Kuma : ils seraient stockés dans une base exposée sur Internet et ouvriraient aussi
le tableau de bord de Traefik.

**Moniteurs sur la même machine que Kuma** (site principal, Minecraft). La connexion part du VPS
vers sa propre adresse publique ; d'après le fonctionnement du système (non testé ici), elle est
probablement rebouclée en interne sans passer par le pare-feu de Scaleway. Le vert signifie « le
service tourne et Traefik le relaie », pas « Internet l'atteint ». Si tout le VPS tombe, aucune
alerte ne part (limite déjà citée).

**Minecraft.** Le port de jeu est relayé par un routeur TCP de Traefik (`HostSNI(*)`, sans TLS)
vers le conteneur. Un moniteur TCP simple serait vert même avec le conteneur éteint (Traefik
accepte la connexion lui-même) : GameDig attend une vraie réponse de serveur Minecraft. Aucun
certificat n'est observable, d'où un capteur de certificat « Indisponible » dans Home Assistant.

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

## Page de statut publique

Identifiant `kiwinet` : `https://status.kiwinet.me/status/kiwinet`, l'adresse de l'ancienne page.

- **Affichés** : site principal, Jellyfin, Home Assistant, Music Assistant, WebDAV, Kiwix, Minecraft.
- **Non affichés** : pgAdmin, Grafana, tableau de bord Traefik, WireGuard Easy. Ils afficheraient à
  tous la liste des interfaces d'administration, et leur disponibilité en direct. Les moniteurs
  restent actifs pour les alertes privées.
- **Kiwix est affiché volontairement**, comme démonstration pour les visiteurs. Son vert ne prouve
  que la chaîne d'accès.
- **Adresses des services** : masquées (option d'envoi de l'URL désactivée pour chaque moniteur).
- Le pied de page indique l'hébergeur : choix assumé.

Contrôle fait à la mise en service, en lecture anonyme : les données publiques ne contiennent ni
adresse, ni port, ni message de battement ; les badges des moniteurs non publiés répondent « N/A » ;
`/metrics` répond `401`.

---

## Intégration Home Assistant

L'intégration officielle de Home Assistant (noyau, depuis 2025.8) demande l'adresse de l'instance et
une **clé d'API** (réglages de Kuma, « Clés d'API »), et interroge Kuma toutes les 30 secondes.

- La clé donne accès à `/metrics`, qui, d'après le fonctionnement de Kuma (non vérifié ici), décrit
  tous les moniteurs, privés compris. **C'est un secret sensible** : il ne vit que dans Home
  Assistant, jamais dans Git ni dans une conversation.
- Prévoir une date d'expiration à la création, et révoquer la clé si l'intégration est retirée.
- Un changement d'instance (nouvelle base) invalide les anciennes clés : créer une nouvelle clé et
  ré-authentifier l'intégration.
- Le capteur « Expiration du certificat » est indisponible pour Minecraft : normal (aucun TLS).

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
