#Projet Nano Tracker Pro

## Description du Projet
Ce dépôt contient l'ensemble du code source du projet Nano Tracker Pro, un système embarqué complet de suivi d'activité physique. Le dispositif repose sur l'intégration d'un microcontrôleur Arduino doté d'une connectivité Bluetooth Low Energy (BLE), d'une centrale inertielle (IMU) pour l'acquisition des données biomécaniques (comptage des pas), et d'un microphone exploitant un modèle TinyML pour la reconnaissance vocale des différents modes d'activité.

## Architecture du Dépôt
Le projet est architecturé autour de trois composants logiciels distincts, assurant la séparation entre l'acquisition des données, leur transmission, et l'interface utilisateur :

* **`script_test`** : Le firmware en C++ destiné à la carte Arduino. Il orchestre le traitement du signal inertiel en temps réel, l'inférence du modèle d'apprentissage automatique sur le flux audio, et la gestion du serveur GATT pour la communication BLE.
* **`code_app`** : Une application mobile multiplateforme développée en Flutter/Dart. Elle gère la découverte et la synchronisation BLE avec l'Arduino, permet le paramétrage des données physiologiques de l'athlète, et calcule les métriques d'effort détaillées (distance, allure, calories).
* **`arduino_web.html`** : Une interface Web allégée (HTML/JS/TailwindCSS) offrant une alternative au client mobile en exploitant l'API Web Bluetooth pour lire et afficher les données de la séance.

## Traitement du Signal et Algorithmique

### 1. Podomètre et Accélérométrie (Embarqué)
Le comptage des pas est réalisé localement par le microcontrôleur afin de minimiser la bande passante Bluetooth. Il repose sur l'analyse de la norme du vecteur d'accélération tridimensionnel :
* **Filtrage** : Application d'un filtre passe-bas (coefficient $\alpha = 0.30$) pour atténuer le bruit de mesure haute fréquence inhérent au capteur LSM9DS1.
* **Seuillage Dynamique** : Le seuil de détection des pas est recalculé dynamiquement sur une fenêtre glissante (50 échantillons) en évaluant la moyenne des accélérations maximales et minimales récentes.
* **Délai Anti-rebond** : Un filtre temporel (350 ms) ignore les pics d'accélération trop rapprochés, prévenant ainsi les faux positifs.

### 2. Classification Vocale via TinyML (Embarqué)
Le changement de contexte d'effort s'effectue par commande vocale. Le système embarque un modèle d'apprentissage automatique généré via Edge Impulse, qui analyse les données du microphone PDM. L'inférence classifie le flux audio selon trois étiquettes avec des seuils de confiance stricts :
* **"Marche"** (Seuil de détection : > 80%)
* **"Course"** (Seuil de détection : > 75%)
* **"Stop"** (Seuil de détection : > 80%)

### 3. Modèles Biomécaniques et Dépense Énergétique (Client)
L'Arduino transmet de manière asynchrone le compteur brut de pas et les changements d'état via le protocole BLE. Les clients (Flutter ou Web) se chargent des calculs physiologiques en s'appuyant sur les données anthropométriques saisies par l'utilisateur (poids, taille) :
* **Longueur de foulée** : Modélisée par l'équation empirique `Taille (cm) × 0.415`.
* **Distance** : `Nombre de pas × Longueur de foulée`.
* **Dépense Énergétique (Calories)** : L'application utilise l'Équivalent Métabolique (MET), variant selon le type d'effort (ex: 3.8 pour la marche dynamique, 8.0 pour la course). La puissance calorique est évaluée par la formule clinique : 
  `Kcal/min = (MET × 3.5 × Poids en kg) / 200`
  Cette puissance est ensuite intégrée sur la durée chronométrée de l'effort pour obtenir l'énergie totale dépensée.

## Déploiement et Utilisation

1. **Firmware Arduino** : Flashez le contenu du fichier `script_test` sur votre carte (ex: Arduino Nano 33 BLE Sense). Assurez-vous d'avoir installé les bibliothèques requises (`PDM`, `ArduinoBLE`, `Arduino_LSM9DS1`) ainsi que l'archive de votre modèle Edge Impulse.
2. **Client Mobile** : Compilez le dossier `code_app` via l'environnement Flutter (`flutter run`). Autorisez l'accès au Bluetooth et à la localisation (requis par Android/iOS pour le scan BLE).
3. **Mise en route** : Allumez la carte, lancez l'application, et connectez l'appareil. Prononcez distinctement "Marche" ou "Course" près du microphone de l'Arduino pour initier le calcul des métriques.
