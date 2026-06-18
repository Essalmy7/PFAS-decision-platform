# Rapport — Couche PROCESSED et modélisation du Data Warehouse PFAS

## 1. Objectif du notebook

Le notebook `03_processed_dw_modeling.ipynb` a pour objectif de construire la couche **PROCESSED** du projet PFAS.

Cette couche vient après la couche STAGING. Elle transforme les données nettoyées et standardisées en tables organisées selon une logique de Data Warehouse.

L’objectif principal est de préparer les données pour leur futur chargement dans PostgreSQL, en construisant :

* les dimensions d’analyse ;
* les tables de faits principales ;
* les clés techniques ;
* les relations entre les tables ;
* les fichiers Parquet prêts à être chargés dans la base décisionnelle.

Cette phase répond directement à la consigne de modélisation du projet : identifier les dimensions pertinentes, identifier plusieurs tables de faits intéressantes pour l’analyse des PFAS, puis synthétiser le modèle dans un diagramme décisionnel.

---

## 2. Rôle de la couche PROCESSED dans l’architecture du projet

L’architecture globale du projet suit le flux suivant :

```text
PFAS Data Hub
↓
RAW
↓
STAGING
↓
PROCESSED
↓
Data Warehouse PostgreSQL
↓
Metabase
↓
Tableaux de bord et aide à la décision
```

La couche `RAW` contient les données originales.
La couche `STAGING` contient des données nettoyées, standardisées et restructurées.
La couche `PROCESSED` contient les données organisées sous forme de dimensions et de tables de faits.

Ainsi, la couche PROCESSED représente le passage d’une donnée préparée à une donnée réellement exploitable dans un système décisionnel.

---

## 3. Données d’entrée de la couche PROCESSED

La couche PROCESSED s’appuie sur les deux tables produites dans le notebook `02_staging_pipeline.ipynb`.

Les fichiers d’entrée sont :

```text
data/staging/pfas_observations_staging.parquet
data/staging/pfas_measurements_long_staging.parquet
```

La première table, `pfas_observations_staging`, contient le niveau global des observations PFAS.

Son grain est :

```text
Une ligne = une observation environnementale globale
```

La deuxième table, `pfas_measurements_long_staging`, contient le niveau détaillé des mesures par substance.

Son grain est :

```text
Une ligne = une substance PFAS mesurée dans une observation
```

Cette séparation entre observation globale et mesure détaillée est essentielle pour éviter les erreurs d’interprétation, notamment le double comptage de `pfas_sum`.

---

## 4. Objectif de la modélisation décisionnelle

La modélisation décisionnelle vise à organiser les données pour faciliter les analyses.

Dans un Data Warehouse, on distingue généralement :

* les **dimensions**, qui représentent les axes d’analyse ;
* les **tables de faits**, qui contiennent les mesures numériques et les indicateurs.

Dans ce projet, les dimensions permettent d’analyser les données PFAS selon plusieurs axes :

* le temps ;
* la localisation ;
* la source de données ;
* la matrice environnementale ;
* l’unité de mesure ;
* la substance PFAS.

Les tables de faits permettent d’analyser :

* les observations globales avec `pfas_sum` ;
* les mesures détaillées substance par substance avec `pfas_value`.

---

## 5. Définition du grain des tables

Avant de construire les tables, le grain de chaque table a été défini.

Le grain représente ce que signifie une ligne dans une table.

| Table                   | Grain                                                                  |
| ----------------------- | ---------------------------------------------------------------------- |
| `dim_date`              | une ligne = une date, une année ou une information temporelle inconnue |
| `dim_dataset`           | une ligne = une source de données                                      |
| `dim_matrix`            | une ligne = une matrice environnementale                               |
| `dim_unit`              | une ligne = une unité de mesure                                        |
| `dim_location`          | une ligne = une localisation ou un site de mesure                      |
| `dim_substance`         | une ligne = une substance PFAS                                         |
| `fact_pfas_observation` | une ligne = une observation environnementale globale                   |
| `fact_pfas_measurement` | une ligne = une substance PFAS mesurée dans une observation            |

La définition du grain est une étape essentielle, car elle permet d’éviter les confusions entre les niveaux d’analyse.

---

## 6. Dimensions construites

Les dimensions principales construites dans cette couche sont :

```text
dim_date
dim_dataset
dim_matrix
dim_unit
dim_location
dim_substance
```

Ces dimensions seront chargées dans PostgreSQL avant les tables de faits, car les tables de faits contiennent des clés étrangères vers ces dimensions.

---

## 7. Dimension `dim_date`

### Objectif

La dimension `dim_date` permet d’analyser les données PFAS selon le temps.

Elle permettra notamment de répondre aux questions suivantes :

* combien de mesures ont été réalisées par année ?
* comment évoluent les concentrations dans le temps ?
* quelles années contiennent le plus de mesures positives ?
* quelles périodes présentent des valeurs extrêmes ?

### Problème traité

Le dataset ne contient pas toujours une date complète. Certaines observations possèdent une date précise, tandis que d’autres possèdent seulement une année.

Pour gérer cette situation, une clé temporelle `date_key` a été créée.

Cette clé peut prendre plusieurs formes :

| Cas                      | Exemple de `date_key` | Niveau      |
| ------------------------ | --------------------- | ----------- |
| date complète disponible | `2020-05-17`          | `date`      |
| année seulement          | `2020`                | `year_only` |
| information absente      | `unknown`             | `unknown`   |

### Colonnes principales

La dimension `dim_date` contient notamment :

| Colonne      | Description                    |
| ------------ | ------------------------------ |
| `date_id`    | clé technique de la dimension  |
| `date_key`   | clé logique temporelle         |
| `date_clean` | date complète nettoyée         |
| `year_clean` | année nettoyée                 |
| `month`      | mois                           |
| `quarter`    | trimestre                      |
| `day`        | jour                           |
| `date_level` | niveau de précision temporelle |

### Utilité

Cette dimension permettra de faire des analyses par année, mois, trimestre ou date complète lorsque l’information est disponible.

---

## 8. Dimension `dim_dataset`

### Objectif

La dimension `dim_dataset` permet de conserver la source des données.

Le dataset PFAS regroupe plusieurs sources différentes. Il est donc nécessaire de garder la traçabilité de chaque observation.

### Colonnes principales

| Colonne         | Description                          |
| --------------- | ------------------------------------ |
| `dataset_dw_id` | clé technique dans le Data Warehouse |
| `dataset_id`    | identifiant source du dataset        |
| `dataset_name`  | nom de la source de données          |

### Utilité

Cette dimension permet d’analyser :

* les volumes de données par source ;
* les valeurs extrêmes par dataset ;
* les taux de zéros par dataset ;
* la qualité des données selon la source.

Elle est particulièrement importante car chaque source peut avoir ses propres conventions de mesure, d’unité ou de codage des valeurs nulles.

---

## 9. Dimension `dim_matrix`

### Objectif

La dimension `dim_matrix` permet d’analyser les mesures PFAS selon le type de milieu environnemental.

Les matrices présentes dans les données incluent notamment :

* eaux souterraines ;
* eaux de surface ;
* eaux usées ;
* eau potable ;
* eau de mer ;
* sol ;
* sédiments ;
* biote ;
* lixiviat ;
* matrice inconnue.

### Colonnes principales

| Colonne               | Description                   |
| --------------------- | ----------------------------- |
| `matrix_id`           | clé technique de la dimension |
| `matrix`              | valeur brute de la matrice    |
| `matrix_standardized` | valeur standardisée           |

### Utilité

Cette dimension est essentielle, car les concentrations PFAS ne doivent pas être comparées globalement sans tenir compte de la matrice.

Une concentration dans l’eau en `ng/L` n’est pas directement comparable à une concentration dans le sol en `ng/kg`.

La dimension `dim_matrix` permet donc de structurer les analyses par milieu environnemental.

---

## 10. Dimension `dim_unit`

### Objectif

La dimension `dim_unit` permet de gérer les unités de mesure utilisées dans les données PFAS.

Les principales unités observées sont :

* `ng/L` ;
* `ng/kg` ;
* `ng/kg dw` ;
* `ng/kg fw`.

### Colonnes principales

| Colonne             | Description                   |
| ------------------- | ----------------------------- |
| `unit_id`           | clé technique de la dimension |
| `unit_standardized` | unité standardisée            |
| `unit_type`         | type d’unité                  |

### Utilité

Cette dimension permet d’éviter les comparaisons incorrectes entre des valeurs exprimées dans des unités différentes.

Elle permet également de distinguer les concentrations dans les liquides, les solides, le poids sec et le poids frais.

---

## 11. Dimension `dim_location`

### Objectif

La dimension `dim_location` permet d’analyser les mesures PFAS selon leur localisation géographique.

Elle servira aux analyses par :

* pays ;
* ville ;
* site ;
* coordonnées géographiques ;
* carte dans Metabase.

### Construction de la clé de localisation

Une clé logique `location_key` a été créée à partir des éléments suivants :

```text
country + city + site_name + latitude + longitude
```

Cette clé permet d’éviter de confondre deux sites qui auraient le même nom mais qui seraient situés dans des pays ou villes différents.

### Colonnes principales

| Colonne                 | Description                            |
| ----------------------- | -------------------------------------- |
| `location_id`           | clé technique de la dimension          |
| `location_key`          | clé logique de localisation            |
| `site_name`             | nom du site                            |
| `country`               | pays                                   |
| `city`                  | ville                                  |
| `lat`                   | latitude                               |
| `lon`                   | longitude                              |
| `has_valid_coordinates` | indicateur de validité des coordonnées |

### Utilité

Cette dimension permet de construire les futures analyses géographiques et cartographiques.

Elle permettra notamment de repérer les territoires les plus surveillés ou les sites présentant les concentrations les plus élevées.

---

## 12. Dimension `dim_substance`

### Objectif

La dimension `dim_substance` permet d’analyser les PFAS substance par substance.

Elle est construite à partir de la table longue issue de `pfas_values`.

### Construction de la clé substance

Une clé logique `substance_key` a été créée à partir de :

```text
nom standardisé de la substance + CAS ID + isomère
```

Cette clé permet d’identifier une substance de manière plus robuste qu’avec son nom seul.

### Colonnes principales

| Colonne                  | Description                   |
| ------------------------ | ----------------------------- |
| `substance_id`           | clé technique de la dimension |
| `substance_key`          | clé logique de substance      |
| `substance_name`         | nom de la substance           |
| `substance_standardized` | nom standardisé               |
| `cas_id`                 | identifiant CAS               |
| `isomer`                 | isomère                       |

### Utilité

Cette dimension est centrale pour analyser :

* les substances les plus fréquentes ;
* les substances les plus concentrées ;
* les substances associées à certaines matrices ;
* les substances associées à des valeurs sous limite de détection.

Elle permet de ne pas se limiter à l’analyse globale de `pfas_sum`.

---

## 13. Tables de faits construites

Deux tables de faits principales ont été construites :

```text
fact_pfas_observation
fact_pfas_measurement
```

Ces deux tables répondent à deux niveaux d’analyse différents.

---

## 14. Table de faits `fact_pfas_observation`

### Objectif

La table `fact_pfas_observation` représente le niveau global des observations PFAS.

Son grain est :

```text
Une ligne = une observation environnementale globale
```

### Colonnes principales

La table contient notamment :

| Colonne                 | Description                            |
| ----------------------- | -------------------------------------- |
| `fact_observation_id`   | clé technique de la table de faits     |
| `observation_id`        | identifiant de l’observation d’origine |
| `date_id`               | clé vers `dim_date`                    |
| `location_id`           | clé vers `dim_location`                |
| `dataset_dw_id`         | clé vers `dim_dataset`                 |
| `matrix_id`             | clé vers `dim_matrix`                  |
| `unit_id`               | clé vers `dim_unit`                    |
| `pfas_sum`              | somme globale des PFAS                 |
| `is_missing_pfas_sum`   | indicateur de valeur manquante         |
| `is_zero_pfas_sum`      | indicateur de valeur égale à zéro      |
| `is_positive_pfas_sum`  | indicateur de valeur positive          |
| `is_extreme_pfas_sum`   | indicateur de valeur extrême           |
| `has_valid_coordinates` | indicateur de coordonnées valides      |

### Utilité

Cette table permet d’analyser :

* la contamination globale ;
* la distribution de `pfas_sum` ;
* le taux de mesures positives ;
* le taux de mesures nulles ;
* les valeurs extrêmes ;
* les observations critiques par pays, matrice, source ou période.

---

## 15. Table de faits `fact_pfas_measurement`

### Objectif

La table `fact_pfas_measurement` représente le niveau détaillé substance par substance.

Son grain est :

```text
Une ligne = une substance PFAS mesurée dans une observation
```

### Colonnes principales

La table contient notamment :

| Colonne                  | Description                                |
| ------------------------ | ------------------------------------------ |
| `fact_measurement_id`    | clé technique de la table de faits         |
| `observation_id`         | identifiant de l’observation d’origine     |
| `date_id`                | clé vers `dim_date`                        |
| `location_id`            | clé vers `dim_location`                    |
| `dataset_dw_id`          | clé vers `dim_dataset`                     |
| `matrix_id`              | clé vers `dim_matrix`                      |
| `unit_id`                | clé vers l’unité globale                   |
| `pfas_unit_id`           | clé vers l’unité de la mesure individuelle |
| `substance_id`           | clé vers `dim_substance`                   |
| `pfas_value`             | valeur individuelle mesurée                |
| `pfas_less_than_flag`    | indicateur de valeur sous limite           |
| `is_missing_pfas_value`  | indicateur de valeur manquante             |
| `is_zero_pfas_value`     | indicateur de valeur égale à zéro          |
| `is_positive_pfas_value` | indicateur de valeur positive              |

### Utilité

Cette table permet d’analyser :

* les concentrations par substance ;
* les substances les plus fréquentes ;
* les substances les plus élevées ;
* les valeurs positives par substance ;
* les valeurs nulles par substance ;
* les mesures sous limite de détection ;
* les comportements par matrice, pays, source ou période.

---

## 16. Différence entre les deux tables de faits

Les deux tables de faits ne répondent pas au même niveau d’analyse.

| Table                   | Niveau                 | Mesure principale |
| ----------------------- | ---------------------- | ----------------- |
| `fact_pfas_observation` | observation globale    | `pfas_sum`        |
| `fact_pfas_measurement` | substance individuelle | `pfas_value`      |

Cette séparation est essentielle.

Après transformation en format long, `pfas_sum` est répétée sur plusieurs lignes correspondant aux différentes substances d’une même observation. Il ne faut donc pas sommer `pfas_sum` dans la table détaillée, car cela créerait un double comptage.

Pour les KPI globaux, il faut utiliser `fact_pfas_observation`.
Pour les analyses substance par substance, il faut utiliser `fact_pfas_measurement`.

---

## 17. Gestion des clés techniques

Chaque dimension possède une clé technique :

| Dimension       | Clé technique   |
| --------------- | --------------- |
| `dim_date`      | `date_id`       |
| `dim_dataset`   | `dataset_dw_id` |
| `dim_matrix`    | `matrix_id`     |
| `dim_unit`      | `unit_id`       |
| `dim_location`  | `location_id`   |
| `dim_substance` | `substance_id`  |

Chaque table de faits possède également une clé technique :

| Table de faits          | Clé technique         |
| ----------------------- | --------------------- |
| `fact_pfas_observation` | `fact_observation_id` |
| `fact_pfas_measurement` | `fact_measurement_id` |

Ces clés permettront de charger les tables dans PostgreSQL et de définir les relations entre les tables.

---

## 18. Méthode de liaison entre faits et dimensions

Pour relier les tables de faits aux dimensions, des clés logiques ont été utilisées.

Exemples :

| Dimension       | Clé logique utilisée           |
| --------------- | ------------------------------ |
| `dim_date`      | `date_key`                     |
| `dim_location`  | `location_key`                 |
| `dim_dataset`   | `dataset_id + dataset_name`    |
| `dim_matrix`    | `matrix + matrix_standardized` |
| `dim_unit`      | `unit_standardized`            |
| `dim_substance` | `substance_key`                |

Une difficulté mémoire a été rencontrée lors de l’utilisation de `merge`, notamment avec `dim_location`. Le problème venait probablement d’un risque de jointure many-to-many, c’est-à-dire une jointure où une clé n’est pas unique dans la dimension.

Pour résoudre ce problème, la méthode a été adaptée :

* sécurisation de l’unicité des clés dans les dimensions ;
* utilisation de dictionnaires de correspondance ;
* ajout des identifiants techniques avec `.map()`.

Cette méthode est plus légère en mémoire et plus adaptée à un volume important de données.

---

## 19. Contrôles qualité réalisés

Un contrôle qualité global a été réalisé après la construction des dimensions et des tables de faits.

Les contrôles effectués sont :

* vérification de l’existence des fichiers PROCESSED ;
* vérification des volumes des tables ;
* vérification de l’unicité des clés primaires ;
* vérification des valeurs manquantes dans les clés étrangères ;
* vérification de la cohérence entre observations et mesures détaillées.

Ces contrôles permettent de valider la cohérence technique de la couche PROCESSED avant le passage à PostgreSQL.

---

## 20. Fichiers produits dans la couche PROCESSED

À la fin du notebook, les fichiers suivants ont été produits dans le dossier :

```text
data/processed/
```

Les dimensions :

```text
dim_date.parquet
dim_dataset.parquet
dim_matrix.parquet
dim_unit.parquet
dim_location.parquet
dim_substance.parquet
```

Les tables de faits :

```text
fact_pfas_observation.parquet
fact_pfas_measurement.parquet
```

Ces fichiers constituent la base du futur Data Warehouse PostgreSQL.

---

## 21. Modèle décisionnel obtenu

Le modèle obtenu correspond à un schéma en étoile évolutif vers un schéma en galaxie.

Dans un premier temps, deux tables de faits principales sont construites :

```text
fact_pfas_observation
fact_pfas_measurement
```

Ces tables partagent plusieurs dimensions communes :

```text
dim_date
dim_location
dim_dataset
dim_matrix
dim_unit
```

La table `fact_pfas_measurement` utilise en plus :

```text
dim_substance
```

Le modèle peut être représenté de manière simplifiée comme suit :

```text
                         dim_date
                            |
dim_location ---- fact_pfas_observation ---- dim_dataset
                            |
                         dim_matrix
                            |
                         dim_unit


                         dim_date
                            |
dim_location ---- fact_pfas_measurement ---- dim_dataset
                            |
                         dim_matrix
                            |
                         dim_unit
                            |
                      dim_substance
```

Ce modèle pourra ensuite évoluer vers un schéma en galaxie plus complet avec d’autres tables de faits, par exemple :

```text
fact_exceedance
fact_anomaly
fact_risk_score
```

---

## 22. Tables de faits futures envisagées

Même si elles ne sont pas encore alimentées à ce stade, plusieurs tables de faits futures peuvent être envisagées.

### 22.1 `fact_exceedance`

Cette table permettrait d’analyser les dépassements de seuils.

Son grain pourrait être :

```text
Une ligne = un dépassement de seuil pour une substance, une observation ou un site
```

Elle pourrait contenir :

* valeur mesurée ;
* seuil de référence ;
* écart au seuil ;
* ratio de dépassement ;
* type de seuil.

### 22.2 `fact_anomaly`

Cette table permettrait d’analyser les anomalies détectées dans les données.

Son grain pourrait être :

```text
Une ligne = une anomalie détectée
```

Elle pourrait contenir :

* score d’anomalie ;
* méthode de détection ;
* indicateur d’anomalie ;
* niveau de criticité.

### 22.3 `fact_risk_score`

Cette table permettrait de classer les sites ou territoires selon un score de risque.

Son grain pourrait être :

```text
Une ligne = un score de risque pour un site ou un territoire sur une période
```

Elle pourrait contenir :

* score global ;
* classe de risque ;
* composante concentration ;
* composante fréquence ;
* composante diversité des substances ;
* composante anomalie.

Ces tables futures permettront d’enrichir le système décisionnel et de passer d’une simple analyse descriptive à une aide à la décision plus avancée.

---

## 23. Apport de la couche PROCESSED pour PostgreSQL

La couche PROCESSED prépare directement le chargement dans PostgreSQL.

Elle permet de disposer de tables déjà structurées avec :

* des clés primaires ;
* des clés étrangères ;
* des dimensions propres ;
* des tables de faits cohérentes ;
* des grains clairement définis.

Lors du passage à PostgreSQL, les dimensions devront être chargées avant les tables de faits.

L’ordre de chargement recommandé est :

```text
1. dim_date
2. dim_dataset
3. dim_matrix
4. dim_unit
5. dim_location
6. dim_substance
7. fact_pfas_observation
8. fact_pfas_measurement
```

Cet ordre garantit que les clés étrangères des tables de faits pourront pointer vers des dimensions déjà existantes.

---

## 24. Apport de la couche PROCESSED pour Metabase

La couche PROCESSED prépare aussi les futurs tableaux de bord Metabase.

Elle permettra de construire des vues SQL pour répondre aux questions suivantes :

* nombre total d’observations ;
* nombre total de mesures détaillées ;
* nombre de substances PFAS distinctes ;
* taux de mesures positives ;
* taux de valeurs nulles ;
* concentrations moyennes et médianes ;
* top substances ;
* top sites ;
* répartition par pays ;
* répartition par matrice ;
* évolution temporelle ;
* identification des valeurs extrêmes.

La séparation entre dimensions et tables de faits facilitera la construction de dashboards lisibles et fiables.

---

## 25. Limites actuelles

La couche PROCESSED prépare la structure décisionnelle, mais certaines limites restent à traiter dans les phases suivantes.

Les principales limites sont :

* les seuils réglementaires ne sont pas encore intégrés ;
* les dépassements ne sont pas encore calculés ;
* les anomalies ne sont pas encore détectées automatiquement ;
* le score de risque n’est pas encore construit ;
* certaines valeurs sous limite de détection nécessitent une interprétation métier ;
* les unités différentes limitent certaines comparaisons globales ;
* les conventions de données peuvent varier selon les datasets sources.

Ces limites seront prises en compte dans les prochaines phases du projet.

---

## 26. Conclusion

Le notebook `03_processed_dw_modeling.ipynb` a permis de construire la couche PROCESSED du projet PFAS.

À partir des tables STAGING, les principales dimensions d’analyse ont été créées :

```text
dim_date
dim_dataset
dim_matrix
dim_unit
dim_location
dim_substance
```

Deux tables de faits principales ont également été construites :

```text
fact_pfas_observation
fact_pfas_measurement
```

La première table permet d’analyser la contamination globale au niveau observation grâce à `pfas_sum`.

La deuxième table permet d’analyser les mesures détaillées substance par substance grâce à `pfas_value`.

Cette couche constitue une étape essentielle du projet, car elle transforme les données nettoyées en un modèle décisionnel structuré, prêt à être chargé dans PostgreSQL.

La prochaine étape consiste à créer le Data Warehouse PostgreSQL, définir les tables SQL, charger les données PROCESSED, puis construire des vues analytiques pour Metabase.
