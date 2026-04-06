# kiwinet-observability

Stack d'observabilité complète de Kiwinet — disponibilité, métriques, logs.

> Contexte global et justification de la fusion : [kiwinet-docs](https://github.com/Rookain-Kiwi/kiwinet-docs) — ADR-001

---

## Prérequis

- Réseau Docker `proxy` existant (`docker network create proxy`)
- Traefik actif sur la VM

---

## Structure

```
kiwinet-observability/
├── uptime/
│   ├── docker-compose.yml      # Uptime Kuma
│   └── data/                   # Base SQLite (gitignored)
└── monitoring/
    ├── docker-compose.yml      # Prometheus, cAdvisor, Node Exporter, Loki, Promtail, Grafana
    ├── prometheus/
    │   └── prometheus.yml
    ├── loki/
    │   └── loki.yml
    └── promtail/
        └── promtail.yml
```

---

## Déploiement

```bash
cd /opt/kiwinet-observability

# Uptime Kuma
cd uptime && docker compose up -d

# Stack métriques + logs
cd monitoring && docker compose up -d
```

Mise à jour :
```bash
git pull
cd uptime    && docker compose up -d --force-recreate
cd monitoring && docker compose up -d --force-recreate
```

---

## Uptime Kuma

Surveillance de disponibilité externe. Répond à : **"Mon service est-il en ligne ?"**

- URL : `https://status.kiwinet.me`
- Page publique : `https://status.kiwinet.me/status/kiwinet`
- Alertes : Discord webhook (salon privé administrateur)

**Monitors configurés :**

| Monitor                  | Type     | Cible                                      |
|--------------------------|----------|--------------------------------------------|
| Routeur                  | HTTP(s)  | `https://freebox.kiwinet.me`               |
| Site principal           | HTTP(s)  | `https://kiwinet.me`                       |
| Traefik                  | HTTP(s)  | `https://traefik.kiwinet.me` (401 accepté) |
| Plex                     | HTTP(s)  | `https://plex.kiwinet.me` (401 accepté)    |
| Home Assistant           | HTTP(s)  | `https://hub.kiwinet.me`                   |
| Status                   | HTTP(s)  | `https://status.kiwinet.me`                |
| Grafana                  | HTTP(s)  | `https://grafana.kiwinet.me`               |
| Minecraft                | TCP Port | `minecraft.kiwinet.me:25565`               |

---

## Stack métriques + logs

Observabilité interne. Répond à : **"Pourquoi ça a planté ?"**

```
Node Exporter  ──→ Prometheus ──→ Grafana (grafana.kiwinet.me)
cAdvisor       ──→ Prometheus
Promtail       ──→ Loki       ──→ Grafana (Explore)
```

**Réseaux Docker :**
- `monitoring` (interne) : Prometheus, cAdvisor, Node Exporter, Loki, Promtail
- `proxy` (externe) : Grafana uniquement — seul composant exposé publiquement

**Dashboards importés :**

| Dashboard                     | ID Grafana | Data source |
|-------------------------------|------------|-------------|
| Node Exporter Full            | 1860       | Prometheus  |
| cAdvisor métriques containers | 19792      | Prometheus  |

Logs : **Explore → Loki** (dashboards communautaires Loki non maintenus — variables orphelines).

---

## Points critiques

**Permissions Loki :**
```yaml
loki:
  user: "0"
  volumes:
    - loki_data:/loki   # Pas /tmp/loki
```

**cAdvisor ARM64** — flags valides pour `--disable_metrics` :
```
cpu_topology,cpuset,memory_numa,process,referenced_memory,resctrl,sched,tcp,udp
```
(`accelerator` n'existe pas sur cette version — provoque une erreur au démarrage)

**Intervalle cAdvisor :** `--housekeeping_interval=30s` (défaut 1s — source principale de charge CPU)

---

## Vérifications post-déploiement

```bash
# Containers actifs
docker compose ps

# Prometheus scrape les targets
# → Grafana : Connections > Data sources > Prometheus > Save & test

# Loki reçoit des logs
# → Grafana : Explore → Loki → sélectionner un label → Run query

# Grafana accessible
curl -I https://grafana.kiwinet.me
```
