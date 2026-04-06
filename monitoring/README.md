# kiwinet-monitoring

Stack d'observabilité interne de `Kiwinet`  
Prometheus - cAdvisor - Node Exporter - Loki - Promtail - Grafana

---

## Rôle de ce repo

Ce repo déploie la **stack d'observabilité interne** : métriques système, métriques containers, et agrégation de logs. Il répond à la question : **Pourquoi ça a planté? Et qu'est-ce qui se passe en ce moment ?**

C'est la couche complémentaire à Uptime Kuma (qui surveille la disponibilité externe). Grafana est le tableau de bord unifié qui consomme à la fois les métriques Prometheus et les logs Loki.

```
VM hôte
    │
    ├── Node Exporter  ──────────────────────────────┐
    │   (CPU, RAM, disk, réseau)                     │
    │                                                ▼
    ├── cAdvisor  ──────────────────────────→  Prometheus
    │   (métriques par container Docker)             │
    │                                                │
    ├── Promtail  ──────────────────────────→  Loki  │
    │   (collecte logs Docker + système)             │
    │                                                ▼
    └────────────────────────────────────→  Grafana
                                            (dashboards + Explore)
                                                     │
                                                     ▼
                                         grafana.kiwinet.me (HTTPS, auth)
```

---

## Structure du repo

```
kiwinet-monitoring/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml          ← targets de scrape (cAdvisor, node-exporter)
├── loki/
│   └── loki.yml                ← config stockage et rétention Loki
└── promtail/
    └── promtail.yml            ← config collecte logs (Docker socket + /var/log)
```

---

## Architecture réseau

La stack utilise **deux réseaux Docker distincts** :

```
Réseau monitoring (bridge interne - isolé) :
    prometheus  ←  cadvisor
    prometheus  ←  node-exporter
    loki        ←  promtail
    grafana     →  prometheus
    grafana     →  loki

Réseau proxy (bridge externe - partagé avec Traefik) :
    grafana  →  grafana.kiwinet.me  (HTTPS, SSL Let's Encrypt, auth-basic)
```

Seul Grafana est exposé publiquement. Prometheus, Loki, cAdvisor et Node Exporter ne sont pas accessibles depuis l'extérieur.

---

## Composants

### Prometheus

Collecte et stocke les métriques en séries temporelles. Configure via `prometheus/prometheus.yml`.

- Scrape interval : 15 secondes
- Rétention : 15 jours (`--storage.tsdb.retention.time=15d`)
- Sources : cAdvisor (containers) + Node Exporter (système)

### cAdvisor

Collecte les métriques Docker en temps réel : CPU, mémoire, réseau, I/O par container. Tourne en mode `privileged` avec accès aux pseudo-systèmes de fichiers `/proc`, `/sys`, `/var/lib/docker`.

### Node Exporter

Collecte les métriques système de la VM hôte : CPU global, RAM, utilisation disque, trafic réseau. Tourne en mode `pid: host` pour accéder aux processus de la machine hôte.

### Loki

Agrège et indexe les logs. Reçoit les entrées de Promtail. Accessible via Grafana (Explore → Loki).

**Point critique - permissions Loki**  
Le container doit tourner avec `user: "0"` et monter son volume sur `/loki` (pas `/tmp/loki`) pour éviter les erreurs de permission à l'initialisation.
```yaml
loki:
  user: "0"
  volumes:
    - loki_data:/loki
```

### Promtail

Agent de collecte de logs. Lit le Docker socket pour récupérer les logs de tous les containers, et remonte les logs système depuis `/var/log` et pousse vers Loki.

### Grafana

Interface de visualisation unifiée. Consomme Prometheus (métriques) et Loki (logs).

- Exposé sur `grafana.kiwinet.me` via Traefik
- Protégé par `auth-basic@file` (middleware Traefik)
- Certificat SSL Let's Encrypt géré automatiquement par Traefik

---

## Dashboards importés

| Dashboard                       | ID Grafana | Data source | Contenu                        |
|---------------------------------|------------|-------------|--------------------------------|
| Node Exporter Full              | `1860`     | Prometheus  | CPU, RAM, disk, réseau VM      |
| cAdvisor métriques containers   | `19792`    | Prometheus  | Métriques par container Docker |

**Logs :** consultés via l'explorateur dans Grafana. Les dashboards communautaires Loki (13639, 15141) présentent des variables orphelines (`DS_LOKI`) et ne sont plus maintenus - Explore reste donc l'interface recommandée.

---

## Déploiement

Ce repo est cloné sur la VM dans `/opt/kiwinet-monitoring/`. Le workflow est entièrement manuel (pas de CI/CD pour l'instant sur ce repo).

**Prérequis :**
- Réseau Docker `proxy` existant (`docker network create proxy`)
- DNS A `grafana.kiwinet.me` → `<IP_PUBLIQUE>`
- Traefik actif sur la VM

```bash
# Sur la machine locale
git push origin main

# Sur la VM
cd /opt/kiwinet-monitoring
git pull
docker compose up -d
```

---

## Vérifications post-déploiement

```bash
# Tous les containers up ?
docker compose ps

# Prometheus scrape les targets ?
# → Grafana : Connections > Data sources > Prometheus > Save & test

# Loki reçoit des logs ?
# → Grafana : Explore → Loki → sélectionner un label → Run query

# Grafana accessible ?
curl -I https://grafana.kiwinet.me
# Attendu : HTTP 302 (redirect login) ou HTTP 200
```

---

## Infrastructure cible

| Composant    | Détail                              |
|--------------|-------------------------------------|
| OS           | Debian GNU/Linux 13.3 (Trixie)      |
| Architecture | ARM AArch64 (Freebox Delta)         |
| RAM estimée  | ~500 Mo (stack complète)            |

### Estimation RAM par service

| Service                        | RAM estimée |
|--------------------------------|-------------|
| Prometheus                     | ~100 Mo     |
| cAdvisor                       | ~50 Mo      |
| Node Exporter                  | ~10 Mo      |
| Loki                           | ~100 Mo     |
| Promtail                       | ~50 Mo      |
| Grafana                        | ~150 Mo     |
| **Total**                      | ~460 Mo     |

Marge confortable sur 12 Go avec Minecraft, Plex et la stack Traefik/web actifs.