# Méthodologie

## Principe général

Le modèle repose sur un moteur de cash-flow unique, piloté par une table technique d'années (`TECH_Annees_CF`) et une table de lignes de cash-flow (`LignesCF`) qui définit l'ordre d'affichage. Une seule mesure centrale calcule, pour chaque combinaison (année × ligne), le montant correspondant — plutôt que d'avoir une mesure par ligne du compte de résultat.

Cette approche (déjà utilisée dans le moteur de valorisation parking dont ce projet reprend l'architecture) permet d'ajouter ou modifier une ligne de cash-flow sans dupliquer la logique de filtrage par année.

## Étapes du calcul

1. **Loyer brut indexé** : `LoyerBase × (1 + ILAT)^(année − année de départ)`
2. **Vacance** : lecture d'un taux de vacance par année dans une table dédiée (`Hyp_Vacance_Annee`), via `LOOKUPVALUE` — indépendante du contexte de filtre du visuel, donc fiable même dans des mesures imbriquées.
3. **Loyer net de vacance** : loyer brut × (1 − taux de vacance de l'année)
4. **Charges** : séparées en récupérables (mémo, neutres sur le NOI) et non récupérables (déduites)
5. **NOI** : loyer net de vacance − charges non récupérables
6. **Résultat imposable / IS** : NOI − amortissement, IS appliqué uniquement si le résultat est positif
7. **Flux net d'exploitation** : NOI − IS
8. **Flux d'investissement** : CAPEX et frais immobilisables l'année de départ (négatifs), et à la sortie, valeur de cession nette des frais de sortie
9. **Valeur de sortie hors droits** : méthode du rendement — `NOI de la dernière année / taux de sortie`, puis frais de sortie déduits
10. **VAN** : somme des flux nets actualisés au taux d'actualisation choisi, `t = 1` pour l'année de départ
11. **TRI** : calculé par `XIRR` sur les flux datés (31/12 de chaque année), donc sur une base de jours réels — à ne pas confondre avec l'actualisation de la VAN, qui utilise des périodes entières. Les deux méthodes sont standards en immobilier mais ne convergent pas au centime près (écart de l'ordre de 0,1 % sur ce modèle, sans conséquence pratique).
12. **Valeur vénale** : `VAN / (1 + taux de frais d'acquisition)`

## VAN cumulée et année de retournement

Une mesure de VAN cumulée (running NPV) recalcule, pour chaque année de l'horizon, la somme des flux actualisés depuis le départ jusqu'à cette année — sans se limiter à l'horizon complet. Elle convient à un visuel en aire pour visualiser la trajectoire de création de valeur, et sert de base à une mesure dérivée qui identifie la première année où cette VAN cumulée devient positive (le "point de retournement").

## Paramétrage interactif

Chaque hypothèse clé (ILAT, taux d'actualisation, taux de sortie, frais de sortie, frais d'acquisition, CAPEX, durée, année de départ) est exposée via une table calculée disjointe (`GENERATESERIES`), utilisée comme slicer. Une mesure `Valeur Param_X` lit la sélection active (`SELECTEDVALUE`) avec un repli par défaut ; une mesure `X_modele` combine ce paramètre avec la donnée de la table d'hypothèses du projet, avec priorité au paramètre.

Ce découpage en deux couches (paramètre interactif / hypothèse de base) permet de piloter le modèle en temps réel via les sliders tout en gardant une valeur de référence documentée pour le cas de base.
