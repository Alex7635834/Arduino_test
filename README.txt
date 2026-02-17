Nombres de pas : calcul de l’accélération
bibliothèque ou juste un seuil de la somme des accélérations

distance : on utilise formule distance = nombre de pas * longueur foulé 
ou longueur foulé soit on met 0,7 m tout le temp ou on adapte selon la taille du mec

calories : Kcal/minute = (MET X 3,5 X Poids en kg)/200 
ou MET= metabolic equivalent of task c’est un coefficient qui change selon l’activité

temps : chronomètre

En pratique l’arduino détecte l'accélération et en déduit le nombre de pas, 
il envoie ça à l’application, l’application récupère ça et fait les calculs du dessus 
a partir du nombre de pas du poid et de la taille.

Changé d'activité reviens donc à changer la valeur du MET.

