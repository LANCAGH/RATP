# Projet RATP 
## Hugo LANCA


## Contexte métier

**Besoin fictif** : Île-de-France Mobilités souhaite prioriser ses investissements en accessibilité PMR sur le réseau ferré (RER, métro, Transilien).L'enjeu est d'identifier quelles stations doivent être rénovées en premier, en croisant leur fréquentation réelle avec leur niveau d'accessibilité actuel.

**Périmètre** : Réseau ferré uniquement (hors bus et tram), données 2015-2024.

---

## 1. Architecture Medallion (Bronze / Silver / Gold)

**Décision** : Pipeline organisé en 3 couches de données sur AWS S3.

**Justification** : Séparer la donnée brute (Bronze), nettoyée (Silver) et métier (Gold) permet de retracer l'origine de chaque transformation, de rejouer une étape sans tout recalculer, et d'assurer la traçabilité — un standard industriel pour les lacs de données.

Structure du projet : 
```
s3://p-idfm-pipeline/
├── bronze/          ← données brutes, immuables
├── silver/          ← données nettoyées, typées, jointes
├── gold/            ← score de priorité PMR, prêt pour le dashboard
├── profiling/       ← rapports d'exploration de la donnée (librairie y-data)
├── quality_reports/ ← résultats des tests qualité à chaque run
├── athena_results/ ← résultats athena
└── pipeline_metadata/ ← checkpoints et métadonnées de run
```

---

## 1bis. IAM roles

**Conformité** : Le rôle IAM qui a été crée se base sur le principe de `least privilege`. Ci-dessous la table des services et rôles associés : 



---


## 2. Stack technique — AWS natif uniquement

**Décision** : Lambda (ingestion) + Glue (transformation) + S3 (stockage) + Athena (requêtage) + EventBridge (orchestration) + Power BI Desktop (visualisation).

**Justification** : Je me suis lancé le défi de réaliser le projet en 1 semaine et un budget de 0€ (Free Tier) : une stack AWS native permet de livrer un pipeline complet sans disperser l'effort. **dbt** aurait apporté de la gouvernance supplémentaire mais allongeait significativement le délai. Il est mentionné dans le README comme évolution naturelle.

---

## 3. Granularité de l'analyse — Zone de Correspondance (ZdC)

**Décision** : L'unité d'analyse est la ZdC (station multimodale), pas la ZdA (arrêt monomodal).

**Justification** : Sur le réseau ferré, la validation se fait à l'entrée de la station indépendamment de la ligne empruntée (source : documentation IDFM). Les validations sont donc physiquement impossibles à ventiler par ZdA — la ZdC est la granularité réelle des données.

**Jointure** :

```
validations.ID_REFA_LDA (ZdC)
    → référentiel Zones d'arrêts (table pivot ZdC ↔ ZdA)
    → score PMR = MIN(accessibilité de toutes les ZdA de la ZdC)
```

**Justification du MIN** : Une station est aussi peu accessible que son entrée la moins accessible. Le MIN reflète l'expérience réelle d'un voyageur PMR.

---

## 4. Score de priorité PMR — Ranking

**Décision** : Pas de formule normalisée. Classement direct par :

1. Fréquentation totale décroissante
2. Score PMR croissant (1 = moins accessible = prioritaire)

En SQL : 
```sql
RANK() OVER (ORDER BY total_validations DESC, score_pmr ASC)
```

**Justification** : L'objectif métier est d'identifier les stations à fort trafic et faible accessibilité. Un classement est plus lisible et actionnable pour un décideur qu'un score composite normalisé. La comparabilité temporelle est assurée en comparant le podium d'une année à l'autre.

---

## 5. Data Quality — Framework WARN / ERROR

**Décision** : Deux niveaux de sévérité sur tous les datasets entrants.

- **ERROR** : anomalie bloquante, pipeline arrêté, flag `pipeline_status=FAILED` écrit en S3
- **WARN** : anomalie non bloquante, pipeline continue, alerte dans le rapport qualité

**Justification** : Distinguer ce qui corrompt silencieusement les données (ERROR) de ce qui mérite surveillance sans bloquer (WARN). Exemple : une correspondance ZdC↔ZdA modifiée fausse toutes les jointures → ERROR. Un nouveau nom de station inconnu → WARN.

**Principaux contrôles par table** :

| Table | Contrôle | Sévérité |
| --- | --- | --- |
| NB_FER | JOUR hors période batch attendue | ERROR |
| NB_FER | Colonnes manquantes ou renommées | ERROR |
| NB_FER | NB_VALD < 0 | ERROR |
| NB_FER | Nouveau COD_STIF_ARRET inconnu | WARN |
| NB_FER | CATEGORIE_TITRE hors 7 modalités | WARN |
| Accessibilité | Score PMR hors [1-6] | ERROR |
| Accessibilité | Coordonnées hors bbox Île-de-France | ERROR |
| Accessibilité | Nouveau stop_name inconnu | WARN |
| Zones d'arrêts | Correspondance ZdC↔ZdA modifiée | ERROR |
| Zones d'arrêts | Cardinalité différente du batch précédent | WARN |

---

## 6. Data Profiling — Exploration et Monitoring

**Décision** : Deux types de rapports générés automatiquement

- **Profiling initial** (one-shot) : au moment de la première ingestion, pour comprendre les distributions et alimenter les règles qualité
- **Profiling continu** : généré à chaque run batch pour détecter les dérives dans le temps (data drift)

**Justification** : Le Data Quality Report répond à "est-ce que la donnée respecte mes règles ?" Le Data Profiling Report répond à "à quoi ressemble ma donnée ?".

---

## 7. Idempotence — Checkpoint par année

**Décision** : À chaque ingestion réussie, un fichier de métadonnées est écrit dans `pipeline_metadata/` avec l'année/semestre traité. Avant chaque run, le pipeline vérifie quels checkpoints existent et ne retraite que ce qui manque.

**Justification** : Un pipeline idempotent peut être relancé plusieurs fois sans corrompre les données. Si le backfill échoue sur l'année 2018, on relance uniquement 2018 sans réingérer 2015-2017.

---

## 8. Gestion des secrets

**Décision** : Les URLs et clés API sont stockées dans **AWS Systems Manager Parameter Store**, jamais dans le code ou sur GitHub.

**Justification** : Le Parameter Store est gratuit, natif AWS, et accessible à runtime par Lambda et Glue sans exposition dans le code source.

---

## 9. Monitoring

**Décision** : CloudWatch surveille l'état des jobs Glue et Lambdas. En cas d'échec, SNS envoie une notification email automatique.

**Justification** : Sans monitoring, un échec silencieux du pipeline peut passer inaperçu pendant des semaines. CloudWatch + SNS est entièrement dans le Free Tier et ne nécessite que quelques minutes de configuration.

---

## 10. Normalisation des identifiants historiques

**Décision** : Les fichiers 2015-2016 utilisent d'anciens identifiants `ID_REFA_LDA`. En couche Bronze ils sont conservés intacts (immuabilité). En couche Silver, la table de correspondance ancien ID → nouvel ID est appliquée pour garantir une clé cohérente sur 10 ans.

**Justification** : Modifier le Bronze contreviendrait au principe d'immuabilité. La normalisation en Silver permet des analyses temporelles cohérentes sans altérer la source.

---

## Évolutions futures (hors scope v1)

- **dbt** : remplacement de Glue pour les transformations Silver/Gold, apporte lineage, documentation et tests natifs
- **Infrastructure as Code** : AWS CDK pour recréer l'environnement from scratch en une commande
- **Tests unitaires** : pytest sur les fonctions Python de nettoyage et transformation

