# Data Platform Sandbox - Spécification

## 1. Contexte & Objectifs

### 1.1 Vision
Construire une **data platform sandbox fonctionnelle et reproductible** basée sur les open data des transports publics genevois (TPG), démonstrée dans Docker et prête pour le portfolio.

### 1.2 Objectifs (par ordre de priorité)
1. **Apprentissage technologique** : Maîtriser l'intégration end-to-end (Airbyte → dbt → DuckDB → Superset)
2. **Portfolio** : Disposer d'un projet complet, déployable et documenté (repo GitHub + README technique + article walkthrough + GIF screencast)

### 1.3 Public cible
- **Utilisateurs primaires** : Toi-même (apprentissage + portfolio)
- **Audience secondaire** : Recruteurs/clients (démonstration de compétences data engineering)

---

## 2. Architecture Globale

### 2.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────┐
│                      DOCKER COMPOSE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  Airbyte     │────▶│   DuckDB     │────▶│   dbt Core   │    │
│  │  (EL)        │     │  (Storage)   │     │  (T)         │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│         ▲                      ▲                      │           │
│         │                      │                      ▼           │
│    TPG API                Raw Data              Transformed Data │
│    (GTFS)              (raw schema)             (marts schema)   │
│                                                      │           │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │   Airflow    │────▶│   dbt Docs   │     │  Superset    │    │
│  │  (Orch.)     │     │  (Lineage)   │     │  (BI/Viz)    │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│         │                                            ▲           │
│         └────────────────────────────────────────────┘           │
│                  (coordination)                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Flux de données

**Lot 1 (Données statiques GTFS)**
```
TPG API (GTFS) 
  → Airbyte (HTTP connector)
  → DuckDB (raw schema)
  → dbt (sources → staging → intermediate → marts)
  → DuckDB (marts schema)
  → Superset (dashboards)
```

**Lot 2 (Temps réel)**
```
TPG API (Real-time positions)
  → Airbyte (HTTP connector - polling)
  → DuckDB (raw_realtime schema)
  → dbt (streaming models)
  → DuckDB (marts schema - updated)
  → Superset (dashboards with real-time data)
```

### 2.3 Orchestration

- **Airflow** : Python API pour orchestrer les pipelines Airbyte + dbt
- Tâches :
  - `sync_airbyte_gtfs` → `dbt_run_lot1` → `dbt_test_lot1` (Lot 1)
  - `sync_airbyte_realtime` → `dbt_run_lot2` → `dbt_test_lot2` (Lot 2, phase ultérieure)
  - Tous les 12h pour GTFS, toutes les 5min pour temps réel

---

## 3. Lot 1 : Données Statiques (GTFS)

### 3.1 Scope

**Données sources (TPG API - GTFS)**
- Routes (lignes de transport : bus, tram, trolleybus)
- Stops (arrêts)
- Stop times (horaires d'arrivée/départ par arrêt et trajet)
- Trips (trajets)
- Calendar (calendriers de service : jours de semaine, vacances, etc.)

**Géographie** : Tout Genève

**Fréquence** : 1x par jour (ou à la demande)

### 3.2 Connecteur Airbyte

**Source** : HTTP (API TPG publique)
- Endpoint : `https://opendata.transport.swiss/gtfs/` (ou équivalent TPG)
- Format : JSON ou CSV GTFS standard
- Extractions :
  - Routes
  - Stops
  - Stop_times
  - Trips
  - Calendar

**Destination** : DuckDB (local)
- Schema : `raw`
- Tables : `raw_routes`, `raw_stops`, `raw_stop_times`, `raw_trips`, `raw_calendar`
- Mode : Full refresh (ou incremental si API le supporte)

### 3.3 Structure DuckDB - Lot 1

```sql
-- Schema RAW (brutes, après Airbyte)
CREATE SCHEMA IF NOT EXISTS raw;

CREATE TABLE raw.raw_routes (
  route_id STRING,
  route_short_name STRING,
  route_long_name STRING,
  route_type INT,
  route_color STRING,
  route_text_color STRING,
  _airbyte_extracted_at TIMESTAMP,
  _airbyte_loaded_at TIMESTAMP
);

CREATE TABLE raw.raw_stops (
  stop_id STRING,
  stop_code STRING,
  stop_name STRING,
  stop_lat DOUBLE,
  stop_lon DOUBLE,
  stop_timezone STRING,
  _airbyte_extracted_at TIMESTAMP,
  _airbyte_loaded_at TIMESTAMP
);

CREATE TABLE raw.raw_stop_times (
  trip_id STRING,
  arrival_time STRING,
  departure_time STRING,
  stop_id STRING,
  stop_sequence INT,
  stop_headsign STRING,
  _airbyte_extracted_at TIMESTAMP,
  _airbyte_loaded_at TIMESTAMP
);

CREATE TABLE raw.raw_trips (
  trip_id STRING,
  route_id STRING,
  service_id STRING,
  trip_headsign STRING,
  direction_id INT,
  shape_id STRING,
  _airbyte_extracted_at TIMESTAMP,
  _airbyte_loaded_at TIMESTAMP
);

CREATE TABLE raw.raw_calendar (
  service_id STRING,
  monday INT,
  tuesday INT,
  wednesday INT,
  thursday INT,
  friday INT,
  saturday INT,
  sunday INT,
  start_date STRING,
  end_date STRING,
  _airbyte_extracted_at TIMESTAMP,
  _airbyte_loaded_at TIMESTAMP
);
```

---

## 4. Modélisation dbt - Lot 1

### 4.1 Structure des dossiers

```
dbt_project.yml
├── models/
│   ├── sources.yml (définition des sources raw)
│   ├── staging/
│   │   ├── stg_tpg_routes.sql
│   │   ├── stg_tpg_stops.sql
│   │   ├── stg_tpg_stop_times.sql
│   │   ├── stg_tpg_trips.sql
│   │   └── stg_tpg_calendar.sql
│   ├── intermediate/
│   │   ├── int_routes_enriched.sql
│   │   ├── int_stops_enriched.sql
│   │   └── int_trips_with_schedule.sql
│   └── marts/
│       ├── dim_route.sql
│       ├── dim_stop.sql
│       ├── dim_calendar.sql
│       ├── fct_trip_schedule.sql
│       └── fct_stop_times_analysis.sql
├── tests/
│   └── generic/ (not_null, unique)
└── dbt_docs/ (auto-generated)
```

### 4.2 Modèles clés - Lot 1

#### Staging (nettoyage, validation)
- `stg_tpg_routes` : Normalisation des routes (types, couleurs)
- `stg_tpg_stops` : Validation de géolocalisation
- `stg_tpg_stop_times` : Parsing des horaires (HH:MM:SS)
- `stg_tpg_trips` : Jointures route + service
- `stg_tpg_calendar` : Mapping des calendriers

#### Intermediate (logique métier)
- `int_routes_enriched` : Routes avec stats (nb arrêts, nb trajets)
- `int_stops_enriched` : Arrêts avec info de zone/région
- `int_trips_with_schedule` : Trajets avec calendrier appliqué

#### Marts (analytique)
- `dim_route` : Dimension pour les routes
- `dim_stop` : Dimension pour les arrêts
- `dim_calendar` : Dimension pour calendriers
- `fct_trip_schedule` : Faits pour planification des trajets
- `fct_stop_times_analysis` : Faits pour analyse des temps d'arrêt

### 4.3 Tests génériques - Lot 1

**Tests dbt activés :**
- `not_null` : Clés primaires (route_id, stop_id, trip_id, service_id)
- `unique` : Identifiants uniques
- Exemple en YAML :

```yaml
models:
  - name: dim_route
    columns:
      - name: route_id
        tests:
          - unique
          - not_null

  - name: dim_stop
    columns:
      - name: stop_id
        tests:
          - unique
          - not_null
      - name: stop_lat
        tests:
          - not_null
      - name: stop_lon
        tests:
          - not_null
```

---

## 5. Dashboards Superset - Lot 1

### 5.1 Architecture

**Approche** : 1 dashboard consolidé + drilldowns interactifs

### 5.2 Dashboard principal : "Vue Consolidée TPG"

#### Sections
1. **Résumé exécutif**
   - Nombre total de routes
   - Nombre total d'arrêts
   - Nombre total de trajets (trips)
   - Couverture géographique (min/max lat/lon)

2. **Analyse par type de transport**
   - KPIs par type (Bus / Tram / Trolleybus) :
     - Nombre de routes
     - Nombre d'arrêts
     - Nombre de trajets

3. **Géographie & Couverture**
   - Carte des arrêts (stop_lat / stop_lon)
   - Distribution géographique par ligne

4. **Horaires & Calendrier**
   - Jours de service (lundi-dimanche)
   - Couverture calendaire (start_date → end_date)
   - Nombre de trajets par jour de semaine

5. **Détails par ligne (drilldown)**
   - Sélectionner une route → voir :
     - Arrêts desservis
     - Trajets et horaires
     - Couverture calendaire

### 5.3 Drilldowns interactifs

- **Cliquer sur une route** → Voir détail des arrêts + horaires
- **Cliquer sur un arrêt** → Voir routes qui le desservent + temps de passage

---

## 6. Lot 2 : Temps Réel (Phase 2)

### 6.1 Scope additionnel

**Nouvelles données sources (TPG API - Real-time)**
- Position des véhicules (latitude, longitude, timestamp)
- Délai/avance par rapport à l'horaire
- Occupation (capacité utilisée si disponible)

**Fréquence** : Polling toutes les 5 minutes

### 6.2 Connecteur Airbyte (Lot 2)

**Source** : HTTP (API TPG temps réel)
- Endpoint : API vehicle positions de TPG
- Format : JSON
- Frequency : 5 minutes

**Destination** : DuckDB
- Schema : `raw_realtime`
- Tables : `raw_vehicle_positions`, `raw_delays`

### 6.3 Modèles dbt - Lot 2

#### Staging
- `stg_tpg_vehicle_positions` : Nettoyage des coordonnées
- `stg_tpg_delays` : Calcul des délais vs horaires

#### Intermediate
- `int_vehicle_journeys` : Trajectoires des véhicules
- `int_route_performance` : Perf par ligne (ponctualité, retards)

#### Marts
- `fct_vehicle_position` : Positions en temps réel
- `fct_route_punctuality` : Taux de ponctualité par ligne/heure
- `fct_vehicle_occupancy` : Taux d'occupation (si données dispo)

### 6.4 Dashboards Superset - Lot 2

**Ajout au dashboard consolidé :**
- Position live des véhicules (carte)
- Ponctualité par ligne (last 24h)
- Occupation moyenne par heure
- Alertes (retards > seuil)

---

## 7. Stack Technique Détaillée

### 7.1 Versions & Images Docker

| Composant | Version | Image | Port |
|-----------|---------|-------|------|
| Airbyte | Latest | `airbyte/server:latest` | 8000 |
| DuckDB | Latest | N/A (lib Python) | N/A |
| dbt Core | 1.7+ | `dbt/dbt-core:latest` | N/A |
| Apache Airflow | 2.7+ | `apache/airflow:latest` | 8080 |
| Apache Superset | Latest | `apache/superset:latest` | 8088 |
| Postgres (metadata) | 15 | `postgres:15` | 5432 |

### 7.2 Docker Compose - Structure

```yaml
version: '3.8'

services:
  # Orchestration
  airflow:
    image: apache/airflow:latest
    environment:
      AIRFLOW_HOME: /airflow
      AIRFLOW__CORE__DAGS_FOLDER: /airflow/dags
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql://user:password@postgres:5432/airflow
    ports:
      - "8080:8080"
    volumes:
      - ./airflow/dags:/airflow/dags
      - ./airflow/logs:/airflow/logs
    depends_on:
      - postgres

  # ELT
  airbyte:
    image: airbyte/server:latest
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:password@postgres:5432/airbyte
    depends_on:
      - postgres

  # Storage (local file-based DuckDB)
  duckdb:
    # DuckDB runs as library in Python (no container)
    # Data persisted to volume
    volumes:
      - ./data/duckdb:/data/duckdb

  # Transformation (via Airflow)
  dbt:
    # dbt runs in container or via Airflow DAG
    image: dbt/dbt-core:latest
    volumes:
      - ./dbt:/dbt
      - ./data/duckdb:/data/duckdb
    working_dir: /dbt

  # Visualization
  superset:
    image: apache/superset:latest
    environment:
      DATABASE_URL: duckdb:////data/duckdb/tpg_data.duckdb
    ports:
      - "8088:8088"
    depends_on:
      - postgres

  # Metadata (pour Airflow & Superset)
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: airflow
    ports:
      - "5432:5432"
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
```

### 7.3 Outils de développement

- **IDE** : VS Code + extensions dbt Power User
- **Version control** : Git + GitHub
- **CI/CD** : GitHub Actions (optionnel mais recommandé)
- **Monitoring** : dbt Cloud (optionnel) ou dbt Docs local

---

## 8. Structure Git & Livrables

### 8.1 Arborescence du repo

```
tpg-data-platform-sandbox/
├── README.md (documentation technique)
├── docker-compose.yml
├── .env (variables d'environnement)
├── .gitignore
│
├── airflow/
│   ├── dags/
│   │   ├── __init__.py
│   │   ├── tpg_lot1_pipeline.py (orchestration Lot 1)
│   │   └── tpg_lot2_pipeline.py (orchestration Lot 2)
│   ├── logs/
│   └── config/
│
├── dbt/
│   ├── dbt_project.yml
│   ├── profiles.yml (DuckDB connection)
│   ├── models/
│   │   ├── sources.yml
│   │   ├── staging/
│   │   ├── intermediate/
│   │   └── marts/
│   ├── tests/
│   └── dbt_docs/ (output)
│
├── superset/
│   ├── dashboards.json (export)
│   └── datasources.json (export)
│
├── data/
│   ├── duckdb/ (persisted DuckDB files)
│   └── postgres/ (persisted Postgres)
│
├── docs/
│   ├── architecture.md
│   ├── setup_guide.md
│   └── walkthrough_article.md (blog post)
│
└── scripts/
    ├── init_duckdb.py (setup initial)
    └── generate_screencast.sh (GIF generation)
```

### 8.2 Livrables finaux

#### A. Repo GitHub
- Code complet et prêt à l'emploi
- `docker-compose.yml` 100% fonctionnel
- Instructions de démarrage (3 commandes max)

#### B. README.md technique
Sections :
- Architecture globale
- Prérequis (Docker, Docker Compose)
- Quick start (` docker-compose up`)
- Structure des données
- Modèles dbt
- Accès aux outils (URLs)
- Troubleshooting

#### C. Article Walkthrough
- ~1500-2000 mots
- Sections :
  - Motivation & contexte
  - Architecture & decisions techniques
  - Setup & déploiement
  - Données : sources & modélisation
  - Dashboards : création & insights
  - Learnings & next steps
- Accompagné de screenshots

#### D. GIF Screencast
- Démonstration visuelle du sandbox en action
- Séquence :
  1. Démarrage (`docker-compose up`)
  2. Accès Airflow UI (DAG en cours)
  3. Accès Superset UI (dashboards)
  4. Drill into un tableau/chart
  5. Vue dbt Docs (lineage)
- Durée : 60-90 secondes
- Outil : ScreenFlow (Mac) ou ScreenToGif (Windows)

#### E. Portfolio Website (optionnel)
- Lien vers repo GitHub
- Embedded GIF + description
- Lien vers article walkthrough

---

## 9. Phases de Développement

### Phase 1 : Infrastructure & Airbyte (Week 1)
- [ ] Docker Compose setup (tous les services)
- [ ] Configuration Airbyte (GTFS connector)
- [ ] Test d'ingestion (raw tables en DuckDB)

### Phase 2 : dbt & Transformation (Week 2)
- [ ] Modèles dbt (sources → marts)
- [ ] Tests génériques (not_null, unique)
- [ ] dbt Docs generation

### Phase 3 : Orchestration Airflow (Week 2-3)
- [ ] DAG Lot 1 (Airbyte → dbt → tests)
- [ ] Scheduling
- [ ] Monitoring basique

### Phase 4 : Superset & Dashboards (Week 3)
- [ ] Connexion DuckDB → Superset
- [ ] Dashboard consolidé
- [ ] Drilldowns interactifs

### Phase 5 : Documentation & Portfolio (Week 4)
- [ ] README.md technique
- [ ] Article walkthrough
- [ ] Screencast GIF
- [ ] Push final vers GitHub

### Phase 6 : Lot 2 - Temps Réel (Post-MVP)
- [ ] Connecteur Airbyte RT
- [ ] Modèles dbt Lot 2
- [ ] DAG Airflow Lot 2
- [ ] Dashboards temps réel

---

## 10. Success Criteria

### Lot 1 complet
- ✅ Données TPG ingérées via Airbyte
- ✅ Modèles dbt transformés (staging → marts)
- ✅ Tests dbt passants (not_null, unique)
- ✅ Dashboards Superset fonctionnels + drilldowns
- ✅ Orchestration Airflow opérationnelle
- ✅ dbt Docs générés et consultables
- ✅ Repo GitHub avec README + article + GIF

### Lot 2 (optionnel mais recommandé)
- ✅ Données temps réel ingérées
- ✅ Modèles dbt RT transformés
- ✅ Dashboards temps réel en Superset
- ✅ Performances acceptables (< 30s pour refresh)

---

## 11. Risques & Mitigations

| Risque | Mitigation |
|--------|-----------|
| API TPG indisponible/changement | Utiliser données GTFS statiques en fallback ; tester avec échantillon |
| Complexité Airbyte | Commencer par simple HTTP connector ; escalader après |
| Performance DuckDB | Commencer petit (1 mois de données) ; scale progressivement |
| Superset apprentissage | Utiliser templates d'exemples ; documentation officielle |
| Temps pour documentation | Documenter au fur et à mesure ; template walkthrough préparé |

---

## 12. Notes additionnelles

### Tech decisions
- **DuckDB** : Stockage local, facile à déployer, performant pour analytique
- **dbt** : Transformation SQL transparente, tests intégrés, documentation auto
- **Airflow** : Orchestration scalable et observable
- **Superset** : BI open-source, intuitive, connecteurs multiples
- **Docker Compose** : Déploiement simple et reproductible

### Future improvements (hors scope initial)
- [ ] Intégration DataHub pour data governance (phase 2+)
- [ ] Great Expectations pour data quality avancée
- [ ] API dbt Cloud pour CI/CD
- [ ] Monitoring Prometheus/Grafana pour infra
- [ ] Migration vers Snowflake/BigQuery

---

**Version** : 1.0  
**Date** : 2026-01-30  
**Statut** : Spécification initiale - Prête pour itération
