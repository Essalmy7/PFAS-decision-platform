# Rapport — Phase 4 : Création du Data Warehouse PostgreSQL et des vues analytiques

## 1. Objectif de la phase

Cette phase a pour objectif de passer d’une couche `PROCESSED`, stockée sous forme de fichiers Parquet, à une vraie base de données décisionnelle PostgreSQL.

Les phases précédentes ont permis de construire des tables propres et structurées :

* dimensions ;
* tables de faits ;
* clés techniques ;
* indicateurs qualité.

La phase actuelle consiste donc à charger ces tables dans PostgreSQL afin de préparer l’exploitation SQL et la connexion future à Metabase.

L’objectif n’est pas seulement de stocker les données, mais de mettre en place une architecture décisionnelle propre, contrôlée et évolutive.

---

## 2. Position de cette phase dans l’architecture globale

L’architecture globale du projet est organisée comme suit :

```text
PFAS Data Hub
↓
RAW
↓
STAGING
↓
PROCESSED
↓
PostgreSQL Data Warehouse
↓
Vues analytiques MART
↓
Metabase
```

Les trois premières couches ont été construites en Python.
La couche PostgreSQL permet maintenant de passer à une logique plus proche d’un système décisionnel réel.

Cette étape est importante car elle permet :

* de centraliser les données structurées ;
* d’interroger les données avec SQL ;
* de séparer les données techniques des vues analytiques ;
* de préparer les tableaux de bord Metabase ;
* de rendre le projet plus robuste et plus professionnel.

---

## 3. Création de la base PostgreSQL

Une base de données PostgreSQL dédiée au projet a été créée :

```text
pfas_dw
```

Le choix de créer une base dédiée permet d’isoler le projet PFAS des autres bases de données éventuelles.
Cela facilite la maintenance, les tests, la documentation et le futur déploiement.

La connexion entre Python et PostgreSQL a été testée avec succès.
Le test de connexion a confirmé que le serveur PostgreSQL était accessible et opérationnel.

Résultat obtenu :

```text
PostgreSQL 18.2 on x86_64-windows, compiled by msvc-19.44.35222, 64-bit
```

Ce test est important car il valide la communication entre l’environnement Python du projet et la base de données PostgreSQL.

---

## 4. Création des schémas `dw` et `mart`

Deux schémas ont été créés dans PostgreSQL :

```text
dw
mart
```

### 4.1 Schéma `dw`

Le schéma `dw` contient les tables du Data Warehouse :

* dimensions ;
* tables de faits.

Il représente la couche structurée et fiable du système décisionnel.

Les tables chargées dans ce schéma sont :

```text
dw.dim_date
dw.dim_dataset
dw.dim_matrix
dw.dim_unit
dw.dim_location
dw.dim_substance
dw.fact_pfas_observation
dw.fact_pfas_measurement
```

### 4.2 Schéma `mart`

Le schéma `mart` contient les vues analytiques destinées à faciliter l’exploitation dans Metabase.

Ce schéma ne stocke pas directement les données brutes.
Il contient plutôt des vues SQL préparées, qui simplifient les futures analyses.

Cette séparation est volontaire.

Elle permet de distinguer :

* la couche de stockage décisionnel : `dw` ;
* la couche d’analyse métier : `mart`.

Ce choix rend l’architecture plus claire et plus maintenable.

---

## 5. Pourquoi ne pas tout mettre dans un seul schéma ?

Il aurait été possible de mettre toutes les tables et toutes les vues dans un seul schéma.
Cependant, ce choix aurait rendu l’architecture moins lisible.

En séparant `dw` et `mart`, on distingue clairement deux responsabilités.

| Schéma | Rôle                                                |
| ------ | --------------------------------------------------- |
| `dw`   | Stocker les tables décisionnelles propres           |
| `mart` | Préparer les indicateurs et les vues pour l’analyse |

Cette séparation est plus professionnelle car elle correspond à une logique fréquente dans les projets BI et Data Warehouse.

Elle permet aussi d’éviter que Metabase interroge directement des tables complexes ou trop techniques.

---

## 6. Tables chargées dans le schéma `dw`

Les fichiers Parquet de la couche PROCESSED ont été chargés dans PostgreSQL.

Les dimensions chargées sont :

```text
dim_date
dim_dataset
dim_matrix
dim_unit
dim_location
dim_substance
```

Les tables de faits chargées sont :

```text
fact_pfas_observation
fact_pfas_measurement
```

Ce chargement permet de transformer les fichiers locaux en vraies tables relationnelles interrogeables en SQL.

---

## 7. Justification de l’ordre de chargement

Les dimensions ont été chargées avant les tables de faits.

Cet ordre est logique dans un modèle décisionnel.

Les tables de faits contiennent des clés étrangères vers les dimensions.
Il est donc préférable que les dimensions existent déjà avant de charger les faits.

L’ordre de chargement retenu est :

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

Cette méthode montre que le chargement n’a pas été fait de manière aléatoire, mais en respectant la structure du modèle décisionnel.

---

## 8. Méthode de chargement progressive

Le chargement a été réalisé progressivement.

Ce choix est important car la table `fact_pfas_measurement` est très volumineuse.
Elle contient plus de 21 millions de lignes.

Charger une telle table en une seule fois peut provoquer :

* des erreurs mémoire ;
* un ralentissement important ;
* un blocage du notebook ;
* une perte de contrôle sur le suivi du chargement.

Pour éviter cela, les données ont été chargées par lots.

Cette méthode permet :

* de contrôler progressivement le chargement ;
* de limiter l’utilisation mémoire ;
* de suivre le nombre de lignes chargées ;
* d’identifier plus facilement une erreur en cas de problème.

Le chargement progressif est donc un choix technique prudent et adapté au volume réel des données.

---

## 9. Contrôle des volumes chargés

Après le chargement, les volumes PostgreSQL ont été comparés aux volumes des fichiers Parquet locaux.

Ce contrôle permet de vérifier que les données chargées dans PostgreSQL correspondent bien aux données produites dans la couche PROCESSED.

L’objectif est d’éviter les situations suivantes :

* table partiellement chargée ;
* perte de lignes ;
* duplication involontaire ;
* erreur silencieuse pendant le chargement.

Cette vérification est importante car, dans un projet décisionnel, un dashboard n’a de valeur que si les données de base sont correctement chargées.

---

## 10. Création des index

Après le chargement des tables, des index ont été créés sur les principales clés des tables de faits.

Les index ont notamment été ajoutés sur :

* `date_id` ;
* `location_id` ;
* `dataset_dw_id` ;
* `matrix_id` ;
* `unit_id` ;
* `substance_id` ;
* `observation_id`.

Les index servent à accélérer les requêtes SQL, en particulier les jointures entre les tables de faits et les dimensions.

Ce choix est important car les futures analyses dans Metabase utiliseront beaucoup de requêtes de type :

```text
table de faits + dimensions
```

Sans index, certaines requêtes pourraient devenir lentes, surtout avec une table de faits détaillée contenant plus de 21 millions de lignes.

La création des index montre donc que la phase PostgreSQL ne s’est pas limitée au chargement des données.
Elle a aussi pris en compte les performances futures du système.

---

## 11. Création des vues analytiques dans le schéma `mart`

Après le chargement des tables dans `dw`, des vues analytiques ont été créées dans le schéma `mart`.

Les vues déjà créées sont :

```text
mart.vw_overview_kpis
mart.vw_pfas_by_country
mart.vw_pfas_by_matrix
```

Ces vues permettent de préparer les premiers indicateurs nécessaires à l’analyse et aux futurs dashboards Metabase.

L’intérêt des vues est de simplifier les requêtes.
Au lieu de refaire les mêmes jointures et agrégations dans chaque graphique, les calculs principaux sont centralisés dans PostgreSQL.

---

## 12. Vue `mart.vw_overview_kpis`

### Objectif

La vue `vw_overview_kpis` donne une vision globale du projet.

Elle permet de calculer les indicateurs principaux :

* nombre total d’observations ;
* nombre total de mesures détaillées ;
* nombre total de sites ;
* nombre total de substances ;
* nombre total de sources ;
* taux d’observations positives ;
* taux d’observations nulles ;
* valeur maximale de `pfas_sum`.

### Justification

Cette vue est utile pour créer une première page de dashboard.

Elle répond à la question :

```text
Quelle est la taille et la qualité générale du jeu de données ?
```

Elle permet aussi à l’encadrant ou à l’utilisateur final de comprendre immédiatement le périmètre des données.

---

## 13. Vue `mart.vw_pfas_by_country`

### Objectif

La vue `vw_pfas_by_country` agrège les observations par pays.

Elle permet d’analyser :

* le nombre d’observations par pays ;
* le nombre d’observations positives ;
* le taux de positivité ;
* la moyenne de `pfas_sum` ;
* la médiane de `pfas_sum` ;
* la valeur maximale observée.

### Justification

Cette vue est importante car la localisation est un axe central dans l’analyse environnementale.

Elle permet de répondre à des questions comme :

* quels pays sont les plus représentés dans les données ?
* quels pays présentent le plus de mesures positives ?
* quels pays ont les concentrations globales les plus élevées ?
* existe-t-il des différences importantes entre territoires ?

Le choix de calculer à la fois la moyenne, la médiane et le maximum est volontaire.

La moyenne peut être influencée par les valeurs extrêmes.
La médiane donne une mesure plus robuste du niveau typique.
Le maximum permet d’identifier les situations les plus critiques.

---

## 14. Vue `mart.vw_pfas_by_matrix`

### Objectif

La vue `vw_pfas_by_matrix` agrège les observations par matrice environnementale et par unité.

Elle permet d’analyser les PFAS selon le type de milieu :

* eaux souterraines ;
* eaux de surface ;
* eaux usées ;
* eau potable ;
* sol ;
* sédiments ;
* biote ;
* autres matrices.

### Justification

Cette vue est essentielle car les concentrations PFAS ne doivent pas être analysées globalement sans tenir compte de la matrice.

Une concentration exprimée en `ng/L` dans l’eau ne doit pas être comparée directement avec une concentration exprimée en `ng/kg` dans le sol.

Pour cette raison, la vue regroupe les données par :

```text
matrix_standardized + unit_standardized
```

Ce choix montre que l’analyse respecte la nature des données environnementales et évite les comparaisons incorrectes.

---

## 15. Gestion consciente d’un problème rencontré

Lors de la création d’une vue, une erreur est apparue car la colonne suivante n’existait pas dans la table PostgreSQL :

```text
is_extreme_pfas_sum
```

Au lieu de forcer la requête ou de supposer que la colonne existait, la structure réelle de la table a été vérifiée.

La vue a ensuite été corrigée en supprimant temporairement l’indicateur des valeurs extrêmes.

Cette correction montre une démarche rigoureuse :

1. identifier précisément l’erreur ;
2. comprendre son origine ;
3. vérifier la structure réelle de la table ;
4. adapter la requête SQL à l’état réel de la base ;
5. continuer avec une vue cohérente.

Cette approche est importante dans un projet Data, car il ne faut pas construire des indicateurs sur des colonnes supposées, mais sur des données réellement disponibles.

L’indicateur des valeurs extrêmes pourra être réintégré ensuite de manière plus propre, soit en ajoutant la colonne dans la table de faits, soit en recalculant les extrêmes directement dans une vue SQL.

---

## 16. Vues complémentaires prévues

Les vues actuellement créées constituent une première base pour le dashboard.

Des vues complémentaires sont prévues pour enrichir l’analyse :

```text
mart.vw_pfas_by_year
mart.vw_top_substances
mart.vw_extreme_observations
```

### `mart.vw_pfas_by_year`

Cette vue permettra d’analyser l’évolution des observations dans le temps.

Elle servira à répondre aux questions suivantes :

* comment évolue le nombre de mesures par année ?
* le taux de positivité augmente-t-il ou diminue-t-il ?
* certaines années présentent-elles des valeurs plus élevées ?

### `mart.vw_top_substances`

Cette vue exploitera la table détaillée `fact_pfas_measurement`.

Elle permettra d’analyser :

* les substances les plus fréquentes ;
* les substances les plus souvent positives ;
* les substances ayant les concentrations les plus élevées.

Cette vue est importante car l’analyse des PFAS ne doit pas se limiter à une somme globale.
Il faut aussi comprendre le comportement des substances individuelles.

### `mart.vw_extreme_observations`

Cette vue permettra d’identifier les observations les plus atypiques ou les plus élevées.

Elle pourra être construite en recalculant les extrêmes directement en SQL, par exemple avec un percentile 99 par groupe :

```text
matrix_id + unit_id
```

Cette approche est préférable à un seuil global, car elle respecte les différences entre matrices et unités.

---

## 17. Pourquoi créer des vues au lieu d’utiliser directement les tables ?

Les tables du schéma `dw` sont utiles pour le stockage décisionnel, mais elles sont parfois trop techniques pour une utilisation directe dans un outil BI.

Les vues du schéma `mart` permettent :

* de simplifier les jointures ;
* de préparer les agrégations ;
* de rendre les indicateurs plus lisibles ;
* de limiter les erreurs dans Metabase ;
* de centraliser la logique métier dans SQL ;
* de faciliter la maintenance.

Par exemple, au lieu de refaire dans Metabase une jointure entre `fact_pfas_observation`, `dim_location`, `dim_matrix` et `dim_unit`, on peut utiliser directement une vue prête.

C’est un choix de conception qui améliore la clarté et la fiabilité du dashboard.

---

## 18. Cohérence avec le modèle décisionnel

Cette phase respecte le modèle construit dans la couche PROCESSED.

Les tables de faits restent dans `dw` :

```text
fact_pfas_observation
fact_pfas_measurement
```

Les dimensions restent dans `dw` :

```text
dim_date
dim_location
dim_dataset
dim_matrix
dim_unit
dim_substance
```

Les vues analytiques sont créées dans `mart`.

Ce découpage respecte une logique claire :

```text
dw = données structurées
mart = données prêtes pour l’analyse
```

Il permet de construire progressivement une architecture décisionnelle complète.

---

## 19. Apport de cette phase pour Metabase

Cette phase prépare directement la connexion à Metabase.

Une fois Metabase connecté à PostgreSQL, les vues du schéma `mart` pourront être utilisées pour construire les premiers dashboards.

Les indicateurs déjà préparés permettront de créer :

* une page de synthèse globale ;
* une analyse par pays ;
* une analyse par matrice ;
* une analyse temporelle ;
* une analyse par substance ;
* une analyse des valeurs extrêmes.

Cette organisation permettra d’obtenir un dashboard plus clair, car les calculs importants seront déjà préparés côté PostgreSQL.

---

## 20. Limites actuelles

Certaines limites restent à traiter dans les prochaines étapes.

Premièrement, toutes les vues analytiques ne sont pas encore finalisées.
Les vues temporelles, substances et valeurs extrêmes devront être créées ou complétées.

Deuxièmement, les seuils réglementaires ne sont pas encore intégrés.
Il n’est donc pas encore possible de produire une analyse officielle des dépassements.

Troisièmement, la gestion des valeurs extrêmes doit être approfondie.
Il faudra choisir une méthode robuste, idéalement par groupe de matrice et d’unité.

Enfin, les vues créées constituent une première couche analytique.
Elles pourront évoluer selon les besoins de l’encadrant et les objectifs du dashboard.

---

## 21. Conclusion

Cette phase a permis de transformer les fichiers PROCESSED en un véritable Data Warehouse PostgreSQL.

Les tables décisionnelles ont été chargées dans le schéma `dw`, en respectant l’ordre logique de chargement : dimensions d’abord, tables de faits ensuite.

Des index ont été créés pour améliorer les performances futures des requêtes, notamment dans les jointures entre faits et dimensions.

Un schéma `mart` a également été créé pour accueillir les vues analytiques destinées à Metabase.

Les premières vues créées sont :

```text
mart.vw_overview_kpis
mart.vw_pfas_by_country
mart.vw_pfas_by_matrix
```

Ces vues permettent déjà de produire une analyse globale, une analyse géographique et une analyse par matrice environnementale.

Cette phase montre que le projet avance vers une architecture décisionnelle complète, structurée et exploitable.

La prochaine étape consiste à finaliser les vues analytiques restantes, puis à connecter Metabase à PostgreSQL afin de construire les premiers tableaux de bord décisionnels.
