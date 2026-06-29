# Arduiwatch — Fitness Tracker Embarqué avec Interface Mobile & Web

> Suivi d'activité physique basé sur Arduino Nano 33 BLE, avec application Flutter et interface web. Communication sans fil via Bluetooth Low Energy (BLE).

---

## Présentation

Arduiwatch est un système de fitness tracker complet développé autour d'un microcontrôleur **Arduino Nano 33 BLE**. Il détecte les pas en temps réel via l'accéléromètre intégré, reconnaît des commandes vocales via un modèle TinyML embarqué, et transmet les données à une application mobile (Flutter) ou une interface web (HTML/JS), qui calcule distance, calories brûlées et durée d'effort.

Le projet couvre toute la chaîne technique :
- Traitement du signal embarqué (C++ / Arduino)
- Inférence TinyML embarquée (Edge Impulse) - keyword spotting
- Communication sans fil BLE
- Application mobile cross-platform (Flutter/Dart)
- Interface web légère (HTML + Tailwind CSS)

---

## Fonctionnement technique

### 1. Détection des pas (firmware Arduino)

L'accéléromètre 3 axes est échantillonné en continu. La norme du vecteur d'accélération est calculée à chaque mesure :

```
|a| = sqrt(ax² + ay² + az²)
```

Le signal brut passe ensuite par une chaîne de traitement en quatre étapes.

**Étape 1 — Filtre passe-bas (EMA)**

Un filtre exponentiel lisse le signal et élimine les vibrations parasites haute fréquence :

```
smoothed = (ALPHA * rawAccMag) + ((1 - ALPHA) * smoothed)
```

Avec `ALPHA = 0.30`, chaque nouvelle mesure contribue à 30% de la valeur lissée, les 70% restants provenant de l'historique.

**Étape 2 — Fenêtre glissante et seuil dynamique**

Les 50 dernières valeurs lissées sont stockées dans un buffer circulaire. Le min et le max de cette fenêtre sont calculés en continu, et le seuil de détection est le point médian entre les deux :

```
dynamicThreshold = (maxAcc + minAcc) / 2.0
```

Cela rend la détection auto-adaptative : le seuil se recale automatiquement selon l'intensité réelle du mouvement, que l'on marche lentement ou en courant.

**Étape 3 — Détection de front montant**

Un pas est comptabilisé uniquement lorsque le signal lissé **passe au-dessus** du seuil dynamique, c'est-à-dire quand `smoothedAccMag > threshold` et `oldAccMag <= threshold` simultanément. Compter uniquement les fronts montants (et non descendants) garantit qu'un seul pas génère exactement une incrémentation.

La détection est conditionnée à deux gardes supplémentaires :
- `(maxAcc - minAcc) > NOISE_GATE` (0.3) : l'amplitude de la fenêtre doit être suffisante ; si la personne est immobile, le bruit résiduel est filtré
- `bufferFilled` : le buffer doit être plein avant toute détection pour éviter un seuil faussé au démarrage

**Étape 4 — Anti-rebond temporel**

Même si un front montant est détecté, le compteur n'est incrémenté que si 350ms se sont écoulées depuis le dernier pas validé. Cette valeur correspond à la cadence maximale physiologiquement atteignable (~170 pas/min) ; tout signal plus rapide est considéré comme du bruit.

```
if (currentMillis - lastStepTime > DEBOUNCE_DELAY):
    stepCount++
```

Le compteur est transmis via BLE toutes les secondes.

---

### 2. Reconnaissance vocale par TinyML

Le firmware intègre un modèle d'inférence embarqué généré avec **Edge Impulse**. Le microphone PDM de la carte échantillonne l'audio en continu. À chaque fenêtre d'analyse, le classifieur évalue la probabilité de chaque label :

| Label   | Seuil de confiance | Action déclenchée        |
|---------|--------------------|--------------------------|
| `marche`| > 80%              | Démarrage mode marche     |
| `course`| > 75%              | Démarrage mode course     |
| `stop`  | > 80%              | Retour en veille          |
| `noise` | —                  | Ignoré                   |

L'inférence et le podomètre s'exécutent en concurrence : pendant l'attente d'un nouveau buffer audio, `gererPodometre()` est appelé à chaque itération pour ne manquer aucun pas.

Par convention, le modèle en .zip n'a pas été push dans ce git.

---

### 3. Communication Bluetooth Low Energy

| Paramètre              | Valeur                                   |
|------------------------|------------------------------------------|
| Service UUID           | `19b100aa-e8f2-537e-4f6c-d104768a1214`  |
| Caractéristique Pas    | `19b10001-e8f2-537e-4f6c-d104768a1214`  |
| Caractéristique État   | `19b10002-e8f2-537e-4f6c-d104768a1214`  |
| Format (pas)           | Int32, Little-Endian                     |
| Format (état)          | Byte (`0` = veille, `1` = marche, `2` = course) |
| Intervalle d'envoi     | 1 seconde                                |

L'Arduino pilote l'application via la caractéristique d'état : c'est la commande vocale détectée embarquée qui déclenche le démarrage et l'arrêt de la séance côté application.

---

### 4. Calculs physiologiques (côté application)

**Distance :**

D'après l'article de R. Guest "Exploring the relationship between stride, stature and hand size for forensic assessment" publié dans Journal of Forensic and Legal Medicine.

```
longueur_foulée (m) = taille (cm) x 0.415 / 100
distance (km)       = nombre_de_pas x longueur_foulée / 1000
```

**Calories (méthode MET — Metabolic Equivalent of Task) :**
```
Kcal/min = (MET x 3.5 x poids_kg) / 200
calories = Kcal/min x durée_en_minutes
```

| Activité          | Valeur MET |
|-------------------|------------|
| Marche lente      | 2.5        |
| Marche dynamique  | 3.8        |
| Marche rapide     | 5.0        |
| Course / Jogging  | 8.0        |

---

## Application Flutter

Interface mobile pour Android et iOS, développée avec Flutter/Dart.

Fonctionnalités :
- Saisie du profil athlète (taille, poids) via sliders
- Connexion BLE automatique à la carte Arduino
- Dashboard temps réel : pas, distance, calories, chronomètre
- Démarrage et arrêt pilotés par les commandes vocales Arduino
- Écran de résumé post-activité avec calcul de l'allure (min/km)

Dépendances principales :
- `flutter_blue_plus` — gestion BLE
- `google_fonts` — typographie
- `permission_handler` — permissions Android/iOS

---

## Interface Web

Alternative légère à l'application mobile, compatible avec tout navigateur supportant la Web Bluetooth API (Chrome/Edge).

Fonctionnalités :
- Configuration du profil via formulaire
- Connexion BLE navigateur natif
- Dashboard identique : pas, distance, calories, chronomètre

Note : la Web Bluetooth API requiert Chrome ou Edge sur Android/Desktop. Non supportée sur iOS/Safari.

---

## Installation

### Firmware Arduino

Matériel requis : Arduino Nano 33 BLE

```
1. Ouvrir script_test.ino dans l'IDE Arduino
2. Installer les bibliothèques via le Gestionnaire de bibliothèques :
   - ArduinoBLE
   - Arduino_LSM9DS1
   - La bibliothèque Edge Impulse exportée depuis le projet (Projet_de_Recherche_inferencing)
3. Selectionner la carte : Arduino Nano 33 BLE
4. Téléverser
```

### Application Flutter

```bash
git clone https://github.com/<username>/Arduiwatch.git
cd Arduiwatch/mobile

flutter pub get
flutter run
```

Permissions Android requises dans `AndroidManifest.xml` :
```xml
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

### Interface Web

Ouvrir `web/arduino_web.html` directement dans Chrome ou Edge. Aucun serveur requis.

---

## Architecture système

```
┌──────────────────────────────────┐              BLE              ┌──────────────────────────┐
│        Arduino Nano 33 BLE       │                               │  App Flutter / Web UI    │
│                                  │   stepChar (Int32, 1s)        │                          │
│  Accéléromètre LSM9DS1           │ ────────────────────────────► │  Calcul distance (MET)   │
│  └─ Filtre EMA                   │                               │  Calcul calories         │
│  └─ Fenetre glissante 50pts      │   etatChar (byte)             │  Chronometre             │
│  └─ Seuil dynamique              │ ────────────────────────────► │  Dashboard temps reel    │
│  └─ Debounce 350ms               │                               │  Ecran resume            │
│                                  │                               └──────────────────────────┘
│  Microphone PDM                  │
│  └─ Modele TinyML (Edge Impulse) │
│  └─ Labels : marche/course/stop  │
└──────────────────────────────────┘
```

---

## Pistes d'amélioration

- [ ] Historique des séances persistant côté application (SQLite / Supabase)
- [ ] Export des données au format CSV ou GPX
- [ ] Réentraînement du modèle TinyML avec un dataset plus large pour améliorer la robustesse de la reconnaissance vocale en conditions bruyantes
- [ ] Ajout d'un label `noise` plus agressif pour réduire les faux positifs

---

## Stack technique

| Couche              | Technologie                        |
|---------------------|------------------------------------|
| Firmware embarqué   | C++ / Arduino IDE                  |
| Inférence embarquée | Edge Impulse (TinyML)              |
| Communication       | Bluetooth Low Energy (BLE)         |
| Application mobile  | Flutter / Dart                     |
| Interface web       | HTML5, JavaScript, Tailwind CSS    |
| Capteur             | Accéléromètre + Micro PDM LSM9DS1  |
| Matériel            | Arduino Nano 33 BLE                |



## Auteurs
Ruiz Alexandre
Fracso Yanis
Lecadre Barthélémy
Sendrey Baptiste

Projet réalisé dans le cadre d'un projet de recherche à **IMT Nord Europe**.