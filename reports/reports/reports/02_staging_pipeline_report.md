# Rapport — Pipeline STAGING des données PFAS

## 1. Objectif du notebook

Le notebook `02_staging_pipeline.ipynb` a pour objectif de construire la couche **STAGING** du projet PFAS.

Cette couche constitue une étape intermédiaire entre les données brutes issues de la zone `RAW` et les futures tables du Data Warehouse. Elle permet de transformer les données initiales en tables propres, structurées et exploitables pour la suite du projet.

L’objectif principal est de passer d’un dataset brut contenant des informations parfois semi-structurées à deux tables STAGING principales :

* `pfas_observations_staging.parquet` ;
* `pfas_measurements_long_staging.parquet`.

La première table conserve le niveau global de l’observation, tandis que la deuxième table détaille les mesures PFAS substance par substance.

---

## 2. Rôle de la couche STAGING dans le projet

Dans l’architecture globale du projet, la couche STAGING se situe entre les données brutes et les données prêtes pour le Data Warehouse.

Le flux général est le suivant :

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
```

La zone `RAW` contient les données originales sans modification.
La zone `STAGING` contient une version nettoyée, standardisée et restructurée des données.
La zone `PROCESSED` servira ensuite à construire les dimensions et les tables de faits du Data Warehouse.

La couche STAGING ne doit donc pas supprimer agressivement les données. Son rôle est plutôt de conserver la traçabilité des valeurs brutes tout en ajoutant des colonnes standardisées et des indicateurs de qualité.

---

## 3. Données d’entrée

Les données d’entrée proviennent du fichier brut placé dans le dossier :

```text
data/raw/pfas_data_hub
```

Ce fichier est chargé avec Pandas à l’aide de la fonction :

```python
pd.read_parquet()
```

Le dataset contient des données environnementales liées aux PFAS. Il regroupe plusieurs sources de données, plusieurs pays, plusieurs matrices environnementales et plusieurs unités de mesure.

Les colonnes importantes utilisées dans le pipeline STAGING sont notamment :

| Colonne        | Description                                                |
| -------------- | ---------------------------------------------------------- |
| `dataset_id`   | identifiant du dataset source                              |
| `dataset_name` | nom du dataset source                                      |
| `name`         | nom du site ou point de mesure                             |
| `country`      | pays                                                       |
| `city`         | ville                                                      |
| `lat`          | latitude                                                   |
| `lon`          | longitude                                                  |
| `year`         | année de mesure                                            |
| `date`         | date de mesure si disponible                               |
| `category`     | catégorie de la ligne                                      |
| `matrix`       | matrice environnementale : eau, sol, sédiment, biote, etc. |
| `unit`         | unité globale de mesure                                    |
| `pfas_sum`     | somme globale des PFAS pour une observation                |
| `pfas_values`  | détail des substances PFAS sous forme semi-structurée      |

---

## 4. Filtrage des observations de type Measurement

La première étape du pipeline consiste à conserver uniquement les lignes correspondant à des mesures environnementales.

Le filtrage est effectué avec la condition :

```python
df_raw["category"] == "Measurement"
```

Ce choix est important, car le Data Warehouse doit principalement analyser des mesures PFAS exploitables. Les autres catégories peuvent contenir des informations utiles, mais elles ne constituent pas le cœur de la table de faits principale.

Après filtrage, un identifiant unique `observation_id` est créé pour chaque observation.

Cet identifiant est essentiel, car il permet de relier :

* la table globale des observations ;
* la table détaillée des mesures par substance PFAS.

Ainsi, une observation peut être liée à plusieurs substances PFAS mesurées.

---

## 5. Création de la table des observations

Une première table de travail est créée au niveau observation.

Le grain de cette table est :

```text
Une ligne = une observation environnementale globale
```

Les colonnes conservées sont les informations principales relatives à la source, à la localisation, à la date, à la matrice, à l’unité et à la somme PFAS.

Cette table servira plus tard de base à la future table de faits :

```text
fact_pfas_observation
```

Elle permettra d’analyser la contamination globale par site, pays, année, matrice ou source de données.

---

## 6. Nettoyage de `pfas_values`

La colonne `pfas_values` contient les détails des substances PFAS mesurées. Cependant, certaines lignes contiennent une liste vide `[]`.

Dans le pipeline STAGING, ces listes vides sont transformées en valeurs manquantes dans une colonne de travail appelée :

```text
pfas_values_clean
```

Cette transformation ne modifie pas la donnée brute directement. Elle crée une version nettoyée permettant de savoir si une observation contient ou non des détails PFAS exploitables.

Un indicateur est ensuite créé :

```text
has_pfas_details
```

Cet indicateur vaut :

* `True` si l’observation contient des détails PFAS exploitables ;
* `False` sinon.

Cette étape est importante, car seules les observations avec des détails PFAS peuvent être transformées en format long.

---

## 7. Nettoyage de `pfas_sum`

La colonne `pfas_sum` représente la somme globale des concentrations PFAS pour une observation.

Elle est convertie en format numérique afin de permettre les calculs statistiques et les contrôles qualité.

Plusieurs indicateurs sont ensuite créés :

| Indicateur             | Signification                                              |
| ---------------------- | ---------------------------------------------------------- |
| `is_missing_pfas_sum`  | la somme PFAS est manquante                                |
| `is_zero_pfas_sum`     | la somme PFAS vaut zéro                                    |
| `is_positive_pfas_sum` | la somme PFAS est positive                                 |
| `is_extreme_pfas_sum`  | la somme PFAS est considérée comme extrême dans son groupe |

Ces indicateurs permettent d’éviter une interprétation trop simpliste de `pfas_sum`.

En effet, une valeur `pfas_sum = 0` ne signifie pas forcément une absence totale de contamination. Elle peut correspondre à une non-détection, à une valeur sous limite de quantification ou à une convention de codage propre à un dataset.

---

## 8. Transformation de `pfas_values` en format long

La transformation principale du notebook consiste à convertir la colonne semi-structurée `pfas_values` en une table longue.

Avant transformation, une observation peut contenir plusieurs substances PFAS dans une seule cellule.

Exemple simplifié :

```text
Observation 1 → [PFOA, PFOS, PFHxA]
```

Après transformation, chaque substance devient une ligne séparée :

```text
Observation 1 → PFOA
Observation 1 → PFOS
Observation 1 → PFHxA
```

Cette transformation est réalisée en plusieurs étapes :

1. sélection des observations ayant des détails PFAS ;
2. conservation uniquement des colonnes utiles ;
3. parsing de `pfas_values_clean` en liste Python ;
4. application de `explode()` pour créer une ligne par substance ;
5. normalisation des dictionnaires PFAS avec `pd.json_normalize()` ;
6. ajout du préfixe `pfas_` aux colonnes issues de `pfas_values`.

Cette étape permet de passer d’un format semi-structuré à une structure tabulaire exploitable par SQL et par le futur Data Warehouse.

---

## 9. Résultat de la transformation en format long

L’exécution du notebook a produit les résultats suivants :

| Élément                                                   |                 Valeur observée |
| --------------------------------------------------------- | ------------------------------: |
| Observations `Measurement` avec détails PFAS exploitables |                         929 442 |
| Nombre moyen de substances PFAS par observation           |                           23,17 |
| Médiane du nombre de substances par observation           |                              22 |
| Nombre maximal de substances dans une observation         |                              96 |
| Taille après transformation `explode`                     |               21 530 916 lignes |
| Taille finale de la table longue                          | 21 530 916 lignes × 19 colonnes |

Ces résultats sont cohérents avec la logique de transformation.

Le passage de 929 442 observations à plus de 21,5 millions de lignes s’explique par le fait qu’une observation contient souvent plusieurs substances PFAS. Après transformation, chaque ligne représente une substance PFAS mesurée.

Le grain de la table longue est donc :

```text
Une ligne = une substance PFAS mesurée dans une observation
```

Cette table servira plus tard à construire la table de faits :

```text
fact_pfas_measurement
```

---

## 10. Différence entre `pfas_sum` et `pfas_value`

Une distinction importante a été identifiée entre deux niveaux de mesure.

| Colonne      | Niveau                 | Description                                    |
| ------------ | ---------------------- | ---------------------------------------------- |
| `pfas_sum`   | observation globale    | somme totale des PFAS pour une observation     |
| `pfas_value` | substance individuelle | valeur mesurée pour une substance PFAS précise |

Par exemple, si une observation contient :

```text
PFOA = 90 ng/L
PFOS = 40 ng/L
```

alors :

```text
pfas_sum = 130 ng/L
```

Après transformation en format long, `pfas_sum` est répétée sur chaque ligne issue de la même observation. Il ne faut donc pas sommer directement `pfas_sum` dans la table longue, car cela provoquerait un double comptage.

Pour les analyses globales, il faudra utiliser la future table :

```text
fact_pfas_observation
```

Pour les analyses par substance, il faudra utiliser :

```text
fact_pfas_measurement
```

---

## 11. Standardisation des unités

Les unités peuvent être écrites de différentes manières dans les données brutes.

La couche STAGING ajoute donc des colonnes standardisées :

```text
unit_standardized
pfas_unit_standardized
```

L’objectif est d’harmoniser les écritures, par exemple :

| Valeur brute | Valeur standardisée |
| ------------ | ------------------- |
| `ng/l`       | `ng/L`              |
| `ng/L`       | `ng/L`              |
| `ng/kg`      | `ng/kg`             |
| `ng/kg dw`   | `ng/kg dw`          |
| `ng/kg fw`   | `ng/kg fw`          |

Cette standardisation est nécessaire pour les futures analyses, car les concentrations ne doivent être comparées que lorsqu’elles utilisent des unités cohérentes.

---

## 12. Standardisation des matrices

La colonne `matrix` indique le type de milieu environnemental analysé.

Les principales matrices identifiées sont :

| Matrix           | Description        |
| ---------------- | ------------------ |
| `Groundwater`    | eaux souterraines  |
| `Surface water`  | eaux de surface    |
| `Wastewater`     | eaux usées         |
| `Drinking water` | eau potable        |
| `Sea water`      | eau de mer         |
| `Soil`           | sol                |
| `Sediment`       | sédiment           |
| `Biota`          | organismes vivants |
| `Leachate`       | lixiviat           |
| `Unknown`        | matrice inconnue   |

Une colonne standardisée est créée :

```text
matrix_standardized
```

Cette standardisation est importante, car les concentrations PFAS ne sont pas directement comparables entre l’eau, le sol, les sédiments ou le biote.

---

## 13. Nettoyage des dates et des années

Les colonnes temporelles sont nettoyées afin de préparer les futures analyses chronologiques.

La colonne `date` est convertie en format date dans :

```text
date_clean
```

La colonne `year` est convertie en année propre dans :

```text
year_clean
```

Lorsque la date est disponible, l’année peut être extraite directement de `date_clean`. Si la date est manquante, la valeur de `year` est utilisée.

Cette étape permettra ensuite de construire la dimension temporelle :

```text
dim_date
```

Elle permettra aussi de produire des indicateurs par année, mois ou période.

---

## 14. Contrôle des coordonnées géographiques

Les coordonnées géographiques sont contrôlées à partir des colonnes :

```text
lat
lon
```

Une coordonnée est considérée comme valide si :

```text
-90 ≤ lat ≤ 90
-180 ≤ lon ≤ 180
```

Un indicateur est créé :

```text
has_valid_coordinates
```

Cet indicateur permettra de savoir quelles observations peuvent être utilisées dans des analyses géographiques ou des cartes dans Metabase.

---

## 15. Gestion des valeurs extrêmes

Les valeurs extrêmes de `pfas_sum` sont conservées dans la couche STAGING.

Elles ne sont pas supprimées, car elles peuvent correspondre à des situations environnementales réellement critiques. Cependant, elles sont identifiées à l’aide d’un indicateur :

```text
is_extreme_pfas_sum
```

La méthode utilisée consiste à calculer le percentile 99 par groupe homogène :

```text
matrix_standardized + unit_standardized
```

Une valeur est considérée comme extrême si elle dépasse le percentile 99 de son groupe.

Cette méthode évite de comparer directement des concentrations qui ne sont pas exprimées dans les mêmes unités ou qui ne concernent pas les mêmes matrices environnementales.

---

## 16. Nettoyage des valeurs détaillées PFAS

Dans la table longue, la colonne `pfas_value` représente la valeur individuelle d’une substance PFAS.

Elle est convertie en format numérique.

Des indicateurs sont créés :

| Indicateur               | Signification                    |
| ------------------------ | -------------------------------- |
| `is_missing_pfas_value`  | valeur individuelle manquante    |
| `is_zero_pfas_value`     | valeur individuelle égale à zéro |
| `is_positive_pfas_value` | valeur individuelle positive     |

Ces indicateurs permettront de construire des KPI plus robustes dans le Data Warehouse, notamment le taux de détection par substance ou par matrice.

---

## 17. Traitement de `pfas_less_than`

La colonne `pfas_less_than` indique si une valeur est associée à une logique de type “inférieur à une limite”.

Elle est transformée en indicateur booléen :

```text
pfas_less_than_flag
```

Cet indicateur est important, car les valeurs sous limite de détection ou de quantification ne doivent pas être interprétées de la même manière qu’une mesure directement quantifiée.

---

## 18. Contrôles qualité réalisés

Plusieurs contrôles qualité sont intégrés dans le notebook avant l’export des tables STAGING.

Les principaux contrôles sont :

| Contrôle                                 | Objectif                                                     |
| ---------------------------------------- | ------------------------------------------------------------ |
| volume des tables                        | vérifier le nombre de lignes et colonnes                     |
| valeurs manquantes                       | identifier les colonnes incomplètes                          |
| unités PFAS                              | vérifier la cohérence des unités                             |
| matrices                                 | comprendre la répartition des milieux                        |
| substances les plus fréquentes           | identifier les PFAS les plus représentés                     |
| coordonnées valides                      | préparer les analyses géographiques                          |
| incohérences entre `unit` et `pfas_unit` | vérifier la cohérence entre unité globale et unité détaillée |

Ces contrôles permettent de documenter la qualité des données avant le passage vers la couche PROCESSED.

---

## 19. Tables STAGING produites

À la fin du notebook, deux tables sont exportées au format Parquet compressé.

### 19.1 Table `pfas_observations_staging`

Chemin :

```text
data/staging/pfas_observations_staging.parquet
```

Grain :

```text
Une ligne = une observation environnementale globale
```

Cette table contient notamment :

* l’identifiant de l’observation ;
* la source de données ;
* la localisation ;
* l’année et la date ;
* la matrice ;
* l’unité ;
* `pfas_sum` ;
* les indicateurs qualité ;
* les colonnes standardisées.

Cette table servira à construire :

```text
fact_pfas_observation
```

---

### 19.2 Table `pfas_measurements_long_staging`

Chemin :

```text
data/staging/pfas_measurements_long_staging.parquet
```

Grain :

```text
Une ligne = une substance PFAS mesurée dans une observation
```

Cette table contient notamment :

* l’identifiant de l’observation ;
* la source de données ;
* la localisation ;
* la matrice ;
* l’unité ;
* la substance PFAS ;
* l’identifiant CAS ;
* la valeur mesurée ;
* l’unité de la substance ;
* l’indicateur `pfas_less_than_flag`.

Cette table servira à construire :

```text
fact_pfas_measurement
```

---

## 20. Apport de cette couche pour le Data Warehouse

La couche STAGING prépare directement la modélisation décisionnelle.

Elle permet de séparer deux niveaux d’analyse :

| Niveau               | Table STAGING                    | Future table de faits   |
| -------------------- | -------------------------------- | ----------------------- |
| observation globale  | `pfas_observations_staging`      | `fact_pfas_observation` |
| mesure par substance | `pfas_measurements_long_staging` | `fact_pfas_measurement` |

Elle prépare aussi les futures dimensions :

| Dimension future | Colonnes sources                               |
| ---------------- | ---------------------------------------------- |
| `dim_date`       | `date_clean`, `year_clean`                     |
| `dim_location`   | `name`, `country`, `city`, `lat`, `lon`        |
| `dim_dataset`    | `dataset_id`, `dataset_name`                   |
| `dim_matrix`     | `matrix`, `matrix_standardized`                |
| `dim_unit`       | `unit`, `unit_standardized`                    |
| `dim_substance`  | `pfas_substance`, `pfas_cas_id`, `pfas_isomer` |

Ainsi, le notebook STAGING constitue une base essentielle pour la construction du schéma en étoile puis du schéma en galaxie du Data Warehouse.

---

## 21. Difficultés rencontrées

Une difficulté importante a été rencontrée lors de la transformation de `pfas_values`.

L’application de `explode()` directement sur tout le dataset brut a provoqué une erreur mémoire. Cette erreur s’explique par le fait que l’opération duplique toutes les colonnes pour chaque substance PFAS contenue dans une observation.

Pour résoudre ce problème, la méthode a été optimisée :

1. filtrage des observations utiles ;
2. conservation uniquement des colonnes nécessaires ;
3. suppression des colonnes lourdes avant `explode()` ;
4. transformation uniquement sur une table de travail réduite.

Cette optimisation est importante, car elle montre que le pipeline doit être pensé non seulement pour produire un résultat correct, mais aussi pour être exécutable sur un volume important de données.

---

## 22. Limites actuelles de la couche STAGING

La couche STAGING permet de structurer les données, mais certaines limites restent à traiter dans les étapes suivantes.

Les principales limites sont :

* les seuils réglementaires ne sont pas encore intégrés ;
* les valeurs sous limite de détection nécessitent une interprétation métier ;
* les unités différentes empêchent certaines comparaisons directes ;
* les valeurs extrêmes doivent être analysées plus finement ;
* certaines dates sont manquantes ou incomplètes ;
* certaines coordonnées peuvent être absentes ou invalides ;
* les données proviennent de plusieurs sources ayant des conventions différentes.

Ces limites devront être prises en compte dans la construction du Data Warehouse et des indicateurs décisionnels.

---

## 23. Conclusion

Le notebook `02_staging_pipeline.ipynb` a permis de construire une couche STAGING propre, structurée et exploitable pour la suite du projet PFAS.

Les données brutes ont été filtrées, nettoyées, standardisées et transformées en deux tables principales :

* une table globale au niveau observation ;
* une table détaillée au niveau substance PFAS.

Cette séparation est essentielle pour éviter les erreurs d’interprétation, notamment le double comptage de `pfas_sum` après transformation en format long.

La couche STAGING conserve la traçabilité des données brutes tout en ajoutant des colonnes standardisées et des indicateurs qualité. Elle prépare directement la création des dimensions et des tables de faits du Data Warehouse PostgreSQL.

La prochaine étape du projet consiste à construire la couche PROCESSED, avec les dimensions et les tables de faits suivantes :

```text
dim_date
dim_location
dim_dataset
dim_matrix
dim_unit
dim_substance

fact_pfas_observation
fact_pfas_measurement
```

Cette étape permettra de passer progressivement vers un modèle décisionnel de type étoile, puis vers un schéma en galaxie lorsque d’autres tables de faits seront ajoutées, comme les dépassements, les anomalies et les scores de risque.
