# Détection d'Intrusions Réseau — CICIDS2017

Ce projet implémente un pipeline de Machine Learning pour la détection d'attaques réseau (DDoS, DoS, Brute Force, Port Scanning, Web Attacks, Bots) à partir du dataset **CICIDS2017** (version nettoyée/prétraitée). Trois modèles de classification sont entraînés et comparés : **Random Forest**, **XGBoost** et **KNN**.

## Objectif

Classifier chaque flux réseau (« flow ») en trafic normal ou en l'un des types d'attaque présents dans le dataset, à partir d'un sous-ensemble des 20 caractéristiques les plus importantes du trafic (ports, tailles de paquets, durées de flux, flags TCP, etc.).

## Dataset

- **Source** : [CICIDS2017 cleaned and preprocessed](https://www.kaggle.com/datasets/ericanacletoribeiro/cicids2017-cleaned-and-preprocessed) (Kaggle)
- **Colonne cible** : `Attack Type`
- **Classes** : Normal Traffic, DDoS, DoS, Brute Force, Port Scanning, Web Attacks, Bots

### Caractéristiques retenues

Le dataset original contient de nombreuses colonnes ; seules les 19 plus pertinentes (+ le label) sont conservées, par exemple :

`Destination Port`, `Flow Duration`, `Flow Bytes/s`, `Flow Packets/s`, `Init_Win_bytes_forward/backward`, `Fwd/Bwd Packet Length`, `Flow IAT Max/Mean`, `PSH Flag Count`, `ACK Flag Count`, etc.

## Pipeline

1. **Téléchargement & extraction** du dataset via l'API Kaggle.
2. **Chargement** du CSV et sauvegarde d'une copie sur Google Drive.
3. **Sélection de colonnes** : réduction à 20 features pertinentes pour alléger le dataset.
4. **Split train/test** (70/30, stratifié sur `Attack Type`).
5. **Normalisation** avec `RobustScaler` (robuste aux outliers, adapté au trafic réseau).
6. **Rééquilibrage des classes** :
   - `RandomUnderSampler` pour réduire la classe majoritaire (`Normal Traffic` → 500 000 échantillons).
   - `SMOTE` pour sur-échantillonner les classes minoritaires (Bots, Web Attacks, Brute Force, Port Scanning, DDoS, DoS).
7. **Entraînement de 3 modèles** :
   - **Random Forest** : entraîné sur les données non scalées (sous-échantillonnées).
   - **XGBoost** : entraîné sur les données scalées + SMOTE, avec encodage des labels (`LabelEncoder`), requis pour l'objectif `multi:softmax`.
   - **KNN** : entraîné sur les données scalées + SMOTE (la distance euclidienne exige des features normalisées).
8. **Évaluation** sur le jeu de test via `classification_report` (précision, rappel, F1-score par classe).
9. **Sauvegarde** des modèles (`rf_model.joblib`, `xgb_model.joblib`, `xgb_label_encoder.joblib`, `knn_model.joblib`) et du scaler (`robust_scaler.joblib`) pour un déploiement ultérieur.

## Point important : quel jeu de test pour quel modèle ?

| Modèle | Données d'entraînement | Données de test à utiliser |
|---|---|---|
| Random Forest | Non scalées, sous-échantillonnées | `X_test` |
| XGBoost | Scalées, sous+sur-échantillonnées (SMOTE) | `X_test_scaled` |
| KNN | Scalées, sous+sur-échantillonnées (SMOTE) | `X_test_scaled` |

Le Random Forest étant insensible à l'échelle des variables (contrairement à KNN), il est entraîné et évalué sur les données brutes ; XGBoost et KNN utilisent les versions normalisées.


## Dépendances

```
pandas
numpy
scikit-learn
xgboost
imbalanced-learn
matplotlib
seaborn
joblib
```

## Structure des fichiers

```
├── CICDDOS_2017_corrige.ipynb   # Notebook complet (pipeline + entraînement des 3 modèles)
├── robust_scaler.joblib          # Scaler entraîné (généré à l'exécution)
├── rf_model.joblib                # Modèle Random Forest entraîné
├── xgb_model.joblib               # Modèle XGBoost entraîné
├── xgb_label_encoder.joblib       # Encodeur des labels pour XGBoost
└── knn_model.joblib               # Modèle KNN entraîné
```

## Utilisation

Le notebook est conçu pour être exécuté sur **Google Colab** (montage de Google Drive, téléchargement via l'API Kaggle). Placer un fichier `kaggle.json` valide à la racine avant exécution.
