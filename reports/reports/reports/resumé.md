# Fiche récapitulative — Avancement du projet PFAS

## De la découverte des données à la modélisation PROCESSED du Data Warehouse

## 1. Objectif général du travail réalisé

L’objectif de cette première partie du projet est de transformer des données brutes issues du PFAS Data Hub en une structure décisionnelle propre, organisée et exploitable pour un futur Data Warehouse PostgreSQL.

Le travail a été organisé en trois couches successives :

```text
RAW / Découverte
↓
STAGING / Nettoyage et préparation
↓
PROCESSED / Modélisation décisionnelle
↓
PostgreSQL / Data Warehouse
↓
Metabase / Tableaux de bord
```

Cette organisation permet de ne pas passer directement des données brutes au dashboard. Elle garantit une meilleure traçabilité, une meilleure qualité des données et une modélisation plus robuste.

---

## 2. Couche 1 — Découverte des données

### Objectif

La première étape a consisté à comprendre le dataset avant toute transformation.

L’objectif était d’identifier :

* le volume des données ;
* les colonnes importantes ;
* les valeurs manquantes ;
* les unités disponibles ;
* les matrices environnementales ;
* le comportement de `pfas_sum` ;
* la structure de `pfas_values` ;
* les problèmes à traiter dans le pipeline ETL.

### Travail réalisé

J’ai commencé par charger le dataset brut, puis j’ai analysé sa structure générale :

* nombre de lignes et de colonnes ;
* types de données ;
* liste des colonnes ;
* dictionnaire de données ;
* taux de valeurs manquantes ;
* nombre de valeurs distinctes par colonne.

Un dictionnaire de données a été généré dans le dossier `reports`, afin de documenter proprement la structure initiale du dataset.

### Résultats importants identifiés

Le dataset contient environ :

```text
942 084 lignes
```

Les colonnes clés identifiées sont notamment :

* `dataset_name` : source des données ;
* `country`, `city`, `lat`, `lon` : informations géographiques ;
* `year`, `date` : informations temporelles ;
* `matrix` : milieu environnemental ;
* `unit` : unité de mesure ;
* `pfas_sum` : somme globale des PFAS ;
* `pfas_values` : détail des substances PFAS.

### Analyse de `pfas_sum`

La colonne `pfas_sum` représente la somme globale des concentrations PFAS pour une observation.

J’ai identifié deux points importants :

1. **Présence de valeurs extrêmes**
   Certaines valeurs de `pfas_sum` sont très élevées. Elles ne sont pas réparties aléatoirement, mais concentrées dans certains datasets, matrices et unités.

2. **Présence importante de valeurs nulles**
   Le taux global de `pfas_sum = 0` est d’environ :

```text
60,31 %
```

Ces zéros ne doivent pas être interprétés automatiquement comme une absence totale de contamination. Ils peuvent correspondre à des non-détections, des valeurs sous limite de détection ou des conventions propres aux datasets sources.

### Analyse de `pfas_values`

La colonne `pfas_values` est centrale, car elle contient le détail des substances PFAS mesurées.

Cependant, cette colonne est semi-structurée : plusieurs substances peuvent être stockées dans une seule cellule.

Exemple :

```text
[
  {"substance": "PFOA", "value": 90, "unit": "ng/l"},
  {"substance": "PFOS", "value": 40, "unit": "ng/l"}
]
```

Cette structure n’est pas directement adaptée à un Data Warehouse. Elle doit donc être transformée en format long.

Après nettoyage des listes vides `[]`, le taux de lignes sans détail exploitable est faible, environ :

```text
1,34 %
```

Cela confirme que `pfas_values` est majoritairement exploitable pour construire une table de mesures détaillées.

### Conclusion de la couche découverte

La découverte des données a montré que le dataset est riche, mais hétérogène.

Les principales difficultés identifiées sont :

* plusieurs sources de données ;
* plusieurs matrices environnementales ;
* plusieurs unités de mesure ;
* beaucoup de zéros dans `pfas_sum` ;
* présence de valeurs extrêmes ;
* données semi-structurées dans `pfas_values`.

Cette étape a justifié la construction d’un pipeline ETL structuré.

---

## 3. Couche 2 — STAGING : nettoyage et préparation

### Objectif

La couche STAGING a pour objectif de transformer les données brutes en tables propres, standardisées et exploitables.

Cette couche ne supprime pas agressivement les données. Elle conserve la traçabilité tout en ajoutant des colonnes nettoyées, standardisées et des indicateurs qualité.

### Travail réalisé

À partir des données brutes, j’ai construit deux tables STAGING principales :

```text
pfas_observations_staging.parquet
pfas_measurements_long_staging.parquet
```

### Table 1 — `pfas_observations_staging`

Cette table conserve le niveau global de l’observation.

Son grain est :

```text
Une ligne = une observation environnementale globale
```

Elle contient :

* la source ;
* la localisation ;
* la date ou l’année ;
* la matrice ;
* l’unité ;
* `pfas_sum` ;
* les indicateurs qualité.

Des indicateurs ont été créés :

* `has_pfas_details` ;
* `is_missing_pfas_sum` ;
* `is_zero_pfas_sum` ;
* `is_positive_pfas_sum` ;
* `is_extreme_pfas_sum` ;
* `has_valid_coordinates`.

### Table 2 — `pfas_measurements_long_staging`

Cette table est obtenue en transformant `pfas_values` en format long.

Son grain est :

```text
Une ligne = une substance PFAS mesurée dans une observation
```

Résultat obtenu :

```text
929 442 observations exploitables
21 530 916 lignes détaillées après transformation
```

Cela signifie qu’une observation contient en moyenne plusieurs substances PFAS.

Le nombre moyen de substances par observation est d’environ :

```text
23,17 substances
```

Cette transformation est essentielle, car elle permet d’analyser les PFAS substance par substance.

### Nettoyage et standardisation réalisés

Dans la couche STAGING, plusieurs traitements ont été appliqués :

| Élément          | Traitement réalisé                                                               |
| ---------------- | -------------------------------------------------------------------------------- |
| `pfas_values`    | remplacement des listes vides par des valeurs manquantes dans une colonne propre |
| `pfas_sum`       | conversion numérique et création d’indicateurs                                   |
| `pfas_value`     | conversion numérique                                                             |
| `unit`           | standardisation des unités                                                       |
| `matrix`         | standardisation des matrices                                                     |
| `date` / `year`  | création de `date_clean` et `year_clean`                                         |
| `lat` / `lon`    | contrôle de validité des coordonnées                                             |
| `pfas_less_than` | création d’un indicateur booléen                                                 |

### Difficulté technique rencontrée

Lors de la transformation de `pfas_values`, une erreur mémoire est apparue lorsque l’opération `explode()` était appliquée directement à tout le dataset.

Pour résoudre ce problème, j’ai optimisé la méthode :

1. filtrer seulement les observations utiles ;
2. garder uniquement les colonnes nécessaires ;
3. supprimer les colonnes lourdes avant transformation ;
4. appliquer `explode()` sur une table de travail plus légère.

Cette correction montre que le pipeline n’est pas seulement logique, mais aussi adapté au volume réel des données.

### Conclusion de la couche STAGING

La couche STAGING a permis de passer d’un dataset brut hétérogène à deux tables propres :

* une table globale au niveau observation ;
* une table détaillée au niveau substance PFAS.

Cette séparation est essentielle pour éviter les erreurs d’analyse, notamment le double comptage de `pfas_sum`.

---

## 4. Couche 3 — PROCESSED : modélisation décisionnelle

### Objectif

La couche PROCESSED transforme les tables STAGING en un modèle décisionnel prêt pour le Data Warehouse PostgreSQL.

Cette couche répond directement à la consigne :

> Identifier les dimensions pertinentes, identifier plusieurs tables de faits intéressantes pour l’analyse des PFAS, puis synthétiser le tout dans un diagramme.

### Principe de modélisation

J’ai choisi une approche de type :

```text
Schéma en étoile évolutif vers un schéma en galaxie
```

Le modèle commence avec deux tables de faits principales :

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
dim_substance
```

Ce choix est pertinent car les analyses PFAS nécessitent deux niveaux d’analyse :

1. le niveau global de l’observation ;
2. le niveau détaillé de la substance.

---

## 5. Dimensions construites et justification

### 5.1 `dim_date`

Cette dimension permet d’analyser les données selon le temps.

Elle est nécessaire pour répondre à des questions comme :

* comment évoluent les mesures dans le temps ?
* quelles années contiennent le plus de mesures ?
* quelles périodes présentent des valeurs élevées ?

Elle gère deux cas :

* date complète disponible ;
* année seulement disponible.

### 5.2 `dim_location`

Cette dimension permet l’analyse géographique.

Elle contient :

* pays ;
* ville ;
* site ;
* latitude ;
* longitude ;
* validité des coordonnées.

Elle servira aux cartes et aux analyses par territoire dans Metabase.

### 5.3 `dim_dataset`

Cette dimension conserve la source des données.

Elle est importante parce que les données viennent de plusieurs datasets ayant potentiellement des conventions différentes.

Elle permet d’analyser :

* les volumes par source ;
* les valeurs extrêmes par dataset ;
* les taux de zéros par dataset ;
* la qualité des données selon la source.

### 5.4 `dim_matrix`

Cette dimension représente le milieu environnemental :

* eaux souterraines ;
* eaux de surface ;
* eaux usées ;
* eau potable ;
* sol ;
* sédiments ;
* biote ;
* lixiviat.

Elle est essentielle parce qu’une concentration dans l’eau n’est pas comparable directement à une concentration dans le sol.

### 5.5 `dim_unit`

Cette dimension permet de gérer les unités :

* `ng/L` ;
* `ng/kg` ;
* `ng/kg dw` ;
* `ng/kg fw`.

Elle évite les comparaisons incorrectes entre unités différentes.

### 5.6 `dim_substance`

Cette dimension permet d’analyser les PFAS substance par substance.

Elle contient :

* nom de la substance ;
* nom standardisé ;
* identifiant CAS ;
* isomère.

Elle est essentielle car le projet ne doit pas seulement analyser une somme globale, mais aussi comprendre quelles substances PFAS sont les plus présentes ou les plus critiques.

---

## 6. Tables de faits construites et justification

### 6.1 `fact_pfas_observation`

Cette table représente le niveau global.

Son grain est :

```text
Une ligne = une observation environnementale globale
```

Elle contient :

* `pfas_sum` ;
* indicateur de valeur nulle ;
* indicateur de valeur positive ;
* indicateur de valeur extrême ;
* indicateur de coordonnées valides ;
* clés vers les dimensions.

Elle permet de répondre à des questions comme :

* quels sites ont les plus grandes sommes PFAS ?
* quels pays ou matrices présentent les niveaux globaux les plus élevés ?
* quel est le taux de mesures positives ?
* où se situent les observations extrêmes ?

Cette table est adaptée aux KPI globaux.

### 6.2 `fact_pfas_measurement`

Cette table représente le niveau détaillé par substance.

Son grain est :

```text
Une ligne = une substance PFAS mesurée dans une observation
```

Elle contient :

* `pfas_value` ;
* indicateur de valeur positive ;
* indicateur de valeur nulle ;
* indicateur de valeur sous limite ;
* clé vers la substance ;
* clés vers les dimensions.

Elle permet de répondre à des questions comme :

* quelles substances sont les plus fréquentes ?
* quelles substances ont les concentrations les plus élevées ?
* quelles substances sont présentes dans chaque matrice ?
* quelles substances sont associées à des valeurs sous limite de détection ?

Cette table est adaptée aux analyses détaillées par substance.

---

## 7. Pourquoi deux tables de faits ?

Le choix de deux tables de faits est important.

Il existe deux niveaux d’information différents :

| Niveau                 | Table                   | Mesure principale |
| ---------------------- | ----------------------- | ----------------- |
| Observation globale    | `fact_pfas_observation` | `pfas_sum`        |
| Substance individuelle | `fact_pfas_measurement` | `pfas_value`      |

Ce choix évite une erreur importante : le double comptage de `pfas_sum`.

Après transformation en format long, une observation est répétée plusieurs fois, une fois par substance. Si on utilisait uniquement la table détaillée et qu’on sommait `pfas_sum`, on compterait plusieurs fois la même observation.

Donc :

* pour les analyses globales, on utilise `fact_pfas_observation` ;
* pour les analyses par substance, on utilise `fact_pfas_measurement`.

Cette séparation rend le modèle plus fiable.

---

## 8. Tables futures prévues

Le modèle est conçu pour évoluer vers un schéma en galaxie.

Trois tables de faits futures sont prévues :

```text
fact_exceedance
fact_anomaly
fact_risk_score
```

### `fact_exceedance`

Cette table permettra d’analyser les dépassements de seuils.

Elle pourra contenir :

* valeur mesurée ;
* seuil de référence ;
* écart au seuil ;
* ratio de dépassement.

### `fact_anomaly`

Cette table permettra d’identifier les valeurs atypiques.

Elle pourra contenir :

* score d’anomalie ;
* méthode utilisée ;
* indicateur d’anomalie.

### `fact_risk_score`

Cette table permettra de classer les sites ou territoires selon un score de risque.

Elle pourra contenir :

* score global ;
* classe de risque ;
* composante concentration ;
* composante fréquence ;
* composante diversité des substances ;
* composante anomalie.

Ces tables ne sont pas encore alimentées, mais elles sont prévues dans le modèle pour montrer que l’architecture est évolutive.

---

## 9. Schéma décisionnel proposé

Le modèle peut être résumé ainsi :

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

Évolution future :

```text
fact_exceedance
fact_anomaly
fact_risk_score
```

Ces futures tables partageront plusieurs dimensions avec les tables existantes.

Cela correspond à un schéma en galaxie.

---

## 10. Contrôles qualité réalisés dans la couche PROCESSED

Avant de considérer la couche PROCESSED comme terminée, plusieurs contrôles ont été réalisés :

* vérification de l’existence des fichiers ;
* vérification des volumes ;
* vérification de l’unicité des clés primaires ;
* vérification des clés étrangères ;
* vérification de la cohérence entre observations et mesures détaillées.

Une difficulté mémoire a également été rencontrée lors de certaines jointures. Pour éviter les jointures lourdes, j’ai utilisé des dictionnaires de correspondance avec `.map()` au lieu de `merge`.

Cette méthode est plus adaptée au volume important du dataset, notamment pour la table détaillée qui contient plus de 21 millions de lignes.

---

## 11. Fichiers produits

Les fichiers produits dans `data/processed` sont :

### Dimensions

```text
dim_date.parquet
dim_location.parquet
dim_dataset.parquet
dim_matrix.parquet
dim_unit.parquet
dim_substance.parquet
```

### Tables de faits

```text
fact_pfas_observation.parquet
fact_pfas_measurement.parquet
```

Ces fichiers sont maintenant prêts à être chargés dans PostgreSQL.

---

## 12. Message principal à défendre auprès de l’encadrant

Le travail réalisé ne se limite pas à un nettoyage de données.

Il met en place une vraie architecture décisionnelle :

1. la découverte a permis de comprendre les limites et la structure des données ;
2. la couche STAGING a permis de nettoyer, standardiser et transformer les données ;
3. la couche PROCESSED a permis de construire un modèle décisionnel clair avec dimensions et tables de faits.

Le choix des deux tables de faits principales est justifié par les deux niveaux d’analyse nécessaires :

* analyse globale avec `pfas_sum` ;
* analyse détaillée avec `pfas_value`.

Les dimensions choisies sont pertinentes car elles correspondent aux principaux axes d’analyse des PFAS :

* temps ;
* localisation ;
* source ;
* matrice ;
* unité ;
* substance.

Cette structure prépare directement :

* le chargement dans PostgreSQL ;
* la création de vues SQL ;
* la construction de dashboards Metabase ;
* les analyses futures : dépassements, anomalies, score de risque et recommandations.

---

## 13. Conclusion courte pour la présentation orale

J’ai structuré le projet en trois couches.
La première couche m’a permis de comprendre les données, notamment les valeurs extrêmes, les zéros et la structure de `pfas_values`.
La deuxième couche a permis de nettoyer et standardiser les données, puis de transformer `pfas_values` en format long.
La troisième couche a permis de construire un premier modèle décisionnel avec des dimensions et deux tables de faits principales.

Le modèle est volontairement séparé en deux niveaux :

* `fact_pfas_observation` pour l’analyse globale des observations avec `pfas_sum` ;
* `fact_pfas_measurement` pour l’analyse détaillée des substances avec `pfas_value`.

Ce choix évite le double comptage, améliore la lisibilité du modèle et prépare le passage vers PostgreSQL et Metabase.
