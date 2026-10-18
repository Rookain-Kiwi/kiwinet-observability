# kiwinet-monitoring

Supervision interne de Kiwinet : métriques, journaux, tableaux de bord et alertes.
Prometheus, cAdvisor, Node Exporter, Loki, Grafana Alloy, Grafana.

Répond à : **« Pourquoi ça a planté, et que se passe-t-il en ce moment ? »**
Complète Uptime Kuma (dossier `uptime/`), qui répond à « mon service est-il en ligne ? ».

---

## Architecture

```
                 serveur x86 (Debian 13)
  ┌──────────────────────────────────────────────────────────────┐
  │  Node Exporter (réseau hôte) ──┐                             │
  │  cAdvisor ─────────────────────┼──▶ Prometheus ──┐           │
  │  Loki, Alloy, Grafana (métriques propres) ──────▶│           │
  │                                                  ▼           │
  │  Docker (via proxy de socket, lecture seule) ─┐            Grafana ──▶ Traefik ──▶ Internet
  │  Journal systemd de l'hôte (lecture seule) ───┴──▶ Alloy ──▶ Loki ──▶│   (HTTPS, auth)
  │                                                                │
  │                              alertes Grafana ──▶ webhook Discord
  └──────────────────────────────────────────────────────────────┘
```

| Réseau Docker  | Rôle | Membres |
|----------------|------|---------|
| `monitoring`   | interne à la supervision (pont `br-monitoring`) | tous, sauf Node Exporter |
| `proxy`        | exposition par Traefik | Grafana seulement |
| `db`           | PostgreSQL mutualisé | Grafana seulement |
| `socket-proxy` | proxy de socket Docker en lecture seule | Alloy seulement (avec Traefik) |
| `traefik-metrics` | métriques de Traefik (interne, créé par `kiwinet-services`) | Prometheus seulement (avec Traefik) |

Aucun port n'est publié par la pile. Seul Grafana est joignable depuis Internet, par Traefik.

---

## Composants

| Composant | Image (épinglée par empreinte dans le compose) | Rôle |
|-----------|-----------------------------------------------|------|
| Prometheus | `prom/prometheus` | stockage des métriques, 30 jours ou 10 Go |
| cAdvisor | `ghcr.io/google/cadvisor` | métriques par conteneur |
| Node Exporter | `prom/node-exporter` | métriques de l'hôte (CPU, mémoire, disques, réseau) |
| Loki | `grafana/loki` | stockage des journaux, 15 jours |
| Alloy | `grafana/alloy` | collecte des journaux (remplace Promtail, fin de vie) |
| Grafana | `grafana/grafana` | tableaux de bord, alertes ; base dans PostgreSQL |

Tous les services ont : image épinglée (`tag@sha256:<index multi-architecture>`), rotation des
journaux (10 Mo, 3 fichiers), limite de mémoire, `no-new-privileges`. Sauf mention contraire
ci-dessous : `cap_drop: ALL`, système de fichiers en lecture seule, utilisateur non root.

---

## Choix de sécurité et leur coût

**Node Exporter en réseau hôte.** La documentation officielle l'exige pour mesurer la machine
et non le conteneur (`--path.rootfs`, réseau hôte, PID hôte). Sans cela, les disques
`Libraries` et `Data` et le trafic réseau réels sont invisibles. Conséquence : il écoute sur
la machine elle-même, **hors des règles `DOCKER-USER`**. Seul `ufw` le filtre (refus par
défaut en entrée). Une seule règle l'ouvre, et seulement au pont `br-monitoring` :

```bash
sudo ufw allow in on br-monitoring to any port 9100 proto tcp comment 'Prometheus vers Node Exporter'
```

Les métriques exposées sont en lecture seule. Si cette règle manque, Prometheus signale la
cible injoignable (alerte `cible-injoignable`). Aucune règle ne l'ouvre au réseau local.

**cAdvisor est très privilégié par nature.** L'image officielle demande `privileged` et l'accès
au socket Docker. Risque assumé, limité : aucun port publié, réseau `monitoring` seulement.
Il ne peut donc pas être durci comme les autres (ni `cap_drop`, ni lecture seule).

**Alloy tourne sous l'utilisateur de son image (473), jamais en root**, sans aucune capacité, en
lecture seule, et **sans accès au socket Docker** : il lit les conteneurs par le proxy de
socket. Le journal systemd est monté en lecture seule, lisible grâce au groupe
`systemd-journal` ajouté au conteneur (GID de l'hôte, 999 par défaut : variable `JOURNAL_GID`
du `.env` si le numéro diffère, `getent group systemd-journal`). Les lignes de `sudo` sont écartées
avant l'envoi à Loki (elles contiennent les commandes saisies, arguments compris).

**Proxy de socket.** Il autorise la lecture des conteneurs et des réseaux (`NETWORKS=1`, requis par la découverte d'Alloy). Tout client de ce proxy peut lire
les variables d'environnement des conteneurs par l'inspection : c'est le cas de Traefik et
d'Alloy. Les secrets de cette pile passent donc par `.env`, jamais par un fichier versionné,
mais restent visibles par ces deux clients. Limite connue.

**Grafana exposé.** Limitation de débit, authentification de base Traefik, puis connexion
Grafana. Grafana OSS n'a pas de double authentification. Protection renforcée en attendant
la restriction au réseau de confiance (WireGuard). Pas d'inscription, pas d'accès anonyme,
ni télémétrie, ni installation automatique de plugins.

**Le réseau `monitoring` n'est pas `internal`** : ses conteneurs ont un accès sortant. Le
rendre `internal` empêcherait Prometheus de joindre Node Exporter par la passerelle de l'hôte.

---

## Alertes (Grafana vers Discord)

Fichiers versionnés dans `grafana/provisioning/alerting/`. Le webhook est un **secret** :
variable `DISCORD_WEBHOOK_URL` du `.env`, jamais dans Git.

| Règle | Condition | Durée |
|-------|-----------|-------|
| `disque-ssd-data` | SSD système ou Data utilisé à plus de 85 % | 15 min |
| `disque-libraries` | Libraries utilisé à plus de 97 % (presque plein par nature, à ajuster) | 30 min |
| `memoire-faible` | mémoire disponible sous 10 % | 10 min |
| `swap-sature` | swap utilisé à plus de 85 % | 15 min |
| `cible-injoignable` | une cible Prometheus ne répond plus (alerte aussi sans donnée) | 5 min |
| `conteneur-redemarre` | plus de 2 démarrages en 15 minutes | immédiat |

Seule `cible-injoignable` alerte en cas de donnée absente ou d'erreur de requête : si
Prometheus tombe, c'est elle seule qui prévient, les autres ne font pas de bruit en cascade.

Changer de cible un jour : modifier `policies.yml` (champ `receiver`) et le point de contact ;
les règles ne changent pas. Limite : si le serveur s'arrête, aucune alerte ne part. Une
vérification externe relève d'Uptime Kuma.

---

## Grafana provisionné par fichiers

| Fichier | Contenu |
|---------|---------|
| `grafana/provisioning/datasources/` | sources Prometheus et Loki (UID fixes) |
| `grafana/provisioning/dashboards/` | fournisseur : dossier « Kiwinet » |
| `grafana/provisioning/alerting/` | point de contact, politique, règles |
| `grafana/dashboards/` | tableaux de bord JSON |

Tableaux de bord, tels que publiés sur grafana.com :
Node Exporter Full (1860, révision 45), cAdvisor (19792, révision 6, labels de projet et de
service Compose conservés pour ses filtres), Traefik officiel (17346, révision 9, alimenté par
le job `traefik`). Les journaux se consultent par Explore, Loki.
`allowUiUpdates: false` : le dépôt fait foi, exporter en JSON pour modifier.

---

## Prérequis

- Réseaux externes existants : `proxy`, `db`, `socket-proxy`, `traefik-metrics`. Traefik et le proxy de socket actifs, ce dernier avec `NETWORKS=1`.
- Base `grafana` et rôle `grafana` sur l'instance PostgreSQL (mot de passe dans `.env`).
- Fichier `.env` créé depuis `env.example` (jamais versionné).
- Règle `ufw` de Node Exporter (ci-dessus).

---

## Déploiement

Le dépôt est cloné sur le serveur ; tout changement passe par commit, push, puis :

```bash
git pull --ff-only
docker compose config -q          # valide sans rien afficher
docker compose up -d <service>    # un composant à la fois : node-exporter, cadvisor,
                                  # prometheus, loki, alloy, grafana
```

Ne pas utiliser `docker compose config` sans `-q` : il affiche les secrets.

## Vérifications

- Prometheus, cibles : toutes à l'état `up` (une requête `up` dans Grafana, Explore).
- Loki, Explore : les journaux apparaissent avec les labels `container`, `service`, et `job="journal"`.
- Grafana : `curl --resolve` vers la route, réponse attendue `401` sans identifiants Traefik.
- Alertes : bouton « Test » du point de contact `discord-admin` dans Grafana.
- Après un redémarrage de la machine : tous les conteneurs reviennent seuls.

## Métriques de Traefik

Réseau dédié `traefik-metrics`, créé par `kiwinet-services` (`traefik/docker-compose.yml`) : il ne
contient que Traefik et Prometheus (un Traefik compromis n'atteint ni Loki ni cAdvisor). Le point
d'entrée `metrics` de Traefik est lié à son adresse fixe sur ce réseau : les autres conteneurs
exposés ne peuvent pas lire les métriques. Prometheus rejoint ce réseau (compose) et collecte le
job `traefik` (`prometheus/prometheus.yml`), qu'affiche le tableau Traefik de Grafana. Ordre de déploiement : Traefik d'abord, car le réseau
doit exister avant le démarrage de Prometheus.

## Mettre à jour une image

Choisir une version stable couvrant amd64, lire ses notes de version, puis relever
l'empreinte de l'**index multi-architecture** (celui du tag) : champ `digest` de
`hub.docker.com/v2/repositories/<organisation>/<image>/tags/<tag>`, ou l'entête
`Docker-Content-Digest` du registre pour GHCR. Une version qui corrige une faille de sécurité
l'emporte sur la règle des deux semaines.

## Limites connues

- Pas de healthcheck : l'outillage disponible dans chaque image n'est pas vérifié. Les services
  reviennent par `restart: unless-stopped` (Grafana attend PostgreSQL en redémarrant).
- Aucune vérification externe : à traiter avec Uptime Kuma.
- Journalisation de WebDAV : niveau par défaut, sans les requêtes ; à décider avant de la relever
  (chemins et noms d'utilisateur entreraient dans Loki).
