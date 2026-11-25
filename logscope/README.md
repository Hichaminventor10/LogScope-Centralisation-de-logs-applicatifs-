# LogScope — Centralisation de logs applicatifs (Sujet 16)

Outil qui rassemble les logs de plusieurs applications et signale les pics d'erreurs.

| Brique | Rôle |
|---|---|
| **Cassandra** | Stockage des logs (+ compteurs horaires pour le graphe de volume) |
| **Redis** | Compteur d'erreurs glissant sur les **5 dernières minutes** (+ logs récents, activité) |
| **MongoDB** | Applications surveillées, **seuils d'alerte** et alertes ouvertes/résolues |
| **Elasticsearch** *(optionnel)* | Recherche plein texte (fonctionnalité avancée) |
| **FastAPI** | API REST de consultation, d'ingestion et de gestion des seuils |
| **React + Vite** | Interface de supervision (Vue d'ensemble, Applications, Journaux, Alertes, Recherche, Configuration) |
| **Docker Compose** | Orchestration de toute la pile |

## Démarrage rapide

```bash
# 1. Pile complète (Cassandra, Redis, MongoDB, API, UI, générateur de logs)
docker compose up --build

# 2. Ouvrir l'interface
#    → http://localhost:8080   (API : http://localhost:8000/docs)
```

Le conteneur **generator** envoie en continu des logs (INFO/WARN/ERROR) et déclenche
périodiquement un pic d'erreurs : l'alerte de seuil apparaît alors sur le tableau de bord.

### Avec Elasticsearch (recherche plein texte)

```bash
docker compose --profile advanced up --build
```

L'API détecte Elasticsearch au démarrage ; s'il est absent, la route `/api/search`
retombe automatiquement sur un filtrage Cassandra (l'UI l'indique via la pastille de source).

## Développement local (sans Docker pour l'UI)

```bash
# Backend (les magasins doivent être joignables : localhost:9042 / 6379 / 27017)
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000

# Frontend (proxy /api → localhost:8000)
cd frontend
npm install
npm run dev          # → http://localhost:5173
```

> Sans backend, l'interface bascule automatiquement sur des **données de démonstration**
> (bandeau « Données de démonstration » dans la barre supérieure).

## Fonctionnalités

- **Script qui génère des logs** — `backend/app/generator.py` (`python -m app.generator`)
- **Afficher les logs d'une application sur une période** — page *Journaux* + `GET /api/logs`
- **Signaler quand le nombre d'erreurs dépasse le seuil** — à chaque ERROR, le compteur
  Redis (+1 dans une fenêtre glissante de 5 min) est comparé au seuil MongoDB ; une alerte
  est ouverte/mise à jour, puis résolue automatiquement quand la fenêtre redescend
  (tâche de fond toutes les 15 s). Sévérité : **avertissement** ≥ seuil,
  **critique** ≥ 1,5 × seuil.
- **Recherche plein texte** — page *Recherche* (`GET /api/search`, Elasticsearch en priorité)

## API (aperçu)

| Méthode | Route | Description |
|---|---|---|
| `POST` | `/api/logs` | Ingestion d'un log (Cassandra + Redis si ERROR + index ES) |
| `GET` | `/api/logs?app=&start=&end=&level=&limit=` | Logs sur une période |
| `GET` | `/api/overview?hours=24` | Données complètes du tableau de bord |
| `GET` | `/api/apps` | Applications + erreurs/5 min + statut |
| `POST/PUT/DELETE` | `/api/apps[/{name}]` | CRUD des applications et seuils (MongoDB) |
| `GET` | `/api/alerts?status=open\|resolved\|all` | Alertes |
| `POST` | `/api/alerts/{id}/resolve` · `/api/alerts/resolve-all` | Résolution |
| `GET` | `/api/search?q=&app=&hours=` | Recherche plein texte |
| `GET` | `/api/health` | Santé de chaque magasin |

## Structure du projet

```
logscope/
├── docker-compose.yml
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── schema.cql                  # schéma Cassandra (appliqué au démarrage)
│   └── app/
│       ├── main.py                 # FastAPI + lifespan + /api/health
│       ├── cassandra_store.py      # logs + hourly_counts
│       ├── redis_store.py          # fenêtre d'erreurs 5 min, activité, récents
│       ├── mongo_store.py          # apps, seuils, alertes (+ seed de démo)
│       ├── es_store.py             # indexation / recherche (optionnelle)
│       ├── alerting.py             # détection de pics + tâche de fond
│       ├── generator.py            # script générateur de logs
│       └── routers/                # logs, overview, apps, alerts, search
└── frontend/
    ├── Dockerfile                  # build nginx (proxy /api → backend)
    └── src/
        ├── App.jsx                 # routes
        ├── api.js                  # client HTTP + repli démo
        ├── mock.js                 # données de démonstration
        ├── components/             # Sidebar, Topbar, StatCard, VolumeChart…
        └── pages/                  # Overview, Applications, Journaux…
```

## Notes d'implémentation

- **Schéma Cassandra** : logs partitionnés par `(app, day)` avec clustering `ts DESC` ;
  les agrégats horaires vivent dans une table `counter` (`hourly_counts`) — pas de scan
  du table `logs` pour le graphe.
- **Fenêtre glissante Redis** : un *sorted set* par application, member = identifiant
  unique, score = timestamp ; deux fenêtres (5 min courantes + 5 min précédentes) sont
  conservées pour afficher la tendance.
- **Résolution automatique** : tant que le compteur reste sous le seuil, la tâche de fond
  clôt les alertes ouvertes.
- L'ingestion reste disponible même si Elasticsearch est en panne (indexation *best effort*).
