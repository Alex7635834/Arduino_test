PROJET NANO TRACKER PRO

DESCRIPTION DU PROJET
Ce depot contient l'ensemble du code source du projet Nano Tracker Pro, un systeme embarque complet de suivi d'activite physique. Le dispositif repose sur l'integration d'un microcontroleur Arduino dote d'une connectivite Bluetooth Low Energy, d'une centrale inertielle pour l'acquisition des donnees biomecaniques, et d'un microphone exploitant un modele d'apprentissage automatique pour la reconnaissance vocale des differents modes d'activite.

ARCHITECTURE DU SYSTEME
Le projet est architecture autour de trois composants logiciels distincts, assurant une separation stricte entre l'acquisition des signaux, leur transmission et l'interface utilisateur :

script_test : Le firmware en C++ destine a la carte Arduino. Il orchestre le traitement du signal inertiel en temps reel, l'inference du modele sur le flux audio, et la gestion du serveur GATT pour la communication Bluetooth.

code_app : Une application mobile developpee en Flutter et Dart. Elle gere la decouverte et la synchronisation avec l'Arduino, permet le parametrage des donnees physiologiques de l'athlete, et calcule les metriques d'effort detaillees (distance, allure, calories).

arduino_web.html : Une interface Web HTML et JavaScript offrant une alternative au client mobile en exploitant l'API Web Bluetooth pour lire et afficher les donnees de la seance directement dans le navigateur.

TRAITEMENT DU SIGNAL ET ALGORITHMIQUE

PARTIE 1 : PODOMETRE ET ACCELEROMETRIE
Le comptage des pas est realise localement par le microcontroleur afin de minimiser la charge de la bande passante Bluetooth. Il repose sur l'analyse rigoureuse de la norme du vecteur d'acceleration tridimensionnel :

Filtrage : Application d'un filtre passe-bas avec un coefficient alpha de 0.30 pour attenuer le bruit de mesure haute frequence inherent au capteur physique.

Seuillage Dynamique : Le seuil de detection des pas n'est pas fixe. Il est recalcule dynamiquement sur une fenetre glissante de 50 echantillons en evaluant la moyenne des accelerations maximales et minimales recentes.

Delai Anti-rebond : Un filtre temporel strict de 350 millisecondes ignore les pics d'acceleration trop rapproches, prevenant de maniere proactive les faux positifs.

PARTIE 2 : CLASSIFICATION VOCALE
Le changement de contexte d'effort s'effectue exclusivement par commande vocale. Le systeme embarque un reseau de neurones genere via Edge Impulse, qui analyse les donnees brutes du microphone PDM. L'inference classifie le flux audio selon trois classes avec des seuils de probabilite severes :

La detection du mot Marche necessite un degre de certitude superieur a 80 pourcents.

La detection du mot Course necessite un degre de certitude superieur a 75 pourcents.

La detection du mot Stop necessite un degre de certitude superieur a 80 pourcents.

PARTIE 3 : MODELES BIOMECANIQUES ET DEPENSE ENERGETIQUE
L'Arduino transmet asynchrone le compteur brut de pas et les changements d'etat via le protocole Bluetooth. Les clients (Flutter ou Web) ont la responsabilite d'effectuer les calculs physiologiques terminaux en s'appuyant sur les donnees anthropometriques fournies par l'utilisateur :

Longueur de foulee : Elle est modelisee par l'equation empirique multipliant la taille en centimetres par un facteur de 0.415.

Distance absolue : Elle est obtenue en multipliant le nombre total de pas par la longueur de la foulee calculee precedemment.

Depense Energetique : L'application utilise l'Equivalent Metabolique, une variable dependante du type d'effort (par exemple 3.8 pour la marche dynamique, 8.0 pour la course). La puissance calorique est evaluee par la formule clinique suivante : Kcal par minute equivaut au produit de l'Equivalent Metabolique, de la constante 3.5 et du poids en kilogrammes, le tout divise par 200. Cette puissance est ensuite integree sur la duree d'effort pour obtenir l'energie totale consommee.

DEPLOIEMENT ET INSTRUCTIONS D'UTILISATION

Etape 1 : Flashez le contenu du fichier script_test sur votre carte Arduino Nano. Vous devez vous assurer au prealable d'avoir installe les bibliotheques requises dans votre environnement (PDM, ArduinoBLE, Arduino_LSM9DS1) ainsi que l'archive specifique de votre modele d'apprentissage Edge Impulse.
Etape 2 : Compilez le dossier code_app via votre environnement de developpement Flutter. Autorisez imperativement l'acces au Bluetooth et aux services de localisation sur l'appareil cible.
Etape 3 : Mettez la carte sous tension, lancez l'application mobile ou web, et connectez l'appareil. Prononcez distinctement le mot Marche ou Course pres du microphone de l'Arduino pour declencher l'acquisition et le calcul des metriques.
