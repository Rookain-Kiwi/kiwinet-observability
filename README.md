# kiwinet-observability

Supervision de Kiwinet : **disponibilité** (page de statut publique, sur le VPS) et
**observabilité interne** (métriques, journaux, alertes, sur le serveur).

> Contexte global et justification de la fusion des deux dépôts : [kiwinet-docs](https://github.com/Rookain-Kiwi/kiwinet-docs), ADR-001.

---

## Deux questions, deux piles

| Dossier | Répond à | Outils | Machine | README |
|---------|----------|--------|---------|--------|
| [`uptime/`](uptime/) | « Mon service est-il en ligne ? » | Uptime Kuma, alertes Discord | VPS Scaleway | [uptime/README.md](uptime/README.md) |
| [`monitoring/`](monitoring/) | « Pourquoi ça a planté, et que se passe-t-il en ce moment ? » | Prometheus, cAdvisor, Node Exporter, Loki, Alloy, Grafana | Serveur x86 (Debian 13) | [monitoring/README.md](monitoring/README.md) |

```
        Internet
           │
           ▼
   VPS ── Uptime Kuma ───────▶ vérifie de l'extérieur les services de Kiwinet
           │                    ├─▶ page de statut publique
           │                    └─▶ alertes ──▶ Discord (webhook propre à Kuma)
           │
   Serveur ── Prometheus + Loki ──▶ Grafana ──▶ alertes ──▶ Discord (webhook propre à Grafana)
              (cAdvisor, Node Exporter, Alloy)
```

Les deux piles sont indépendantes : chacune a son réseau, son webhook Discord (révocation
indépendante, source identifiable) et son cycle de mise à jour. Uptime Kuma tourne hors du
serveur qu'il surveille : si le serveur ou la connexion de la maison tombe, l'alerte part quand
même et la page de statut reste en ligne.

---

## Structure

```
kiwinet-observability/
├── README.md
├── .gitignore
├── monitoring/                         # Observabilité interne (serveur x86)
│   ├── README.md
│   ├── .gitignore
│   ├── docker-compose.yml              # Prometheus, cAdvisor, Node Exporter, Loki, Alloy, Grafana
│   ├── env.example                     # modèle du .env (jamais versionné)
│   ├── prometheus/prometheus.yml
│   ├── loki/loki.yml
│   ├── alloy/config.alloy
│   └── grafana/
│       ├── dashboards/                 # tableaux de bord JSON
│       └── provisioning/               # alerting/, dashboards/, datasources/
└── uptime/                             # Disponibilité et page de statut (VPS)
    ├── README.md
    └── docker-compose.yml              # Uptime Kuma (aucune donnée dans le dépôt)
```

---

## Prérequis

Chaque pile a les siens, détaillés dans son README. En résumé :

| | Réseaux Docker externes | Autres |
|---|---|---|
| `uptime/` | `proxy` | Traefik actif avec les middlewares `rate-limit` et `secure-headers` ; enregistrement DNS de la page de statut |
| `monitoring/` | `proxy`, `db`, `socket-proxy`, `traefik-metrics` | Traefik et proxy de socket Docker actifs ; base `grafana` sur l'instance PostgreSQL ; fichier `.env` créé depuis `env.example` ; règle `ufw` pour Node Exporter |

Les réseaux externes sont créés hors de ce dépôt (par `kiwinet-services`) et **doivent exister
avant le démarrage** : leurs noms ne se devinent pas.

---

## Déploiement

Le dépôt est cloné **sur chaque machine, en HTTPS et sans clé** (dépôt public, lecture seule).
Aucune modification locale sur les machines : tout passe par modification sur le poste de
travail, commit, push, puis, dans le dossier de la pile concernée :

```bash
git pull --ff-only
docker compose config -q          # valide sans rien afficher
docker compose up -d <service>    # un service à la fois, avec contrôle entre chaque
```

- Ne jamais utiliser `docker compose config` sans `-q` : il affiche les secrets.
- Pas de `--force-recreate` systématique : on recrée uniquement le service modifié.
- Détail de l'ordre et des vérifications : README de chaque dossier. Les mises à jour d'images
  sont manuelles (voir « Mettre à jour une image » dans `monitoring/README.md`).

---

## Conventions communes

| Règle | Application |
|-------|-------------|
| Images épinglées par l'empreinte de l'**index multi-architecture** | `tag@sha256:…` dans les deux piles |
| Élévation de privilèges interdite, aucune capacité | `no-new-privileges`, `cap_drop: ALL` ; seule exception justifiée : cAdvisor (voir `monitoring/README.md`) |
| Système de fichiers en lecture seule, utilisateur non root | quand l'image le permet, à valider au démarrage réel |
| Limite de mémoire, rotation des journaux (10 Mo, 3 fichiers) | tous les services |
| Volumes sur le SSD système | volumes nommés, jamais de données dans le clone Git |
| Secrets hors Git | `.env` ignoré (`monitoring/`) ; moniteurs, webhook et jetons de push de Kuma dans sa base |
| Refus de démarrer sans variable obligatoire | `${VARIABLE:?message}` |
| Aucune adresse IP ni plage locale dans ce dépôt public | |

---

## Limites connues communes

- **Le VPS n'est surveillé par personne** : Uptime Kuma surveille le serveur depuis le VPS, mais
  si le VPS tombe, aucune alerte ne part et la page de statut disparaît. Son disque est petit et
  sans alerte.
- **Les moniteurs d'Uptime Kuma ne prouvent pas tous le service** : voir « Moniteurs » dans
  `uptime/README.md`.
- Autres limites (exposition de Node Exporter, cAdvisor privilégié, proxy de socket…) : voir
  `monitoring/README.md`, section « Choix de sécurité et leur coût ».
