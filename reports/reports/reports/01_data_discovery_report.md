# Rapport — Découverte des données PFAS

## 1. Objectif du notebook

Le notebook `01_data_discovery.ipynb` a pour objectif de réaliser une première exploration du dataset PFAS avant toute transformation avancée.

Cette étape correspond à la phase de **découverte des données**. Elle permet de comprendre la structure du dataset, d’identifier les colonnes importantes, d’analyser la qualité des données et de repérer les principales difficultés qui devront être traitées dans le pipeline ETL.

L’objectif n’est pas encore de construire le Data Warehouse, mais de répondre aux questions suivantes :

* quel est le volume du dataset ?
* quelles sont les colonnes disponibles ?
* quelles colonnes sont importantes pour l’analyse des PFAS ?
* quelles sont les valeurs manquantes ?
* quelles sont les unités et matrices présentes ?
* comment se comporte la variable `pfas_sum` ?
* que contient la colonne semi-structurée `pfas_values` ?
* quelles règles devront être appliquées dans la couche STAGING ?

---

## 2. Position de cette étape dans le projet global

La découverte des données est la première étape technique du projet.

Le flux global du projet est le suivant :

```text
PFAS Data Hub
↓
Découverte des données
↓
Pipeline STAGING
↓
Couche PROCESSED
↓
Data Warehouse PostgreSQL
↓
Metabase
↓
Indicateurs, analyses et recommandations
```

La phase de découverte permet donc de préparer les décisions techniques du pipeline ETL et de la modélisation décisionnelle.

Elle sert à éviter de construire directement un Data Warehouse sans comprendre les données sources.

---

## 3. Données utilisées

Les données analysées proviennent du PFAS Data Hub. Elles ont été chargées dans le projet depuis le dossier :

```text
data/raw/pfas_data_hub
```

Le fichier a été lu avec Pandas à l’aide de la fonction :

```python
pd.read_parquet()
```

Le dataset contient environ :

```text
942 084 lignes
```

Chaque ligne représente une information issue d’un dataset environnemental lié aux PFAS. Ces informations peuvent concerner des mesures environnementales, des sites, des sources ou d’autres types d’éléments selon la colonne `category`.

---

## 4. Première inspection du dataset

La première étape a consisté à afficher :

* le nombre de lignes ;
* le nombre de colonnes ;
* les premières lignes avec `df.head()` ;
* la structure des colonnes avec `df.info()` ;
* la liste complète des colonnes avec `df.columns.tolist()`.

Cette inspection a permis de constater que le dataset contient des informations de plusieurs natures :

* informations sur la source de données ;
* informations géographiques ;
* informations temporelles ;
* informations sur la matrice environnementale ;
* informations sur les unités ;
* informations de concentration PFAS ;
* détails par substance dans la colonne `pfas_values`.

Cette étape est importante, car elle donne une première vision de la structure générale du dataset.

---

## 5. Création du dictionnaire de données

Un dictionnaire de données a été créé afin de documenter chaque colonne du dataset.

Le dictionnaire contient notamment :

| Élément          | Description                       |
| ---------------- | --------------------------------- |
| `column`         | nom de la colonne                 |
| `dtype`          | type de donnée détecté par Pandas |
| `missing_values` | nombre de valeurs manquantes      |
| `missing_rate_%` | pourcentage de valeurs manquantes |
| `unique_values`  | nombre de valeurs distinctes      |

Ce dictionnaire a ensuite été exporté dans le dossier `reports` sous le nom :

```text
reports/data_dictionary.csv
```

Cette étape est utile pour documenter la qualité initiale des données et préparer les choix de nettoyage.

---

## 6. Analyse de la colonne `category`

La colonne `category` permet de distinguer les différents types de lignes présents dans le dataset.

L’analyse de cette colonne est importante, car toutes les lignes ne correspondent pas nécessairement à des mesures environnementales.

Pour la suite du projet, les lignes de type :

```text
Measurement
```

sont les plus importantes, car elles correspondent aux observations qui peuvent alimenter les futures tables de faits du Data Warehouse.

Cette analyse a donc permis de décider que la couche STAGING devra filtrer principalement les observations de type `Measurement`.

---

## 7. Analyse des sources de données

Le dataset regroupe plusieurs sources, identifiées notamment par les colonnes :

* `dataset_id` ;
* `dataset_name`.

L’analyse des sources a montré que les données proviennent de plusieurs jeux de données différents, par exemple :

* `Flanders DOV` ;
* `France - ICPE` ;
* `france_naiades` ;
* `france_ades` ;
* `uk_water_quality` ;
* d’autres sources nationales ou régionales.

Cette diversité des sources est importante, car chaque dataset peut avoir ses propres conventions de mesure, de codage, d’unités ou de traitement des valeurs nulles.

Cela montre que le Data Warehouse devra conserver la traçabilité des sources, notamment à travers une future dimension :

```text
dim_dataset
```

---

## 8. Analyse géographique

Les colonnes géographiques analysées sont principalement :

* `country` ;
* `city` ;
* `lat` ;
* `lon`.

Ces colonnes permettent de situer les mesures PFAS dans l’espace.

L’analyse géographique est importante pour plusieurs raisons :

* identifier les pays les plus représentés ;
* repérer les zones avec beaucoup de mesures ;
* préparer les futures cartes dans Metabase ;
* construire une dimension géographique dans le Data Warehouse.

Les coordonnées `lat` et `lon` devront être contrôlées dans la couche STAGING afin de vérifier leur validité.

Une coordonnée valide doit respecter les conditions suivantes :

```text
-90 ≤ lat ≤ 90
-180 ≤ lon ≤ 180
```

Cette règle permettra de créer l’indicateur :

```text
has_valid_coordinates
```

---

## 9. Analyse temporelle

Les colonnes temporelles principales sont :

* `year` ;
* `date`.

L’analyse temporelle permet de comprendre la période couverte par les données.

Cette étape est importante pour préparer les futures analyses par année, mois ou période.

Dans la couche STAGING, les dates devront être nettoyées et standardisées avec deux colonnes principales :

```text
date_clean
year_clean
```

Ces colonnes prépareront ensuite la création de la dimension temporelle :

```text
dim_date
```

---

## 10. Analyse des matrices environnementales

La colonne `matrix` indique le milieu environnemental dans lequel la mesure a été réalisée.

Les matrices identifiées dans le dataset incluent notamment :

| Matrix           | Signification      |
| ---------------- | ------------------ |
| `Groundwater`    | eaux souterraines  |
| `Surface water`  | eaux de surface    |
| `Wastewater`     | eaux usées         |
| `Drinking water` | eau potable        |
| `Sea water`      | eau de mer         |
| `Soil`           | sol                |
| `Sediment`       | sédiments          |
| `Biota`          | organismes vivants |
| `Leachate`       | lixiviat           |
| `Unknown`        | matrice inconnue   |

Cette colonne est essentielle, car les concentrations PFAS ne doivent pas être comparées globalement sans tenir compte du type de matrice.

Par exemple, une concentration en `ng/L` dans l’eau n’est pas directement comparable à une concentration en `ng/kg` dans le sol.

La couche STAGING devra donc créer une colonne standardisée :

```text
matrix_standardized
```

---

## 11. Analyse des unités

La colonne `unit` indique l’unité de mesure utilisée pour les concentrations PFAS.

Les principales unités observées sont notamment :

| Unité      | Interprétation                            |
| ---------- | ----------------------------------------- |
| `ng/l`     | nanogrammes par litre                     |
| `ng/kg`    | nanogrammes par kilogramme                |
| `ng/kg dw` | nanogrammes par kilogramme de poids sec   |
| `ng/kg fw` | nanogrammes par kilogramme de poids frais |

Cette analyse a montré que les unités ne sont pas toutes comparables entre elles.

Il est donc nécessaire de standardiser les unités dans la couche STAGING avec une colonne :

```text
unit_standardized
```

Cette standardisation est indispensable pour éviter des comparaisons incorrectes dans les futurs indicateurs.

---

## 12. Signification de `pfas_sum`

La colonne `pfas_sum` représente la somme globale des concentrations PFAS associées à une observation.

Autrement dit, si une observation contient plusieurs substances PFAS mesurées, `pfas_sum` correspond à la somme de ces substances.

Exemple simplifié :

```text
PFOA = 90 ng/L
PFOS = 40 ng/L
pfas_sum = 130 ng/L
```

Cette variable donne donc une vision globale du niveau de contamination PFAS pour une observation.

Cependant, elle doit toujours être interprétée avec prudence, car sa signification dépend de la matrice et de l’unité utilisées.

---

## 13. Analyse de la distribution de `pfas_sum`

Une première visualisation de la distribution de `pfas_sum` a montré que les valeurs sont très asymétriques.

La majorité des observations se situe à des niveaux faibles ou modérés, mais certaines valeurs sont extrêmement élevées.

Le graphique brut était difficile à interpréter, car les valeurs extrêmes écrasaient la majorité des données.

Pour mieux comprendre la distribution, une transformation logarithmique a été utilisée :

```text
log10(pfas_sum)
```

Cette transformation a permis de visualiser plus clairement la distribution principale.

L’analyse a montré que la majorité des valeurs positives se situe entre :

```text
log10(pfas_sum) = 0 et 2
```

ce qui correspond environ à :

```text
pfas_sum entre 1 et 100
```

Cependant, la distribution possède une longue queue à droite, ce qui confirme la présence de valeurs extrêmes.

---

## 14. Analyse des valeurs extrêmes de `pfas_sum`

Les valeurs extrêmes ont été analysées en affichant les plus grandes valeurs de `pfas_sum`.

Cette analyse a montré que les valeurs maximales sont principalement associées à certains datasets et certaines matrices, notamment :

* `Flanders DOV` pour les eaux souterraines ;
* `France - ICPE` pour les eaux usées ;
* certains jeux de données britanniques ou italiens ;
* certaines mesures dans le sol ou les sédiments.

Une analyse groupée par :

```text
dataset_name + matrix + unit
```

a permis de mieux comprendre ces valeurs extrêmes.

Cette analyse a montré que les valeurs extrêmes ne sont pas réparties aléatoirement dans tout le dataset. Elles sont concentrées dans certains groupes spécifiques.

Un point important a aussi été identifié : dans plusieurs groupes, la médiane est beaucoup plus faible que la valeur maximale.

Cela signifie que les valeurs extrêmes correspondent probablement à quelques observations particulières et ne représentent pas forcément le comportement général du groupe.

---

## 15. Décision concernant les valeurs extrêmes

Les valeurs extrêmes ne doivent pas être supprimées directement.

Elles peuvent correspondre à des situations environnementales réellement critiques.

La bonne décision est donc :

* conserver les valeurs extrêmes ;
* les identifier avec un indicateur qualité ;
* les analyser par groupes homogènes de matrice et d’unité ;
* éviter de les mélanger dans une moyenne globale non contrôlée.

Dans la couche STAGING, cette décision sera traduite par la création d’un indicateur :

```text
is_extreme_pfas_sum
```

---

## 16. Analyse des valeurs égales à zéro dans `pfas_sum`

Une analyse spécifique a été réalisée sur les valeurs égales à zéro.

Le taux global obtenu est :

```text
Taux de zéro global : 60,31 %
```

Cela signifie qu’environ 60 % des lignes ont une valeur `pfas_sum` égale à zéro.

Cependant, cette information doit être interprétée avec prudence.

Une valeur `pfas_sum = 0` ne signifie pas forcément une absence totale de contamination. Elle peut correspondre à :

* une non-détection ;
* une valeur inférieure à une limite de détection ;
* une valeur non quantifiée ;
* une convention de codage propre au dataset source.

L’analyse par groupe a montré que certains datasets ou matrices présentent des taux de zéros très élevés, parfois proches de 100 %.

Cela confirme que les zéros ne sont pas répartis uniformément dans le dataset.

---

## 17. Décision concernant les zéros

Les valeurs égales à zéro doivent être conservées, mais elles doivent être identifiées.

Dans la couche STAGING, plusieurs indicateurs seront créés :

```text
is_zero_pfas_sum
is_positive_pfas_sum
is_missing_pfas_sum
```

Ces indicateurs permettront ensuite de construire des KPI plus robustes, par exemple :

* taux de mesures positives ;
* moyenne globale ;
* moyenne hors zéros ;
* médiane ;
* maximum ;
* percentiles.

La découverte des données a donc montré qu’il ne faut pas se limiter à une simple moyenne globale de `pfas_sum`.

---

## 18. Analyse de `pfas_values`

La colonne `pfas_values` est une colonne semi-structurée qui contient les détails des substances PFAS mesurées.

Elle peut contenir plusieurs substances dans une seule cellule.

Exemple simplifié :

```text
[
  {"substance": "PFOA", "value": 90, "unit": "ng/l"},
  {"substance": "PFOS", "value": 40, "unit": "ng/l"}
]
```

Cette colonne est essentielle pour les analyses détaillées par substance.

Cependant, elle n’est pas directement exploitable dans un Data Warehouse, car plusieurs informations sont stockées dans une seule cellule.

Il faudra donc la transformer en format long dans le pipeline STAGING.

---

## 19. Analyse des listes vides dans `pfas_values`

Certaines lignes contiennent une liste vide :

```text
[]
```

Ces valeurs ne sont pas considérées comme des valeurs manquantes par Pandas, mais elles ne contiennent pas de détail PFAS exploitable.

Une colonne nettoyée a donc été créée :

```text
pfas_values_clean
```

Dans cette colonne, les listes vides sont transformées en valeurs manquantes.

Après ce traitement, le taux de lignes sans détail PFAS exploitable est d’environ :

```text
1,34 %
```

Cela signifie que la colonne `pfas_values` est majoritairement exploitable.

Cette observation est importante, car elle confirme que la transformation en format long est pertinente pour la suite du projet.

---

## 20. Implications pour le pipeline STAGING

La phase de découverte des données a permis d’identifier les règles principales à appliquer dans le pipeline STAGING.

Les règles retenues sont les suivantes :

| Problème observé               | Décision STAGING                                   |
| ------------------------------ | -------------------------------------------------- |
| plusieurs catégories de lignes | filtrer principalement `Measurement`               |
| `pfas_values` semi-structuré   | parser et transformer en format long               |
| listes vides `[]`              | créer `pfas_values_clean`                          |
| unités hétérogènes             | créer `unit_standardized`                          |
| matrices différentes           | créer `matrix_standardized`                        |
| valeurs extrêmes               | conserver et créer `is_extreme_pfas_sum`           |
| nombreux zéros                 | créer `is_zero_pfas_sum` et `is_positive_pfas_sum` |
| dates parfois incomplètes      | créer `date_clean` et `year_clean`                 |
| coordonnées à vérifier         | créer `has_valid_coordinates`                      |

Ces règles constituent la base technique du notebook `02_staging_pipeline.ipynb`.

---

## 21. Implications pour le Data Warehouse

La découverte des données a aussi permis d’anticiper la future modélisation décisionnelle.

Deux niveaux d’analyse ont été identifiés :

### Niveau observation globale

Ce niveau correspond à une ligne par observation environnementale.

Il permettra de construire la future table :

```text
fact_pfas_observation
```

Cette table servira à analyser :

* `pfas_sum` ;
* les valeurs nulles ;
* les valeurs positives ;
* les valeurs extrêmes ;
* la contamination globale par site, pays, matrice ou année.

### Niveau mesure par substance

Ce niveau correspond à une ligne par substance PFAS mesurée dans une observation.

Il permettra de construire la future table :

```text
fact_pfas_measurement
```

Cette table servira à analyser :

* les substances PFAS les plus fréquentes ;
* les concentrations par substance ;
* les valeurs sous limite de détection ;
* la répartition par matrice, pays, année ou source.

---

## 22. Dimensions pressenties

La découverte des données a permis d’identifier les dimensions pertinentes pour le Data Warehouse.

Les dimensions principales sont :

| Dimension       | Colonnes sources                        |
| --------------- | --------------------------------------- |
| `dim_date`      | `date`, `year`                          |
| `dim_location`  | `name`, `country`, `city`, `lat`, `lon` |
| `dim_dataset`   | `dataset_id`, `dataset_name`            |
| `dim_matrix`    | `matrix`                                |
| `dim_unit`      | `unit`                                  |
| `dim_substance` | informations issues de `pfas_values`    |

Ces dimensions permettront de filtrer, regrouper et analyser les données PFAS selon plusieurs axes.

---

## 23. Difficultés identifiées

La phase de découverte a mis en évidence plusieurs difficultés :

1. Le dataset mélange plusieurs sources de données.
2. Les matrices environnementales sont différentes.
3. Les unités ne sont pas toutes comparables.
4. `pfas_sum` contient beaucoup de zéros.
5. `pfas_sum` contient des valeurs extrêmes très élevées.
6. `pfas_values` est une colonne semi-structurée.
7. Certaines valeurs de `pfas_values` sont des listes vides.
8. Les dates peuvent être manquantes ou incomplètes.
9. Les coordonnées doivent être contrôlées.
10. Les conventions de codage peuvent varier selon les datasets.

Ces difficultés justifient la mise en place d’un pipeline ETL structuré.

---

## 24. Synthèse des résultats principaux

Les principaux résultats de la découverte des données sont les suivants :

| Élément analysé     | Résultat principal                               |
| ------------------- | ------------------------------------------------ |
| Volume              | environ 942 084 lignes                           |
| Colonne clé         | `pfas_values` contient les détails par substance |
| `pfas_sum`          | somme globale des PFAS par observation           |
| Valeurs extrêmes    | présentes et concentrées dans certains groupes   |
| Valeurs nulles      | environ 60,31 % de `pfas_sum = 0`                |
| `pfas_values` vide  | environ 1,34 % sans détail exploitable           |
| Unités              | plusieurs unités non directement comparables     |
| Matrices            | eau, sol, sédiments, biote, etc.                 |
| Décision principale | analyser par groupes homogènes                   |

---

## 25. Conclusion

Le notebook `01_data_discovery.ipynb` a permis de comprendre la structure générale du dataset PFAS et d’identifier les principales difficultés de qualité et d’interprétation.

L’analyse a montré que le dataset est riche, mais hétérogène. Il mélange plusieurs sources, plusieurs matrices environnementales, plusieurs unités et plusieurs conventions de mesure.

La colonne `pfas_sum` donne une information globale utile, mais elle doit être interprétée avec prudence en raison de la présence de nombreux zéros et de valeurs extrêmes.

La colonne `pfas_values` est centrale pour le projet, car elle contient les détails par substance PFAS. Sa transformation en format long est donc indispensable pour construire une table de faits détaillée.

Cette phase de découverte a permis de définir les principales règles du pipeline STAGING :

* filtrer les observations de type `Measurement` ;
* nettoyer `pfas_values` ;
* transformer les détails PFAS en format long ;
* standardiser les unités et matrices ;
* contrôler les coordonnées ;
* créer des indicateurs qualité ;
* préparer les futures tables de faits et dimensions.

La prochaine étape du projet est donc la construction du notebook :

```text
02_staging_pipeline.ipynb
```

Ce notebook aura pour objectif de produire deux tables propres et exploitables :

```text
pfas_observations_staging.parquet
pfas_measurements_long_staging.parquet
```

Ces deux tables constitueront la base de la couche PROCESSED et du futur Data Warehouse PostgreSQL.
