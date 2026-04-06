# kiwinet-status

Page de statut publique et surveillance de disponibilité pour `Kiwinet`  
Stack : Uptime Kuma - Let's Encrypt via Traefik - Discord webhook

---

## Rôle de ce repo

Ce repo déploie **Uptime Kuma**, l'outil de surveillance externe de l'infrastructure. Il répond à une question simple : **Mes services sont-il en ligne en ce moment ?**

C'est la couche de monitoring *orientée disponibilité*, distincte de la stack Prometheus/Grafana qui s'occupe des métriques internes et des logs.

```
                    ┌──────────────────────────────────┐
                    │         Monitoring               │
                    │                                  │
  Question externe  │  Uptime Kuma  ← kiwinet-status   │
  "Est-ce en ligne?"│      │                           │
                    │      ▼                           │
                    │  status.kiwinet.me (public)      │
                    └──────────────────────────────────┘

                    ┌──────────────────────────────────┐
                    │         Observabilité interne    │
  Question interne  │  Prometheus + Loki + Grafana     │
  "Pourquoi ça a    │  ← kiwinet-monitoring            │
   planté ?"        │      │                           │
                    │      ▼                           │
                    │  grafana.kiwinet.me (privé)      │
                    └──────────────────────────────────┘
```

---

## Structure du repo

```
kiwinet-status/
├── docker-compose.yml      ← déploiement Uptime Kuma
├── .gitignore              ← exclut data/
└── data/                   ← base SQLite Uptime Kuma (non commitée)
```

Le répertoire `data/` contient la base SQLite avec la configuration des monitors, les identifiants et l'historique. Il n'est **jamais commité**.

---

## Déploiement

Ce repo est cloné sur la VM dans `/opt/kiwinet-status/`. Le workflow est entièrement manuel pour le status (pas de CI/CD pour l'instant sur ce repo).

**Prérequis :**
- Réseau Docker `proxy` existant (`docker network create proxy`)
- DNS A `status.kiwinet.me` → `IP Fixe`
- Traefik actif sur la VM

```bash
# Sur la machine locale
git push origin main

# Sur la VM
cd /opt/kiwinet-status
git pull
docker compose up -d
```

Uptime Kuma sera accessible à `https://status.kiwinet.me` avec certificat Let's Encrypt géré automatiquement par Traefik.

---

## Intégration Traefik

Uptime Kuma s'intègre à Traefik via labels Docker dans `docker-compose.yml`. Pas de configuration dans `dynamic.yml` - le container est découvert automatiquement.

```
Internet → Traefik (:443) → container uptime-kuma (:3001)
```

Middleware appliqué : `secure-headers@file` (headers de sécurité HTTP).  
Pas d'`auth-basic` - la page de statut est publique par conception.

---

## Monitors configurés

| Monitor            | Type     | Cible                                | Note                    
|--------------------|----------|--------------------------------------|-------------------------
| Routeur            | HTTP(s)  | `https://freebox.kiwinet.me`         | Port admin non publié   
| Site Principal     | HTTP(s)  | `https://kiwinet.me`                 |                         
| Traefik            | HTTP(s)  | `https://traefik.kiwinet.me`         | HTTP 401 accepté        
| Plex               | HTTP(s)  | `https://plex.kiwinet.me`            | HTTP 401 accepté        
| Status             | HTTP(s)  | `https://status.kiwinet.me`          |                         
| Grafana            | HTTP(s)  | `https://grafana.kiwinet.me`         |                         
| Home Assistant     | HTTP(s)  | `https://hub.kiwinet.me`             |                         
| Minecraft          | TCP Port | `minecraft.kiwinet.me:25565`         | TCP brut, hors Traefik  

Les monitors Traefik et Plex retournent un HTTP 401 (authentification requise). Uptime Kuma est configuré pour accepter ce code comme réponse valide - le service est *up*, mais protégé.

---

## Pages et alertes

**Page de statut publique :** `https://status.kiwinet.me/status/kiwinet`

**Alertes Discord :**  
Webhook configuré sur `#monitoring-kiwinet` - salon privé dédié, accessible uniquement par l'administrateur. Notifications envoyées à chaque changement d'état (down / recovery).

---

## Philosophie de nommage

L'URL `status.kiwinet.me` nomme une **fonction** (consulter le statut), pas un outil.  
Cela contraste avec `grafana.kiwinet.me` qui révèle la stack - acceptable car privé et protégé par authentification.

---

## Infrastructure cible

| Composant    | Détail                                         
|--------------|------------------------------------------------
| OS           | Debian GNU/Linux 13.3 (Trixie)                 
| Architecture | ARM AArch64 (Freebox Delta)                    
| Image Docker | `louislam/uptime-kuma:1`                       
| Réseau       | `proxy` (bridge externe, partagé avec Traefik) 
