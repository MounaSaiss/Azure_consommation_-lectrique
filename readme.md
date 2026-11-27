# Prévision de la consommation électrique – Pipeline end-to-end sur Azure

Projet de prévision de la consommation électrique horaire (jeu de données PJME) : du stockage des données au déploiement d'un modèle de Machine Learning sur Microsoft Azure.

## Ressources Azure utilisées

| Ressource | Nom | Rôle |
|---|---|---|
| Groupe de ressources | `MounaSA` | Regroupe toutes les ressources (North Europe) |
| Compte de stockage (ADLS Gen2) | `mlazuredatalake26` | Data Lake, conteneur `raw` |
| Azure Data Factory | `statsmoun` | Ingestion des données |
| Workspace Azure ML | `mlw-pjme` | Datastore, data asset, compute, pipelines |
| Compute cluster | `cpu-cluster-pjme` | Entraînement (Standard_D4s_v3, 0 à 1 nœud) |

---

## Phase 2 : Mise en place du pipeline sur Microsoft Azure

### Étape 1 : Ingestion des données

- [x] Créer un pipeline dans Azure Data Factory (`PL_Copy_Source_To_Raw`).
- [x] Configurer une activité « Copy Data » pour charger les fichiers CSV depuis la source.
- [x] Valider le format et la structure des fichiers (CSV, séparateur virgule, UTF-8, première ligne comme en-tête).
- [x] Stocker les données dans Azure Data Lake Storage Gen2 (`mlazuredatalake26`).

### Étape 2 : Organisation du Data Lake

- [x] Créer le dossier `/raw/YYYY/MM/DD/` pour stocker les données brutes non modifiées avec horodatage.
  - Chemin obtenu : `raw/2026/11/26/pjme-hourly.csv` (généré par une expression dynamique sur la date d'exécution).

### Étape 3 : Configuration d'Azure Machine Learning

- [x] Créer ou accéder au workspace Azure Machine Learning (`mlw-pjme`, North Europe).
- [x] Configurer un datastore pointant vers le dossier `/raw/` du Data Lake (`datastore_pjme`, conteneur `raw`).
- [x] Créer un compute cluster pour l'entraînement en respectant la configuration demandée.
  - `cpu-cluster-pjme` : CPU dédié, `Standard_D4s_v3`, min 0 nœud, max 1 nœud, 120 s d'inactivité avant réduction.

### Étape 4 : Création du dataset

- [x] Dans Azure ML Studio, accéder à Data → Data assets.
- [x] Créer un data asset à partir du datastore configuré (`pjme_hourly_data`, type `uri_folder`, version 1).
- [x] Sélectionner les fichiers CSV présents dans `/raw/`.

### Étape 5 : Création du pipeline dans Designer

- [x] Ouvrir Azure ML Designer et créer un nouveau pipeline (`PL_PJME_Forecasting`).
- [x] Ajouter le dataset créé dans le canvas (`pjmehourlydata`).

### Étape 6 : Nettoyage des données

- [x] Ajouter le module « Summarize Data » pour visualiser les statistiques descriptives.
- [x] Ajouter le module « Clean Missing Data » si nécessaire.
- [x] Choisir une stratégie adaptée aux données : suppression des lignes, remplacement par une valeur statistique ou autre méthode pertinente.
- [x] Ajouter le module « Remove Duplicate Rows » pour traiter les doublons identifiés.
- [x] Ajouter « Edit Metadata » pour corriger les types de données, notamment celui de la date.

> Ordre des modules dans le pipeline : `pjmehourlydata` → Clean Missing Data → Remove Duplicate Rows → Edit Metadata → Summarize Data.

### Étape 7 : Feature Engineering

- [x] Ajouter le module « Execute Python Script ».
- [x] Rédiger un script pour créer des variables temporelles pertinentes : `Hour`, `DayOfWeek`, `Month`, `Quarter`, `Year` et `DayOfYear`.

```python
import pandas as pd

def azureml_main(dataframe1=None, dataframe2=None):
    df = dataframe1.copy()
    df['Datetime'] = pd.to_datetime(df['Datetime'])
    df = df.sort_values('Datetime').reset_index(drop=True)
    df['Hour'] = df['Datetime'].dt.hour
    df['DayOfWeek'] = df['Datetime'].dt.dayofweek
    df['Month'] = df['Datetime'].dt.month
    df['Quarter'] = df['Datetime'].dt.quarter
    df['Year'] = df['Datetime'].dt.year
    df['DayOfYear'] = df['Datetime'].dt.dayofyear
    return df,
```

### Étape 8 : Préparation pour l'entraînement

- [x] Ajouter « Select Columns in Dataset » pour sélectionner les colonnes pertinentes.
- [x] Ajouter « Split Data » pour séparer les données selon l'ordre chronologique : 80 % pour l'entraînement et 20 % pour le test.
- [ ] Vérifier que la séparation respecte effectivement la chronologie des observations.

> À vérifier : dans les paramètres de `Split Data`, le mode de découpage doit être **non aléatoire** (pas de « Randomized split ») et les données doivent rester triées par `Datetime` (c'est le rôle du `sort_values` du script Python). Sinon, des observations futures se retrouvent dans le jeu d'entraînement.

### Étape 9 : Entraînement du modèle

- [x] Choisir un algorithme de régression adapté à l'objectif de prévision (**Linear Regression**).
- [x] Ajouter le module « Train Model » pour entraîner le modèle.
- [x] Ajouter « Score Model » pour générer les prédictions sur les données de test.

### Étape 10 : Évaluation du modèle

- [x] Ajouter « Evaluate Model » pour évaluer les performances du modèle.
- [x] Interpréter les résultats à l'aide des métriques disponibles et identifier les limites du modèle.

| Métrique | Valeur |
|---|---|
| Coefficient de détermination (R²) | 0,2862 |
| Mean Absolute Error (MAE) | 4 192,67 |
| Root Mean Squared Error (RMSE) | 5 443,07 |
| Relative Absolute Error (RAE) | 0,8445 |
| Relative Squared Error (RSE) | 0,7138 |

**Limites du modèle :**

- Une régression linéaire capte mal les cycles non linéaires (heure, jour, saison) : R² ≈ 0,29, soit environ 29 % de la variance expliquée.
- Les variables calendaires sont numériques (23 h et 0 h sont traitées comme éloignées).
- Aucune variable de retard (consommation à t-1, t-24, t-168) ni facteur externe (météo, jours fériés).

**Pistes d'amélioration :** modèles non linéaires (Boosted Decision Tree, Decision Forest), variables de retard, encodage cyclique sin/cos, jours fériés, température.

### Étape 11 : Enregistrement du modèle

- [x] Cliquer sur le module « Train Model » et enregistrer le modèle entraîné dans Azure Machine Learning.
- [ ] Renseigner le nom du modèle, par exemple `forecasting_model_v1`.

> Le modèle a été enregistré depuis la sortie du job (`trained_model_outputs` : `model_spec.yaml`, `score.py`). Pense à utiliser un nom explicite comme `forecasting_model_v1` à l'étape « Model settings ».

### Étape 12 : Création du pipeline d'inférence

- [x] Convertir le training pipeline en inference pipeline : « Create inference pipeline » → « Real-time inference pipeline ».
- [ ] Le Designer génère automatiquement un pipeline simplifié : supprimer les modules d'entraînement et conserver les étapes nécessaires à l'inférence, notamment l'importation des données, le feature engineering et « Score Model ».

Pipeline d'inférence généré (`…-real time inference`) :

```
TD-… (transformation) + pjmehourlydata → Apply Transformation → Remove Duplicate Rows
→ Edit Metadata → Execute Python Script → Select Columns in Dataset
→ Score Model (avec le modèle MD-…) → Web Service Output
```

> Le module `Evaluate Model` est encore présent dans le pipeline d'inférence : il peut être supprimé pour ne conserver que l'importation des données, le feature engineering, `Score Model` et `Web Service Output`.

---

## Suite du projet

1. Finaliser le pipeline d'inférence (supprimer `Evaluate Model`) puis le soumettre.
2. Déployer le pipeline sur un endpoint temps réel.
3. Tester l'endpoint avec un échantillon de données.
4. Itérer sur le modèle (voir pistes d'amélioration).

## Bonnes pratiques

- **Coûts** : cluster avec 0 nœud minimum, supprimer les endpoints et ressources inutilisés en fin de projet.
- **Sécurité** : remplacer la clé de compte par une identité managée / Microsoft Entra ID avec des rôles RBAC limités, et restreindre l'accès au réseau public du stockage.