# TPG Data Platform Sandbox — Roadmap Exécutive

**Version** : 1.0  
**Date** : 30 janvier 2026  
**Horizon** : 4 semaines (MVP — Lot 1 complet)  
**Objectif principal** : Délivrer un projet end-to-end prêt portfolio (GitHub + README + article + screencast)

---

## Table des matières

1. [Vue d'ensemble stratégique](#vue-densemble-stratégique)
2. [Phases de développement détaillées](#phases-de-développement-détaillées)
3. [Roadmap chronologique](#roadmap-chronologique)
4. [Dépendances & blockers](#dépendances--blockers)
5. [Critères d'acceptation par phase](#critères-dacceptation-par-phase)
6. [Ressources & outillage](#ressources--outillage)
7. [Plan de risques](#plan-de-risques)

---

## Vue d'ensemble stratégique

### Objectifs

| Objectif | Poids | Timeline | Critère de succès |
|----------|-------|----------|-------------------|
| **Apprentissage technologique** | 40% | Semaines 1–4 | Maîtrise de l'intégration Airbyte → dbt → DuckDB → Superset |
| **Portfolio deployable** | 40% | Semaines 3–4 | Repo GitHub + README + article + GIF fonctionnels |
| **Dashboards analytiques** | 20% | Semaines 3–4 | Dashboard consolidé + drilldowns interactifs en Superset |

### Approche et principes

- **Itératif** : Chaque phase livre un MVP testable
- **Documenté** : Documentation au fur et à mesure, pas après
- **Reproductible** : Docker Compose comme artefact central
- **Scalable** : Architecture pensée pour extension (Lot 2, DataHub, etc.)

---

## Phases de développement détaillées

### Phase 1 : Infrastructure & Airbyte (Semaine 1 — 5 jours)

**Objectif** : Disposer d'une infrastructure Docker opérationnelle et données brutes en DuckDB.

#### 1.1 Setup Docker Compose (1–2 jours)

**Tâches**
- [ ] Créer repository GitHub `tpg-data-platform-sandbox`
- [ ] Initialiser structure de base (dossiers airflow/, dbt/, superset/, data/, docs/)
- [ ] Rédiger `docker-compose.yml` complet :
  - Airflow (orchestration)
  - Airbyte (EL)
  - PostgreSQL (metadata)
  - DuckDB (storage — via volume)
  - dbt (transformation — optionnel en conteneur)
  - Superset (BI)
- [ ] Configurer `.env` avec variables d'environnement
- [ ] Tester démarrage : `docker-compose up` → tous les services healthy
- [ ] Documenter ports d'accès et credentials de test

**Livrables**
- Repository GitHub fonctionnel
- `docker-compose.yml` + `.env` validés
- Documentation : `docs/setup_guide.md` (draft)

**Dépendances** : Aucune (tâche d'initialisation)

**Risques** : Versions Docker incompatibles → Tester sur deux versions d'OS (Linux, Mac, Windows WSL2)

---

#### 1.2 Configuration Airbyte & Ingestion GTFS (2–3 jours)

**Tâches**
- [ ] Accéder UI Airbyte (http://localhost:8000)
- [ ] Créer **Source** :
  - Type : HTTP (generic)
  - URL : `https://opendata.transport.swiss/gtfs/` (ou API TPG native)
  - Endpoints/streams à configurer :
    - Routes
    - Stops
    - Trips
    - Stop times
    - Calendar
  - Mode sync : Full refresh (initial)
- [ ] Créer **Destination** :
  - Type : DuckDB (ou Postgres si DuckDB connector pas stable)
  - Path : `/data/duckdb/tpg_data.duckdb`
  - Schema : `raw`
  - Mode : Overwrite (initial)
- [ ] Créer **Connection** (source → destination)
- [ ] Lancer sync manuel → Vérifier données en DuckDB
- [ ] Exécuter requête de validation :
  ```sql
  SELECT COUNT(*) FROM raw.raw_routes;
  SELECT COUNT(*) FROM raw.raw_stops;
  SELECT COUNT(*) FROM raw.raw_stop_times;
  SELECT COUNT(*) FROM raw.raw_trips;
  SELECT COUNT(*) FROM raw.raw_calendar;
  ```

**Livrables**
- Airbyte connection GTFS → DuckDB opérationnelle
- Raw tables créées et validées
- Screenshots de configuration Airbyte
- Documentation : `docs/airbyte_setup.md`

**Dépendances** : Phase 1.1 (Docker up)

**Risques**
- API TPG inaccessible → Fallback : télécharger export GTFS statique local
- Schéma API différent du GTFS standard → Adapter mapping Airbyte

---

#### 1.3 Validation des données brutes (1 jour)

**Tâches**
- [ ] Profiling initial (row counts, nulls, data types)
- [ ] Validation métier :
  - Routes : format route_id, présence route_type
  - Stops : géolocalisation valide (lat/lon dans limites Genève)
  - Stop times : format horaires (HH:MM:SS), séquences cohérentes
  - Trips : clés étrangères (route_id, service_id existants)
  - Calendar : dates cohérentes (start_date ≤ end_date)
- [ ] Documenter anomalies détectées + plan de nettoyage
- [ ] Écrire notebook de validation (`scripts/validate_raw_data.py`)

**Livrables**
- Rapport de profiling data
- Notebook validation (`scripts/validate_raw_data.py`)
- Liste des nettoyages à appliquer en staging dbt

**Dépendances** : Phase 1.2 (données en DuckDB)

**Fin Phase 1** : Infrastructure stable, données brutes en place. Prêt pour transformation.

---

### Phase 2 : dbt & Modélisation (Semaine 2 — 5 jours)

**Objectif** : Transformer données brutes en modèles analytiques validés (staging → marts).

#### 2.1 Initialisation dbt & Structure (1 jour)

**Tâches**
- [ ] Initialiser dbt project : `dbt init tpg_models`
- [ ] Configurer `dbt_project.yml` :
  - name : `tpg_models`
  - version : 1.0.0
  - profile : `tpg`
  - models path : models/
- [ ] Configurer `profiles.yml` (DuckDB) :
  ```yaml
  tpg:
    target: dev
    outputs:
      dev:
        type: duckdb
        path: /data/duckdb/tpg_data.duckdb
        threads: 4
        schema: analytics
  ```
- [ ] Créer structure de dossiers :
  ```
  models/
    ├── sources.yml
    ├── staging/
    ├── intermediate/
    └── marts/
  tests/
  ```
- [ ] Vérifier connectivité : `dbt debug`

**Livrables**
- dbt project initialisé
- `profiles.yml` configuré pour DuckDB
- Structure de dossiers en place

**Dépendances** : Phase 1.2 (données brutes)

**Risques** : Connector DuckDB non stable → Fallback PostgreSQL

---

#### 2.2 Modèles Staging — Nettoyage (1.5 jours)

**Tâches**
- [ ] Créer `models/sources.yml` : définir sources raw
  ```yaml
  sources:
    - name: tpg_raw
      database: main
      schema: raw
      tables:
        - name: raw_routes
        - name: raw_stops
        # etc.
  ```
- [ ] Implémenter modèles staging (1 model par table source) :
  - `stg_tpg_routes.sql` : Normaliser types de routes, couleurs
  - `stg_tpg_stops.sql` : Valider géolocalisation (remove nulls, bounds check)
  - `stg_tpg_stop_times.sql` : Parser horaires (STRING → TIME), valider sequences
  - `stg_tpg_trips.sql` : Jointure route + service validation
  - `stg_tpg_calendar.sql` : Normaliser calendriers, transformer bool (0/1 → true/false)
- [ ] Pour chaque model : tests génériques (not_null, unique sur clés)
- [ ] Valider output : `dbt run`
- [ ] Documenter colonnes clés

**Livrables**
- 5 modèles staging opérationnels
- Tests dbt configurés
- Documentation YAML (description, tests)

**Dépendances** : Phase 2.1 (dbt setup)

**Risques** : Complexité parsing horaires → Utiliser fonctions DuckDB datetime native

---

#### 2.3 Modèles Intermediate — Enrichissement (1.5 jours)

**Tâches**
- [ ] Implémenter intermediate models (logique métier) :
  - `int_routes_enriched.sql` : Routes + count(stops), count(trips), avg_daily_trips
  - `int_stops_enriched.sql` : Stops + count(routes servies), centre géographique
  - `int_trips_with_schedule.sql` : Trips + calendar appliqué, days_of_week, validity_window
- [ ] Tester jointures + calculs : `dbt run --select intermediate`
- [ ] Valider volumétries et data quality

**Livrables**
- 3 modèles intermediate
- Calculs de features métier validés

**Dépendances** : Phase 2.2 (staging)

---

#### 2.4 Modèles Marts — Dimensions & Faits (1 jour)

**Tâches**
- [ ] Implémenter marts (analytics-ready) :
  - **Dimensions** :
    - `dim_route.sql` : PK (route_id), short_name, long_name, type, color, stop_count, trip_count
    - `dim_stop.sql` : PK (stop_id), code, name, lat, lon, served_by_route_count
    - `dim_calendar.sql` : PK (service_id), days_of_week bitmask, start_date, end_date, calendar_type
  - **Faits** :
    - `fct_trip_schedule.sql` : FK(trip_id, route_id, service_id), stop_count, first_stop_time, last_stop_time, duration_minutes
    - `fct_stop_times_analysis.sql` : FK(trip_id, stop_id, service_id), arrival_time, departure_time, stop_sequence, wait_minutes (arrondissage)
- [ ] Tests dbt complets : `dbt test`
- [ ] Générer dbt Docs : `dbt docs generate`

**Livrables**
- 5 modèles marts
- Tests passants (100%)
- dbt Docs générés (index.html)

**Dépendances** : Phase 2.3 (intermediate)

---

#### 2.5 Data Quality & Documentation (1 jour)

**Tâches**
- [ ] Exécuter tous les tests : `dbt test` → Vérifier 100% passants
- [ ] Profiling final marts :
  - Row counts par table
  - Null percentages
  - Cardinality clés
- [ ] Documenter data dictionary en YAML (columns, descriptions, tests)
- [ ] Générer et vérifier dbt Docs localement

**Livrables**
- Tous tests passants
- Data dictionary documenté
- dbt Docs consultables (http://localhost:8000 ou local)

**Fin Phase 2** : Données transformées, validées, documentées. Prêt pour BI.

---

### Phase 3 : Orchestration Airflow (Semaine 2–3 — 3 jours)

**Objectif** : Orchestrer pipelines Airbyte + dbt avec scheduling et monitoring.

#### 3.1 DAG Lot 1 — Pipeline GTFS (2 jours)

**Tâches**
- [ ] Créer `airflow/dags/tpg_lot1_pipeline.py` :
  - DAG ID : `tpg_lot1_gtfs`
  - Schedule : `@daily` (ou `0 2 * * *` pour 2h du matin)
  - Tâches :
    1. `airbyte_sync_gtfs` : AirbyteTriggerSyncOperator
       - Connection ID : `tpg-gtfs-to-duckdb`
       - Wait completion : true
    2. `dbt_run_lot1` : BashOperator
       - Commande : `cd /dbt && dbt run --models staging+ intermediate+ marts`
    3. `dbt_test_lot1` : BashOperator
       - Commande : `cd /dbt && dbt test`
    4. `generate_dbt_docs` : BashOperator (optionnel)
  - Dépendances : 1 → 2 → 3
- [ ] Tester DAG syntaxe : `airflow dags validate`
- [ ] Scheduler Airflow running : `docker-compose logs airflow-scheduler`
- [ ] Trigger manuel DAG → Vérifier logs + output

**Livrables**
- DAG Airflow complet
- Scheduler opérationnel
- Logs validant exécution succès

**Dépendances** : Phase 1.2 (Airbyte), Phase 2.4 (dbt)

**Risques**
- Versions Airflow/dbt incompatibles → Tester sur sandbox avant prod
- Airbyte trigger non disponible → Fallback : Python SDK ou script bash + polling

---

#### 3.2 Monitoring & Alerting (1 jour)

**Tâches**
- [ ] Accéder Airflow UI (http://localhost:8080)
- [ ] Configurer logs :
  - Level : INFO
  - Path : `/airflow/logs/`
- [ ] Ajouter retry policy à tâches critiques :
  - `retries : 2`
  - `retry_delay : timedelta(minutes=5)`
- [ ] Ajouter email alerting (optionnel) :
  - `email_on_failure : true`
  - `email : [your_email@example.com]`
- [ ] Documenter procédure de troubleshooting

**Livrables**
- DAG avec SLA/retry configurés
- Documentation alerting
- Logs centralisés

**Fin Phase 3** : Pipelines orchestrés et monitorables. Prêt pour BI.

---

### Phase 4 : Superset & Dashboards (Semaine 3 — 4 jours)

**Objectif** : Créer dashboards analytiques interactifs.

#### 4.1 Configuration Superset & Datasources (1.5 jours)

**Tâches**
- [ ] Accéder Superset UI (http://localhost:8088)
- [ ] Configurer database connection DuckDB :
  - Type : DuckDB
  - Name : `tpg_warehouse`
  - Connection String : `duckdb:////data/duckdb/tpg_data.duckdb`
  - Test connection
- [ ] Créer datasets pour chaque mart :
  - `dim_route`
  - `dim_stop`
  - `dim_calendar`
  - `fct_trip_schedule`
  - `fct_stop_times_analysis`
- [ ] Pour chaque dataset :
  - Configurer colonnes (rename, type, formatage)
  - Ajouter computed fields si besoin
  - Tester preview (data doit apparaître)

**Livrables**
- Database connection validée
- 5 datasets créés + préviewés

**Dépendances** : Phase 2.4 (marts en DuckDB)

**Risques** : DuckDB connector non présent → Fallback PostgreSQL avec données dupliquées

---

#### 4.2 Dashboard Principal — Sections (2 days)

**Tâches**
- [ ] Créer dashboard : "Vue Consolidée TPG"
- [ ] Section 1 : Résumé exécutif (Big Numbers)
  - Total routes : COUNT(DISTINCT route_id) from dim_route
  - Total stops : COUNT(DISTINCT stop_id) from dim_stop
  - Total trips : COUNT(DISTINCT trip_id) from fct_trip_schedule
  - Couverture calendaire : MIN(start_date) to MAX(end_date) from dim_calendar
- [ ] Section 2 : Analyse par type transport (Bar Chart)
  - X : route_type
  - Y : count routes, count stops, sum trips
  - Drill-down : clic sur type → filtered view
- [ ] Section 3 : Géographie (Map)
  - Dataset : dim_stop
  - Lat/Lon : stop_lat, stop_lon
  - Label : stop_name
  - Color/size : routes servies
- [ ] Section 4 : Horaires & Calendrier (Line Chart + Table)
  - Days of week : trips distribution
  - Couverture calendaire : Gantt ou timeline
- [ ] Section 5 : Détails par ligne (Drilldown)
  - Sélecteur route_id (dropdown)
  - Table : stops + horaires associés

**Livrables**
- Dashboard complet avec 5 sections
- Interactivité (filters, drill-downs) fonctionnelle
- Visualisations lisibles et colorées

**Dépendances** : Phase 4.1 (datasets)

**Risques** : Performance queries lentes → Ajouter indexes DuckDB

---

#### 4.3 Export & Partage (0.5 jour)

**Tâches**
- [ ] Exporter dashboard JSON : Superset UI → "Export" → Save file
- [ ] Exporter datasources JSON
- [ ] Documenter procédure reconstruction dashboard en env neuf
- [ ] Ajouter screenshots dashboard au README

**Livrables**
- `superset/dashboards.json`
- `superset/datasources.json`
- Screenshots pour README

**Fin Phase 4** : Dashboards opérationnels et partageables.

---

### Phase 5 : Documentation & Portfolio (Semaine 4 — 5 jours)

**Objectif** : Packager projet complet pour GitHub + article + screencast.

#### 5.1 README Technique (1.5 jours)

**Tâches**
- [ ] Rédiger `README.md` :
  ```markdown
  # TPG Data Platform Sandbox
  
  ## Architecture
  [Diagramme + explication stack]
  
  ## Quick Start
  1. git clone ...
  2. docker-compose up -d
  3. Accès outils : Airflow (8080), Airbyte (8000), Superset (8088)
  
  ## Structure données
  [Schémas raw, staging, marts]
  
  ## Modèles dbt
  [Description rapide lineage]
  
  ## Troubleshooting
  [FAQ issues courantes]
  ```
- [ ] Ajouter instructions détaillées :
  - Prérequis (Docker, Docker Compose versions)
  - Setup initial (`docker-compose up`, attendre 2 min)
  - Accès (credentials, URLs)
  - Arrêt gracieux (`docker-compose down`)
- [ ] Inclure screenshots :
  - Airflow DAG
  - Superset dashboard
  - dbt Docs
- [ ] Ajouter badges (build status, license, etc.)

**Livrables**
- README.md complet et formaté
- Screenshots intégrés
- Instructions testées (dry-run sur env neuve)

---

#### 5.2 Article Walkthrough (2 jours)

**Tâches**
- [ ] Rédiger article (1500–2000 mots) :
  - **Intro** : Motivation (portfolio, apprentissage)
  - **Contexte** : TPG, cas d'usage analytique
  - **Architecture** : Stack & choix techniques
  - **Implémentation** : Phases, challenges, learnings
  - **Résultats** : Dashboards, insights, metrics
  - **Conclusion** : Next steps (Lot 2, scaling, etc.)
- [ ] Inclure :
  - Code snippets (DAG Airflow, model dbt)
  - Screenshots (avant/après)
  - Diagrammes (architecture, data flow)
- [ ] Publier sur :
  - Medium, Dev.to, ou blog personnel
  - Partager sur LinkedIn
- [ ] Format : Markdown → Medium formatting

**Livrables**
- Article complet (fichier `docs/walkthrough_article.md`)
- Publié sur plateforme
- Lien partagé sur LinkedIn/Twitter

---

#### 5.3 Screencast GIF (1 jour)

**Tâches**
- [ ] Préparer démo :
  - Env clean (ou state stable)
  - Notifications/popups cachées
  - Terminal largeur 120 chars
- [ ] Enregistrer séquence (60–90 sec) :
  1. Terminal : `docker-compose up` (x2 speed, skip attente)
  2. Browser : Ouvrir Airflow (8080) → Cliquer DAG → Voir execution
  3. Browser : Ouvrir Superset (8088) → Naviguer dashboard
  4. Browser : Cliquer drill-down chart
  5. Browser : Ouvrir dbt Docs → Voir lineage
- [ ] Outils :
  - Mac : ScreenFlow → Export GIF
  - Windows : ScreenToGif
  - Linux : SimpleScreenRecorder → ffmpeg to GIF
- [ ] Optimiser GIF : Max 5 MB, 60 FPS
- [ ] Ajouter GIF au README + article

**Livrables**
- GIF screencast (max 5 MB)
- Intégré README + article

**Dépendances** : Phases précédentes (env stable)

---

#### 5.4 Finalisations & Push GitHub (1 jour)

**Tâches**
- [ ] Cleanup repository :
  - Remove secrets (.env → .env.example)
  - Update .gitignore (data/, logs/, etc.)
  - Vérifier pas de PII en repo
- [ ] Ajouter livrables :
  - `docs/` : architecture.md, setup_guide.md, walkthrough_article.md
  - `README.md` + GIF
  - `CONTRIBUTING.md` (optionnel)
  - `LICENSE` (MIT ou Apache 2.0)
- [ ] Final test :
  - Clone repo en dir vierge
  - Exécuter `docker-compose up`
  - Vérifier accès outils
  - Vérifier dashboard fonctionne
- [ ] Push final vers GitHub
- [ ] Ajouter topics : `data-engineering`, `dbt`, `airflow`, `superset`, `docker`
- [ ] Partager lien sur LinkedIn, GitHub profile

**Livrables**
- Repository GitHub public + polished
- Documentation complète
- Prêt recrutement/clients

**Fin Phase 5** : Projet portfolio complet et prêt à partage.

---

## Roadmap chronologique

### Semaine 1 (29 janvier – 4 février)

| Jour | Lun | Mar | Mer | Jeu | Ven |
|------|-----|-----|-----|-----|-----|
| **Phase 1.1** | Setup repo + docker-compose.yml | ✓ Test démarrage | | | |
| **Phase 1.2** | Airbyte config (Source) | Airbyte config (Destination) | Sync test + validation | | |
| **Phase 1.3** | Data profiling | ✓ Rapport validation | | | |
| **État fin semaine** | Infrastructure + données brutes ✅ | | | | |

### Semaine 2 (5–11 février)

| Jour | Lun | Mar | Mer | Jeu | Ven |
|------|-----|-----|-----|-----|-----|
| **Phase 2.1** | dbt init + profiles | ✓ Debug test | | | |
| **Phase 2.2** | Staging models (routes, stops) | Staging (trips, calendar) | ✓ Tests génériques | | |
| **Phase 2.3** | Intermediate models | ✓ Jointures + calcs | | | |
| **Phase 2.4** | Marts dimensions | Marts faits | ✓ dbt docs gen | | |
| **Phase 2.5** | Data quality | ✓ Documentation | | | |
| **Phase 3.1** | DAG Airflow Lot 1 | ✓ Test exécution | | | |
| **État fin semaine** | Données transformées + orchestration ✅ | | | | |

### Semaine 3 (12–18 février)

| Jour | Lun | Mar | Mer | Jeu | Ven |
|------|-----|-----|-----|-----|-----|
| **Phase 3.2** | Monitoring Airflow | ✓ Alerting config | | | |
| **Phase 4.1** | Superset datasources | ✓ 5 datasets | | | |
| **Phase 4.2** | Dashboard sections 1–3 | Sections 4–5 | ✓ Drill-downs | | |
| **Phase 4.3** | Export JSON | ✓ Screenshots | | | |
| **État fin semaine** | Dashboards opérationnels ✅ | | | | |

### Semaine 4 (19–25 février)

| Jour | Lun | Mar | Mer | Jeu | Ven |
|------|-----|-----|-----|-----|-----|
| **Phase 5.1** | README.md rédaction | ✓ Screenshots + test | | | |
| **Phase 5.2** | Article draft | Article révision + publish | | | |
| **Phase 5.3** | Screencast recording | ✓ GIF optimization | | | |
| **Phase 5.4** | Cleanup repo | ✓ Final test | Push GitHub | Partage LinkedIn | |
| **État fin semaine** | Portfolio complet + shared ✅ | | | | |

---

## Dépendances & blockers

### Graph des dépendances

```
Phase 1.1 (Docker)
    └─→ Phase 1.2 (Airbyte)
            └─→ Phase 1.3 (Validation)
                    └─→ Phase 2.1 (dbt init)
                            └─→ Phase 2.2–2.5 (dbt models)
                                    └─→ Phase 3.1 (DAG Airflow)
                                    └─→ Phase 4.1 (Superset datasources)
                                            └─→ Phase 4.2–4.3 (Dashboards)
                                                    └─→ Phase 5 (Documentation)
```

### Blockers potentiels

| Blocker | Probabilité | Impact | Mitigation |
|---------|-------------|--------|-----------|
| API TPG inaccessible | Moyenne | Phase 1.2 bloquée | Utiliser données GTFS statiques (fallback URLs documentées) |
| DuckDB connector Airbyte instable | Basse | Phase 1.2, 4.1 | Fallback PostgreSQL (inclure `postgres` dans compose) |
| Complexité dbt (learnings courbe) | Moyenne | Phase 2 timeline | Utiliser templates exemple dbt ; documentation officielle |
| Performance Superset lente | Basse | Phase 4 | Ajouter indexes DuckDB ; optimiser queries |
| Temps pour documentation | Moyenne | Phase 5 | Documenter au fur et à mesure ; template article préparé |

---

## Critères d'acceptation par phase

### Phase 1 — Infrastructure ✅
- [ ] `docker-compose up` lance tous services sans erreur
- [ ] Airbyte accessible (http://localhost:8000)
- [ ] Raw tables créées en DuckDB (`raw_routes`, `raw_stops`, etc.)
- [ ] Row counts ≥ 100 pour chaque table
- [ ] Validation queries retournent résultats sans erreur

### Phase 2 — Transformation ✅
- [ ] `dbt run` exécute sans erreur (15 min max)
- [ ] `dbt test` passe 100% (zéro test failures)
- [ ] Mart tables créées (`dim_route`, `dim_stop`, `fct_trip_schedule`)
- [ ] dbt Docs généré (index.html consultable)
- [ ] Toutes colonnes documentées en YAML

### Phase 3 — Orchestration ✅
- [ ] DAG Airflow `tpg_lot1_gtfs` visible dans UI
- [ ] DAG exécutable (trigger manuel succès)
- [ ] Logs Airflow clairs (zéro erreur)
- [ ] Pipeline complet exécute < 10 minutes

### Phase 4 — Dashboards ✅
- [ ] Superset accessible (http://localhost:8088)
- [ ] Dashboard "Vue Consolidée TPG" créé
- [ ] 5 sections visibles + datas chargées
- [ ] Drill-downs interactifs fonctionnels
- [ ] Export JSON réussi

### Phase 5 — Documentation ✅
- [ ] README.md ≥ 500 mots, inclut quick start
- [ ] Article ≥ 1500 mots, publié
- [ ] GIF screencast ≤ 5 MB, ≥ 60 sec
- [ ] Repo GitHub public + README affiché
- [ ] Clone repo vierge → `docker-compose up` succès

---

## Ressources & outillage

### Prérequis

| Composant | Version | Statut |
|-----------|---------|--------|
| Docker | 20.10+ | Required |
| Docker Compose | 2.10+ | Required |
| Git | 2.35+ | Required |
| Python | 3.9+ | Optional (pour dev local) |
| VS Code | Latest | Recommended |

### Extensions VS Code recommandées

- `dbt Power User` (Altimeter AI)
- `Python` (Microsoft)
- `Docker` (Microsoft)
- `YAML` (Red Hat)

### Outils de documentation & design

- Markdown : VS Code + Markdown Preview
- Diagrammes : Mermaid (VS Code extension)
- Screenshots : Snagit ou macOS default (Cmd+Shift+5)
- GIF creation : ScreenFlow (Mac), ScreenToGif (Windows)
- Article publishing : Medium, Dev.to

---

## Plan de risques

### Risques techniques

#### R1 : API TPG inaccessible ou changement format
- **Probabilité** : Moyenne
- **Impact** : Phase 1.2 bloquée
- **Mitigation** :
  - Vérifier accès API en advance
  - Préparer URLs GTFS statiques en fallback (opendata.swiss, gtfs-static)
  - Tester avec petit subset avant intégration complète

#### R2 : DuckDB connector Airbyte instable
- **Probabilité** : Basse
- **Impact** : Phase 1.2, 4.1
- **Mitigation** :
  - Inclure PostgreSQL en backup dans docker-compose.yml
  - Adapter `profiles.yml` dbt pour Postgres si nécessaire
  - Test early avec données sample

#### R3 : Complexité dbt & learning curve steep
- **Probabilité** : Moyenne
- **Impact** : Phase 2 timeline dépassée
- **Mitigation** :
  - Utiliser dbt starter project comme template
  - Consulter dbt official docs (existe en français)
  - Breakpoints simples : 1 model staging = 1 jour max

#### R4 : Performance Superset insuffisante (queries lentes)
- **Probabilité** : Basse
- **Impact** : Phase 4 experience utilisateur dégradée
- **Mitigation** :
  - Ajouter indexes DuckDB sur clés de jointure
  - Limiter dataset preview rows (5k max)
  - Optimiser queries Superset (avoid COUNT(*), use approximations)

#### R5 : Temps pour documentation underestimé
- **Probabilité** : Haute
- **Impact** : Phase 5 timeline
- **Mitigation** :
  - Documenter au fur et à mesure (pas batch fin)
  - Template article préparé (structure + sections vides)
  - Screenshots pendant développement

### Risques humains

#### R6 : Fatigue / context switching
- **Probabilité** : Moyenne
- **Impact** : Qualité code/doc
- **Mitigation** :
  - Bloquer 4h/jour min sur projet (focus time)
  - Changer phase si stuck (ex : doc au lieu de coding)
  - Partager progress hebdo (accountability)

---

## Checklist de lancement

Avant de démarrer semaine 1 :

- [ ] Repository GitHub créé (`tpg-data-platform-sandbox`)
- [ ] Docker & Docker Compose installés (versions ok)
- [ ] API TPG accès testée (curl test endpoint)
- [ ] VS Code setup + extensions
- [ ] Calendrier bloqué (4h/jour min, semaines 1–4)
- [ ] Plan partage : LinkedIn, article platform identifiée
- [ ] Backup local + git push réguliers

---

## Annexe A : Quick Command Reference

```bash
# Démarrer projet
git clone https://github.com/[vous]/tpg-data-platform-sandbox
cd tpg-data-platform-sandbox
docker-compose up -d
docker-compose logs -f  # Monitor

# dbt commands
dbt run                 # Exécuter models
dbt test                # Tester
dbt docs generate       # Docs HTML
dbt docs serve          # Serve docs

# Accès outils
Airbyte    : http://localhost:8000
Airflow    : http://localhost:8080
Superset   : http://localhost:8088
DuckDB     : /data/duckdb/tpg_data.duckdb

# Arrêter
docker-compose down
```

---

**Fin de Roadmap**

Version 1.0 — 30 jan 2026
