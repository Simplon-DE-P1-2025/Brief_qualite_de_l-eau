# 💧 Qualité de l'Eau Potable — Pipeline ETL Medallion

Pipeline de données complet pour l'analyse de la qualité de l'eau potable en France, basé sur l'API publique Hub'eau. Implémenté selon l'architecture **Medallion (Bronze / Silver / Gold)** avec PySpark et Delta Lake sur Azure Databricks.

---

## Sommaire

- [Architecture](#architecture)
- [Stack technique](#stack-technique)
- [Source de données](#source-de-données)
- [Structure du projet](#structure-du-projet)
- [Installation et configuration](#installation-et-configuration)
- [Ordre d'exécution](#ordre-dexécution)
- [Description des couches](#description-des-couches)
- [Schéma des tables](#schéma-des-tables)
- [Limitations connues](#limitations-connues)

---

## Architecture

```
API Hub'eau (JSON)
        │
        ▼
┌───────────────┐
│    EXTRACT    │  Python pur — requests + pagination
│  01_extract   │
└───────┬───────┘
        │  raw_data.json (DBFS)
        ▼
┌───────────────────────────────────────────────┐
│                    BRONZE                     │
│               02_bronze.ipynb                 │
│  Stockage brut Delta Lake — aucune transform  │
│  • bronze/analyses                            │
│  • bronze/stations                            │
│  • bronze/parametres                          │
└───────────────────┬───────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────┐
│                    SILVER                     │
│               03_silver.ipynb                 │
│  Nettoyage, typage, outliers, conformité      │
│  • silver/analyses                            │
│  • silver/stations                            │
│  • silver/conformite                          │
└───────────────────┬───────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────┐
│                     GOLD                      │
│               04_gold.ipynb                   │
│  Modélisation analytique — SQL + agrégats     │
│  • gold/dim_stations                          │
│  • gold/dim_parametres                        │
│  • gold/dim_temps                             │
│  • gold/fait_mesures                          │
│  • gold/fait_conformite                       │
│  • gold/kpi_zone                              │
│  • gold/tendances                             │
│  • gold/alertes                               │
└───────────────────────────────────────────────┘
```

---

## Stack technique

| Outil | Usage | Version |
|---|---|---|
| Python | Extract + orchestration | 3.10+ |
| requests | Appels API Hub'eau | 2.x |
| PySpark | Transformation distribuée | 4.0 (Spark 4.0) |
| Delta Lake | Format de stockage | Natif Databricks |
| SQL | Requêtes analytiques Gold | Spark SQL |
| Azure Databricks | Environnement d'exécution | Runtime 17.3 LTS |
| Git / GitHub | Versionning du code | — |

---

## Source de données

**API Hub'eau — Qualité de l'eau potable**
- URL : `https://hubeau.eaufrance.fr/api/v1/qualite_eau_potable/resultats_dis`
- Documentation : [hubeau.eaufrance.fr](https://hubeau.eaufrance.fr/page/api-qualite-eau-potable)
- Authentification : aucune (API publique)
- Format : JSON paginé (max 10 000 résultats par page)
- Villes couvertes : **Lyon** (69123) et **Saint-Étienne** (42218)
- Période : à partir du `2024-01-01`

---

## Structure du projet

```
eau-potable-etl/
│
├── notebooks/
│   ├── 01_extract.ipynb       # Extract Python pur
│   ├── 02_bronze.ipynb        # Chargement brut Delta Lake
│   ├── 03_silver.ipynb        # Nettoyage & transformation PySpark
│   └── 04_gold.ipynb          # Modélisation analytique + SQL
│
├── data/
│   └── .gitkeep               # Dossier vide versionné
│
├── .gitignore
└── README.md
```

---

## Installation et configuration

### Prérequis locaux

```bash
pip install requests
```

### Prérequis Azure Databricks

- Workspace Azure Databricks (Premium ou Standard)
- Cluster Spark : runtime `17.3 LTS (Scala 2.13, Spark 4.0)`, nœud unique
- Delta Lake : natif dans le runtime, aucune installation requise

### Chemins DBFS

```python
BRONZE_PATH = "dbfs:/eau_potable/bronze"
SILVER_PATH = "dbfs:/eau_potable/silver"
GOLD_PATH   = "dbfs:/eau_potable/gold"
```

### Cloner le repo

```bash
git clone https://github.com/ton-user/eau-potable-etl.git
cd eau-potable-etl
```

---

## Ordre d'exécution

Les notebooks doivent être exécutés **dans l'ordre suivant** :

```
01_extract → 02_bronze → 03_silver → 04_gold
```

Chaque notebook doit être connecté au cluster Spark avant exécution (bouton **Connect** en haut du notebook sur Databricks).

> ⚠️ Éteindre le cluster après utilisation : **Compute → Terminate** pour éviter la consommation inutile de crédits Azure.

---

## Description des couches

### 🥉 Bronze — Données brutes

Stockage des données telles que retournées par l'API, sans aucune transformation. Sert de source de vérité immuable.

| Table | Description |
|---|---|
| `bronze/analyses` | Toutes les mesures brutes API |
| `bronze/stations` | Stations de prélèvement uniques |
| `bronze/parametres` | Paramètres chimiques uniques |

### 🥈 Silver — Données nettoyées

Transformation et standardisation des données brutes.

Opérations appliquées :
- Typage des dates (`to_timestamp`) et des valeurs numériques (`DoubleType`)
- Extraction de la valeur numérique depuis la colonne `limite_qualite_parametre` (ex: `"<=0,1 µg/L"` → `0.1`)
- Vérification de la cohérence des unités entre mesure et limite
- Suppression des doublons sur `(code_commune, date_prelevement, code_parametre)`
- Détection des outliers via méthode IQR (facteur 3)
- Standardisation de la conformité : `"C"` → `True`, `"N"` → `False`
- Calcul du ratio `valeur / limite` (uniquement si unités cohérentes)

| Table | Description |
|---|---|
| `silver/analyses` | Mesures nettoyées, typées, avec outliers et ratio |
| `silver/stations` | Stations validées, noms normalisés |
| `silver/conformite` | Agrégat de conformité par ville et paramètre |

### 🥇 Gold — Données analytiques

Modélisation en étoile pour l'analyse. Requêtes SQL Spark pour les agrégats.

**Dimensions :**

| Table | Description |
|---|---|
| `gold/dim_stations` | Référentiel des stations |
| `gold/dim_parametres` | Référentiel des paramètres chimiques |
| `gold/dim_temps` | Calendrier avec année, mois, trimestre |

**Tables de faits :**

| Table | Description |
|---|---|
| `gold/fait_mesures` | Mesures individuelles avec outliers |
| `gold/fait_conformite` | Conformité par mesure |

**Agrégats :**

| Table | Description |
|---|---|
| `gold/kpi_zone` | Taux de conformité par ville et paramètre |
| `gold/tendances` | Moyennes mensuelles par ville et paramètre |
| `gold/alertes` | Paramètres avec taux de conformité < 95% |

---

## Schéma des tables

### silver/analyses

| Colonne | Type | Description |
|---|---|---|
| `nom_ville` | string | Nom de la ville |
| `code_commune` | string | Code INSEE |
| `date_prelevement` | timestamp | Date et heure du prélèvement |
| `code_parametre` | string | Code du paramètre chimique |
| `libelle_parametre` | string | Nom du paramètre |
| `resultat_numerique` | double | Valeur mesurée |
| `libelle_unite` | string | Unité de mesure |
| `limite_val` | double | Valeur limite réglementaire extraite |
| `limite_unite` | string | Unité de la limite |
| `unite_coherente` | boolean | Cohérence entre unité mesure et limite |
| `ratio_limite` | double | `valeur / limite` si unités cohérentes |
| `conforme` | boolean | `true` conforme, `false` non conforme |
| `outlier` | boolean | `true` si valeur aberrante (IQR × 3) |

### gold/alertes

| Colonne | Type | Description |
|---|---|---|
| `nom_ville` | string | Ville concernée |
| `libelle_parametre` | string | Paramètre en alerte |
| `nb_mesures` | long | Nombre de mesures |
| `taux_conformite` | double | Taux de conformité (%) |
| `niveau_alerte` | string | `CRITIQUE` (< 80%) ou `ATTENTION` (< 95%) |

---

## Limitations connues

- **Volume** : l'API Hub'eau retourne jusqu'à 135 000 résultats pour Lyon seule — la pagination génère des timeouts réseau au-delà de ~50 pages consécutives. Un `sleep` de 1 seconde entre les pages et un retry automatique (3 tentatives, backoff exponentiel) sont implémentés pour mitiger ce problème.
- **Unités hétérogènes** : la colonne `limite_qualite_parametre` contient des chaînes texte non standardisées (`"<=0,1 µg/L"`, `"<=0 n/(100mL)"`). L'extraction regex couvre les cas courants mais peut échouer sur des formats non anticipés — les ratios sont alors `null`.
- **Valeurs qualitatives** : les mesures qualitatives (présence/absence) se trouvent dans `resultat_alphanumerique` et sont exclues du pipeline numérique actuel.
- **Cluster Databricks** : en mode nœud unique, le parallélisme Spark est limité. Pour des volumes de production, un cluster multi-nœuds est recommandé.

---

## Versionning Git

### Convention de commits

```
feat:     nouvelle fonctionnalité
fix:      correction de bug
refactor: réécriture sans changement fonctionnel
docs:     documentation
```

### Workflow

```bash
git status
git add notebooks/03_silver.ipynb
git commit -m "feat: ajout détection outliers Silver"
git push origin main
```
