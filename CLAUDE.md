# CLAUDE.md — kiwinet-monitoring
# Instructions persistantes pour Claude Code sur ce repo.
# Ce fichier est lu automatiquement à chaque session.

---

## 🎯 Contexte du projet

Portfolio DevOps auto-hébergé — domaine `kiwinet.me`, infrastructure ARM AArch64 sur Freebox Delta.
Repo : `kiwinet-monitoring` — stack d'observabilité complète :
Prometheus (métriques) + cAdvisor + Node Exporter + Loki (logs) + Promtail + Grafana (`grafana.kiwinet.me`).

Déploiement : `git pull` manuel sur la VM — pas de CI/CD sur ce repo.

---

## 📐 Règles de commentaire (style kiwinet)

Ces règles s'appliquent à tous les fichiers du repo.
Le but est d'expliquer le **pourquoi**, pas seulement le **quoi**.
Langue : **français**.

---

### Règle 1 — En-tête de fichier obligatoire

```yaml
# nom-du-fichier.yml : Titre court et rôle
# Note de comportement si pertinent (ex: rechargé à chaud, scrape toutes les 15s...)
```

---

### Règle 2 — Séparateurs de sections

```yaml
# ─────────────────────────────────────────────────────────────────────────────
# NOM DE LA SECTION
# ─────────────────────────────────────────────────────────────────────────────
```

---

### Règle 3 — Commentaires inline

Sur la même ligne que la directive, après deux espaces, pour expliquer le *pourquoi*.

```yaml
scrape_interval: 15s   # Fréquence de collecte des métriques — compromis charge/précision
retention_period: 744h   # 31 jours de rétention des logs
```

---

### Règle 4 — Blocs explicatifs pour les flux complexes

```yaml
# ─────────────────────────────────────────────────────────────────────────────
# NOM DU FLUX
#
# Description en prose :
#   1. Étape 1
#   2. Étape 2
#
# Prérequis : conditions nécessaires
# ─────────────────────────────────────────────────────────────────────────────
```

---

## 🚨 Zones interdites aux commentaires — règles absolues

### Blocs multilignes YAML (`|` ou `>`)

Tout contenu d'un bloc multiligne est traité comme **données brutes**.
Le `#` n'y est PAS un commentaire — il est transmis tel quel à l'outil qui lit la valeur.

```yaml
# ❌ INTERDIT
pipeline_stages:
  - regex:
      # ce commentaire sera interprété comme une directive Promtail invalide
      expression: '...'

# ✅ CORRECT — commentaire avant le bloc ou la clé
# Extrait le niveau de log depuis le message brut
pipeline_stages:
  - regex:
      expression: '...'
```

### Valeurs scalaires simples

Pas de commentaire inline sur une valeur lue par un outil externe (image, port, job_name...).

```yaml
# ❌ INTERDIT
image: grafana/grafana:latest   # Dashboard de visualisation

# ✅ CORRECT
# Dashboard de visualisation — interface principale de monitoring
image: grafana/grafana:latest
```

---

## 🚀 Commande batch — commenter tous les fichiers

```
Récupère la liste des fichiers versionnés avec `git ls-files`, puis filtre ceux avec
les extensions .yml et .yaml.

Pour chaque fichier retenu :
  1. Analyse son rôle dans le projet.
  2. Applique les règles de commentaire définies dans CLAUDE.md.
  3. Avant d'écrire un commentaire dans un fichier YAML, vérifie si la ligne cible
     est à l'intérieur d'un bloc multiline (introduit par | ou >). Si oui, place
     le commentaire avant le bloc — jamais à l'intérieur.
  4. Modifie le fichier en place — ne supprime aucun code existant, ajoute uniquement
     les commentaires manquants.
  5. Si le fichier est déjà conforme aux règles, laisse-le tel quel et indique-le.

Traite les fichiers un par un et affiche le nom de chaque fichier au fur et à mesure.
```

---

## ⚠️ Ce que Claude NE doit PAS faire

- Supprimer ou modifier du code fonctionnel
- Traduire des termes techniques en français (`scrape_interval`, `pipeline_stages`, `job_name` restent en anglais)
- Ajouter des commentaires évidents qui répètent le code
- Modifier les fichiers sensibles sans instruction explicite
- Placer un commentaire à l'intérieur d'un bloc multiline YAML (`|` ou `>`)
