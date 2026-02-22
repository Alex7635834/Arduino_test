Le repo est composé de 
"arduino_web.html" : le fichier HTML du site qui affiche l'analyse de l'activité
"script_test" : le code en C++ a faire bouffer à l'arduino


En pratique l’arduino somme l'accélération sur les 3 axes, si la somme dépasse un certain seuil,
onn déduit que 1 pas a été fait. 
On peut aussi utiliser une ibliothèque tinyML, mais faire le seuil ca c'est moins lourd, et l'IA serait utile de fou pour dire "la il marche", "la il court",
si on connait déjà l'activité cest s'embeter pour rien.
En pratique on pourrais aussi faire (et a mon avis c'est la meilleur option) un filtre passe bas pour eliminer le bruit, faire un seuil dynamique et ignorer les pas trop proches les un des autres dans le temps (on fait tout ca a l'aide de bibliotheque arduino).

L'arduino envoir les pas toutes les 2 secondes envoie ça à l’application, l’application récupère ça et fait les calculs suivants 
a partir du nombre de pas, du poid et de la taille.

Nombres de pas : calcul de l’accélération
bibliothèque ou juste un seuil de la somme des accélérations

distance : on utilise formule distance = nombre de pas * longueur foulé 
ou longueur foulé = taille * 0,415

calories : On utilise la formule : Kcal/minute = (MET X 3,5 X Poids en kg)/200 
On multplie juste par le nombre de minute.
ou MET= metabolic equivalent of task c’est un coefficient qui change selon l’activité

temps : chronomètre

Changé d'activité reviens donc à changer la valeur du MET.

