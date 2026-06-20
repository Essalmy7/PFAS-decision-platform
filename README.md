# Système décisionnel pour l’analyse des données environnementales liées aux PFAS

## Présentation générale

Ce projet de recherche vise à construire un système décisionnel pour l’analyse de données environnementales ouvertes liées aux PFAS.

L’objectif est de transformer un jeu de données brut, volumineux et hétérogène en une architecture analytique structurée, interrogeable et exploitable à travers des tableaux de bord.

Le projet est actuellement en cours. Les premières phases de découverte, préparation, modélisation décisionnelle, chargement PostgreSQL et création des vues analytiques ont été réalisées. La phase suivante consiste à connecter PostgreSQL à Metabase afin de construire les premiers tableaux de bord interactifs.

## Contexte et motivation

Les PFAS constituent une famille de substances chimiques suivies de près dans les problématiques environnementales en raison de leur persistance et de leur présence potentielle dans différents milieux : eau, sol, sédiments, biote ou rejets industriels.

Les données disponibles sont riches mais difficiles à exploiter directement. Elles proviennent de plusieurs sources, utilisent différentes unités, concernent plusieurs matrices environnementales et contiennent parfois des informations détaillées sous forme semi-structurée.

Ce projet cherche donc à répondre à une problématique centrale :

Comment organiser, nettoyer, modéliser et valoriser des données environnementales ouvertes afin de faciliter leur analyse et leur supervision décisionnelle ?

## Objectifs du projet

Les objectifs principaux sont :

- Explorer et comprendre la structure du jeu de données PFAS.
- Identifier les limites de qualité : valeurs manquantes, zéros, valeurs extrêmes, hétérogénéité des unités.
- Construire une couche STAGING propre et standardisée.
- Transformer les données détaillées par substance en format long.
- Concevoir un modèle décisionnel adapté aux analyses environnementales.
- Charger les données modélisées dans PostgreSQL.
- Créer des vues analytiques dans un schéma dédié aux usages BI.
- Préparer l’exploitation des données dans Metabase.
- Poser les bases d’un système évolutif pour des analyses futures : dépassements de seuils, anomalies, indicateurs de risque.

## État d’avancement

Le projet est en cours de développement.

Travail réalisé :

- Exploration initiale du dataset PFAS Data Hub.
- Analyse des colonnes, types, valeurs manquantes, matrices, unités et sources.
- Étude de la distribution de `pfas_sum`.
- Identification du taux important de zéros dans `pfas_sum`.
- Analyse des valeurs extrêmes sans suppression automatique.
- Transformation de la colonne `pfas_values` en table longue.
- Construction de deux tables STAGING :

  - `pfas_observations_staging`
  - `pfas_measurements_long_staging`
- Construction d’un modèle PROCESSED avec dimensions et tables de faits.
- Chargement du modèle dans PostgreSQL.
- Création des schémas :

  - `dw`
  - `mart`
- Création d’index sur les principales colonnes de jointure.
- Création des vues analytiques principales pour préparer Metabase.

Travail en cours :

- Connexion de PostgreSQL à Metabase.
- Construction des premiers tableaux de bord.
- Validation des indicateurs avec l’encadrement.
- Amélioration progressive de la documentation technique et analytique.

## Architecture du projet

L’architecture du projet suit une logique progressive, proche d’un pipeline décisionnel professionnel. Elle permet de passer de sources de données environnementales brutes à une base structurée, puis à des vues analytiques et à des tableaux de bord exploitables.

```text
Sources de données PFAS
        |
        v
Extraction des données
        |
        v
Zone RAW
Données brutes conservées sans modification
        |
        v
Découverte des données
Analyse de structure, qualité, unités, matrices et valeurs extrêmes
        |
        v
Zone STAGING
Nettoyage, standardisation et contrôles qualité
        |
        v
Zone PROCESSED
Dimensions, tables de faits et modèle décisionnel
        |
        v
Data Warehouse PostgreSQL
Schéma en étoile évolutif vers galaxie
        |
        v
Data Marts / Vues SQL
Tables prêtes pour l’analyse et la visualisation
        |
        v
Metabase
Tableaux de bord interactifs et indicateurs
        |
        v
Analyses avancées
Statistiques, anomalies, valeurs extrêmes, scores de risque
        |
        v
Recommandations d’aide à la décision
Priorisation, suivi environnemental et interprétation des indicateurs
```

Cette architecture permet de séparer clairement les responsabilités de chaque couche :

- Les sources de données PFAS représentent les jeux de données environnementales ouverts utilisés comme point de départ.
- La phase d’extraction permet de récupérer les données dans un format exploitable.
- La zone RAW conserve les données brutes sans modification afin de garantir la traçabilité.
- La phase de découverte sert à comprendre les colonnes, les unités, les matrices environnementales, les valeurs manquantes, les zéros et les valeurs extrêmes.
- La zone STAGING prépare des données nettoyées, standardisées et contrôlées.
- La zone PROCESSED organise les données sous forme de dimensions et de tables de faits.
- Le Data Warehouse PostgreSQL centralise les tables décisionnelles dans une base relationnelle.
- Les data marts et vues SQL simplifient l’exploitation des données pour la visualisation.
- Metabase servira à construire les tableaux de bord interactifs.
- Les analyses avancées permettront ensuite d’aller au-delà des KPI descriptifs.
- Les recommandations d’aide à la décision constitueront la partie interprétative et opérationnelle du projet.

À l’état actuel du projet, les couches RAW, STAGING, PROCESSED, Data Warehouse PostgreSQL et vues SQL sont déjà mises en place. La connexion à Metabase et la construction des tableaux de bord sont en cours. Les analyses avancées et les recommandations d’aide à la décision constituent les prochaines évolutions du projet.

### 1. Découverte des données

La première étape consiste à charger les données brutes et à analyser leur structure.

Actions réalisées :

- Chargement du fichier Parquet source.
- Analyse du volume.
- Création d’un dictionnaire de données.
- Étude des valeurs manquantes.
- Analyse des colonnes géographiques, temporelles, environnementales et analytiques.
- Étude des variables `matrix`, `unit`, `pfas_sum` et `pfas_values`.

Constats importants :

- Le dataset contient environ 942 000 observations initiales.
- La colonne `pfas_sum` présente une forte asymétrie.
- Une grande partie des observations a une valeur `pfas_sum` égale à zéro.
- Les valeurs extrêmes sont concentrées dans certains couples source, matrice et unité.
- La colonne `pfas_values` contient le détail des substances PFAS et nécessite une transformation spécifique.

### 2. Couche STAGING

La couche STAGING prépare les données pour la modélisation.

Deux tables principales sont produites :

- `pfas_observations_staging`
- `pfas_measurements_long_staging`

La première table conserve le niveau observation globale.
La seconde table représente le niveau substance PFAS mesurée dans une observation.

Traitements réalisés :

- Nettoyage des dates et années.
- Standardisation des unités.
- Standardisation des matrices environnementales.
- Contrôle des coordonnées géographiques.
- Création d’indicateurs qualité.
- Transformation de `pfas_values` en format long.
- Création d’un identifiant d’observation pour relier les deux niveaux.

Cette étape a produit environ 21,5 millions de lignes détaillées au niveau substance.

### 3. Couche PROCESSED

La couche PROCESSED organise les données selon un modèle décisionnel.

Dimensions créées :

- `dim_date`
- `dim_dataset`
- `dim_matrix`
- `dim_unit`
- `dim_location`
- `dim_substance`

Tables de faits créées :

- `fact_pfas_observation`
- `fact_pfas_measurement`

Le choix de deux tables de faits est central.

`fact_pfas_observation` représente le niveau observation globale et porte la mesure `pfas_sum`.

`fact_pfas_measurement` représente le niveau substance mesurée et porte la mesure `pfas_value`.

Cette séparation évite le double comptage de `pfas_sum` après transformation en format long. Elle permet également de distinguer les analyses globales des analyses substance par substance.

### 4. Data Warehouse PostgreSQL

Les tables PROCESSED sont chargées dans PostgreSQL dans une base dédiée.

Organisation retenue :

- Schéma `dw` pour les dimensions et tables de faits.
- Schéma `mart` pour les vues analytiques.

Les dimensions sont chargées avant les tables de faits afin de respecter la logique du modèle décisionnel.

Des index ont été créés sur les principales clés utilisées dans les jointures afin d’améliorer les performances des futures requêtes BI.

### 5. Data marts et vues SQL

Des vues analytiques ont été créées dans le schéma `mart`.

Vues principales :

- `vw_overview_kpis`
- `vw_pfas_by_country`
- `vw_pfas_by_matrix`
- `vw_pfas_by_year`
- `vw_top_substances`
- `vw_extreme_observations`

Ces vues préparent les indicateurs nécessaires aux futurs tableaux de bord Metabase.

Elles permettent de centraliser les jointures, les agrégations et les calculs principaux dans PostgreSQL plutôt que de les reconstruire directement dans l’outil BI.

## Structure du dépôt

La structure actuelle du dépôt est organisée comme suit :

```text
pfas-decision-system/
|
├── data/
│   ├── raw/
│   ├── staging/
│   └── processed/
|
├── notebooks/
│   ├── 01_data_discovery.ipynb
│   ├── 02_staging_pipeline.ipynb
│   ├── 03_processed_dw_modeling.ipynb
│   └── 04_postgresql_datawarehouse.ipynb
|
├── reports/
│   ├── 01_data_discovery_report.md
│   ├── 02_staging_pipeline_report.md
│   ├── 03_processed_dw_modeling_report.md
│   └── 04_postgresql_datawarehouse_report.md
|
├── sql/
│   ├── create_schemas.sql
│   ├── create_indexes.sql
│   └── create_mart_views.sql
|
├── src/
|
├── diagrams/
|
├── README.md
└── requirements.txt
```

Certains fichiers ou dossiers peuvent évoluer au fur et à mesure de l’avancement du projet.

## Comment lancer le projet

### 1. Cloner le dépôt

```bash
git clone <url-du-repository>
cd pfas-decision-system
```

### 2. Créer un environnement Python

```bash
python -m venv .venv
```

Sous Windows :

```bash
.venv\Scripts\activate
```

Sous Linux ou macOS :

```bash
source .venv/bin/activate
```

### 3. Installer les dépendances

```bash
pip install -r requirements.txt
```

### 4. Placer les données brutes

Les données brutes doivent être placées dans :

```text
data/raw/
```

Le fichier source utilisé dans ce projet provient du PFAS Data Hub.

### 5. Exécuter les notebooks dans l’ordre

```text
01_data_discovery.ipynb
02_staging_pipeline.ipynb
03_processed_dw_modeling.ipynb
04_postgresql_datawarehouse.ipynb
```

### 6. Configurer PostgreSQL

Créer une base PostgreSQL dédiée, par exemple :

```text
pfas_dw
```

Puis créer les schémas :

```sql
CREATE SCHEMA IF NOT EXISTS dw;
CREATE SCHEMA IF NOT EXISTS mart;
```

Les paramètres de connexion doivent être adaptés à l’environnement local.

### 7. Lancer la création des vues analytiques

Les vues SQL du schéma `mart` peuvent être créées après le chargement des dimensions et tables de faits dans PostgreSQL.

### 8. Connecter Metabase

La connexion Metabase est l’étape actuellement en cours.

Elle consiste à :

- connecter Metabase à PostgreSQL ;
- sélectionner la base `pfas_dw` ;
- exposer les vues du schéma `mart` ;
- construire les premiers tableaux de bord.

## Résultats ou livrables actuels

Livrables déjà produits :

- Rapport de découverte des données.
- Dictionnaire de données initial.
- Tables STAGING au format Parquet.
- Table longue des mesures par substance.
- Dimensions PROCESSED.
- Tables de faits PROCESSED.
- Base PostgreSQL structurée.
- Schémas `dw` et `mart`.
- Index PostgreSQL sur les principales clés.
- Vues analytiques prêtes pour Metabase.
- Rapports Markdown par phase.
- Rapport d’avancement synthétique pour présentation encadrant.

Résultats techniques importants :

- Transformation d’un dataset brut complexe en tables analytiques structurées.
- Passage d’un format semi-structuré à une table longue exploitable.
- Séparation claire entre observation globale et mesure par substance.
- Préparation d’une architecture décisionnelle évolutive.

## Limites actuelles

Le projet n’est pas encore finalisé.

Limites actuelles :

- Les tableaux de bord Metabase ne sont pas encore finalisés.
- Les seuils réglementaires ne sont pas encore intégrés dans le modèle.
- Les résultats ne doivent pas encore être interprétés comme une analyse environnementale définitive.
- La gestion des valeurs extrêmes reste à approfondir selon des règles métier ou scientifiques.
- Les données proviennent de sources hétérogènes, avec des conventions pouvant varier d’un dataset à l’autre.
- Certaines interprétations nécessitent une validation avec des experts du domaine environnemental.

## Prochaines étapes

Les prochaines étapes prévues sont :

- Finaliser la connexion PostgreSQL à Metabase.
- Construire un premier tableau de bord de synthèse.
- Créer des visualisations par pays, matrice, année et substance.
- Ajouter une page dédiée aux observations extrêmes.
- Documenter les indicateurs utilisés dans le dashboard.
- Étudier l’intégration de seuils réglementaires ou de référence.
- Préparer une logique d’analyse des dépassements.
- Explorer des indicateurs avancés : anomalies, score de risque, suivi temporel.
- Améliorer l’automatisation du pipeline.
- Préparer une documentation finale claire pour les utilisateurs.

## Compétences mobilisées

Ce projet mobilise plusieurs compétences complémentaires.

Analyse de données :

- Exploration de données.
- Statistiques descriptives.
- Analyse de distributions.
- Identification de valeurs extrêmes.
- Construction d’indicateurs.

Data engineering :

- Organisation en couches RAW, STAGING et PROCESSED.
- Transformation de données semi-structurées.
- Manipulation de volumes importants.
- Chargement progressif dans PostgreSQL.
- Indexation et optimisation SQL.

Modélisation décisionnelle :

- Définition du grain des tables.
- Construction de dimensions.
- Construction de tables de faits.
- Modèle en étoile évolutif vers un modèle en galaxie.
- Création de data marts.

Business Intelligence :

- Préparation de vues SQL pour Metabase.
- Construction future de tableaux de bord.
- Organisation des indicateurs.
- Séparation entre couche technique et couche analytique.

Compétences transverses :

- Documentation technique.
- Justification des choix de modélisation.
- Rigueur méthodologique.
- Communication avec un encadrant.
- Vision orientée aide à la décision.

## Valeur académique du projet

Ce projet présente une valeur académique car il combine une problématique environnementale actuelle avec une démarche complète de traitement, modélisation et valorisation de données.

Il ne se limite pas à une analyse ponctuelle. Il propose une architecture décisionnelle capable d’évoluer vers :

- une supervision environnementale ;
- une analyse territoriale ;
- une analyse par substance ;
- une détection de situations atypiques ;
- une comparaison entre matrices environnementales ;
- une aide à la priorisation des zones ou sources à étudier.

Le projet montre également la capacité à prendre des décisions techniques justifiées, notamment :

- ne pas supprimer automatiquement les valeurs extrêmes ;
- distinguer les zéros des valeurs manquantes ;
- ne pas comparer des concentrations sans tenir compte des unités ;
- séparer les observations globales des mesures par substance ;
- créer des vues analytiques adaptées aux futurs utilisateurs.

Cette démarche correspond à une logique professionnelle de projet data appliqué à un sujet environnemental.

## Auteur

Zakariae Es-salmy

Étudiant ingénieur à l’École Centrale Méditerranée

Projet de recherche sur la construction d’un système décisionnel pour l’analyse des données environnementales ouvertes liées aux PFAS.
