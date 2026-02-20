# spark-ecommerce

# 🛒 Segmentation Client RFM avec PySpark

> Segmentation non supervisée et classification supervisée de clients e-commerce via une pipeline complète PySpark — de la donnée brute aux recommandations marketing actionnables.

---

## 📋 Vue d'ensemble

Ce projet analyse un dataset e-commerce (Online Retail) pour identifier des profils clients distincts et prédire leur potentiel de valeur. Il combine deux approches complémentaires :

- **Clustering (non supervisé)** — Bisecting K-Means sur variables RFM pour segmenter les clients en groupes comportementaux
- **Classification (supervisé)** — Gradient Boosted Trees pour prédire si un client est un "gros dépensier" (top 25%)

**Résultats clés :** 3 segments identifiés (Champions / Réguliers / Dormants), modèle GBT retenu avec AUC = 0.914, F1 = 0.865.

---

## 📁 Structure du projet

```
spark-ecommerce/
│
├── notebooks/
│   └── ProjetFinal.ipynb               # Notebook principal (pipeline complète)
│
└── README.md
```

> Le dataset `Online_Retail_CSV.csv` n'est pas versionné dans le repo (fichier lourd). Il est à placer localement dans un dossier `data/raw/` avant d'exécuter le notebook.

---

## 📊 Dataset

| Propriété | Valeur |
|---|---|
| Source | UCI Machine Learning Repository — Online Retail |
| Période | Décembre 2010 — Décembre 2011 |
| Lignes brutes | 541 909 |
| Lignes après nettoyage | 397 884 |
| Clients identifiés | 4 338 |
| Pays | 38 |
| Colonnes | InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country |

**Particularités :** forte dominance UK (23 494 / 25 900 factures), valeurs négatives dans Quantity et UnitPrice (retours, remboursements), 135 080 lignes sans CustomerID.

---

## ⚙️ Pipeline

### 1. Prétraitement

- Suppression des lignes sans `CustomerID` (135 080 lignes)
- Conversion `InvoiceDate` → timestamp, `UnitPrice` → double (séparateur décimal `,` → `.`)
- Filtrage `Quantity > 0` et `UnitPrice > 0`
- Création de la colonne `TotalAmount = Quantity × UnitPrice`

### 2. Calcul RFM

Variables calculées par client à partir de la date de référence (`2011-12-09`) :

| Variable | Définition | Sens |
|---|---|---|
| **Recency** | Jours depuis le dernier achat | Plus bas = meilleur |
| **Frequency** | Nombre de commandes distinctes | Plus haut = meilleur |
| **Monetary** | Chiffre d'affaires total (€) | Plus haut = meilleur |

Deux versions des features ont été testées : log-transformée + standardisée vs. standardisée seule.

### 3. Clustering — Bisecting K-Means

Tests sur k = 2 à 8, deux versions de features.

**Configuration retenue :** k = 3, données **non log-transformées** + standardisation.

> La transformation logarithmique compresse les écarts entre clients (notamment sur Monetary), ce qui réduit la séparation des clusters. Les données brutes standardisées donnent un silhouette score de **0.745** pour k=3, contre 0.38 avec la version log.

### 4. Classification — Prédiction "Gros Dépensier"

Cible : clients au-dessus du **75e percentile** de Monetary (seuil ≈ 1 624 €).

Features utilisées : `Recency`, `Frequency`, `cluster` (encodé one-hot).

Pondération de classe appliquée (×3 pour la classe minoritaire) pour corriger le déséquilibre 74% / 26%.

Split : 70% train / 30% test, seed=42.

---

## 📈 Résultats

### Segmentation RFM (k=3)

| Segment | Clients | Recency moy. | Frequency moy. | Monetary moy. | Part CA |
|---|---|---|---|---|---|
| **Champions** | 26 (0.6%) | 5 j | 66.4 | 85 904 € | ~25% |
| **Réguliers** | 3 205 (73.9%) | 40 j | 4.7 | 1 879 € | ~68% |
| **Dormants** | 1 107 (25.5%) | 244 j | 1.6 | 594 € | ~7% |

### Performance des modèles de classification

| Modèle | Accuracy | F1 | AUC-ROC | TP (Gros détectés) |
|---|---|---|---|---|
| Logistic Regression | 0.861 | 0.862 | 0.911 | 257 |
| Random Forest | 0.867 | 0.867 | 0.920 | 254 |
| **Gradient Boosted Trees** | 0.862 | 0.865 | **0.914** | **265** |

**Modèle retenu : GBT** — bien que le Random Forest ait un F1 légèrement supérieur, le GBT détecte davantage de vrais "Gros Dépensiers" (265 vs 254 TP) et génère moins de faux négatifs. Dans un contexte marketing, rater un client à forte valeur coûte plus cher qu'un faux positif.

---

## 🎯 Recommandations Marketing

### Par segment RFM

**Champions** (26 clients, 100% gros dépensiers)
→ Programme VIP, early access, invitations événements, communication personnalisée 1:1, programme parrainage premium.

**Réguliers** (3 205 clients, 32% gros dépensiers)
→ Upsell produits supérieurs, bundles pour augmenter le panier, emails personnalisés basés sur l'historique, programme de points.

**Dormants** (1 107 clients, 4.5% gros dépensiers)
→ Email de réactivation -30%, offre limitée 7 jours, sondage de sortie, archivage progressif des profils non réactivés.

### Par score prédictif GBT

| Segment | Score GBT | Part | Valeur moy. | Action |
|---|---|---|---|---|
| **Activation** | < 0.3 | 62% | 863 € | Promotions, relance |
| **Standard** | 0.3 – 0.7 | 15% | 1 358 € | Upsell ciblé |
| **Premium** | > 0.7 | 23% | 6 585 € | Fidélisation, offres exclusives |

**Opportunité identifiée :** 105 clients actuellement classés "non gros" ont un score GBT > 0.5 → cible prioritaire de conversion. Objectif : 25% convertis = ~26 nouveaux clients à forte valeur.

---

## 🚀 Pipeline de production recommandée

```
Données transactionnelles quotidiennes
        ↓
Calcul RFM automatisé (Spark)
        ↓
Scoring GBT en batch
        ↓
Segmentation marketing (Activation / Standard / Premium)
        ↓
Personnalisation des campagnes & automatisation CRM
```

---

## ⚠️ Limitations

- **Période limitée** (1 an) — possible biais saisonnier, pics de fin d'année sur-représentés
- **Variables disponibles** — uniquement transactionnelles ; absence de données produit, marge, canal d'acquisition, comportement web, SAV
- **Définition de la cible** — le seuil "gros dépensier" (top 25%) est arbitraire et influence directement les métriques
- **Data leakage potentiel** — le cluster est utilisé comme feature dans le modèle supervisé, bien qu'il soit construit indépendamment de la cible

---

## 🔧 Stack technique

| Composant | Outil |
|---|---|
| Traitement distribué | PySpark 3.x |
| Clustering | `pyspark.ml.clustering.BisectingKMeans` |
| Preprocessing | `VectorAssembler`, `StandardScaler`, `OneHotEncoder` |
| Classification | `LogisticRegression`, `RandomForestClassifier`, `GBTClassifier` |
| Évaluation | `BinaryClassificationEvaluator`, `MulticlassClassificationEvaluator`, `sklearn.metrics` |
| Visualisation | Matplotlib, Seaborn |
| Environnement | Spark local (`local[*]`) |

---

## 📦 Installation

```bash
# Cloner le repo
git clone https://github.com/Haylize/spark-ecommerce.git
cd spark-ecommerce

# Installer les dépendances Python
pip install pyspark pandas numpy matplotlib seaborn scikit-learn

# Placer le dataset dans le bon répertoire
mkdir -p data/raw
# → copier Online_Retail_CSV.csv dans data/raw/

# Lancer le notebook
jupyter notebook notebooks/ProjetFinal.ipynb
```

Le dataset `Online_Retail_CSV.csv` est disponible sur le [UCI Machine Learning Repository](https://archive.ics.uci.edu/ml/datasets/online+retail).

---

## 📌 Pistes d'amélioration

- Historique plus long et données multi-périodes pour limiter le biais saisonnier
- Enrichissement des features : panier moyen, diversité produits, ancienneté client, délai entre commandes, taux de retour
- Tuning des hyperparamètres via `CrossValidator` + `ParamGridBuilder`
- Calibration des probabilités pour ajuster le seuil de classification selon le coût métier (FN vs FP)
- Mise en production avec suivi du drift et ré-entraînement régulier
- A/B test des actions marketing par segment pour mesurer le ROI réel
