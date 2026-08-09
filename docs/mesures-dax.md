# Extraits de mesures DAX clés

Sélection de mesures illustrant la technique, pas l'intégralité du modèle.

## Vacance par année (lookup indépendant du contexte de filtre)

```dax
TauxVacance_modele =
VAR YearSel = SELECTEDVALUE('TECH_Annees_CF'[Année])
RETURN
COALESCE(
    LOOKUPVALUE(Hyp_Vacance_Annee[TauxVacance], Hyp_Vacance_Annee[Année], YearSel),
    0
)
```

`LOOKUPVALUE` ignore le contexte de filtre ambiant — utile ici puisque l'année recherchée vient d'une sélection dans une autre mesure, pas d'un filtre de visuel classique.

## VAN

```dax
VAN_modele =
VAR r      = COALESCE([Valeur Param_TauxActual], SELECTEDVALUE(Hyp_valo_bureau[Taux d'actualisation]))
VAR startY = SELECTEDVALUE(Hyp_valo_bureau[AnnéeDeDépart])
RETURN
SUMX(
    FILTER(ALL('TECH_Annees_CF'), 'TECH_Annees_CF'[Année] >= startY),
    DIVIDE(
        [Flux Net],
        POWER(1 + r, 'TECH_Annees_CF'[Année] - startY + 1)
    )
)
```

## TRI

```dax
TRI_modele =
VAR startY = SELECTEDVALUE(Hyp_valo_bureau[AnnéeDeDépart])
VAR CFTable =
    FILTER(
        ADDCOLUMNS(
            VALUES('TECH_Annees_CF'[Année]),
            "Amt", [Flux Net],
            "Dt",  DATE('TECH_Annees_CF'[Année], 12, 31)
        ),
        NOT ISBLANK([Amt]) && 'TECH_Annees_CF'[Année] >= startY - 1
    )
VAR HasData = COUNTROWS(CFTable) > 1
VAR HasPos  = COUNTROWS(FILTER(CFTable, [Amt] > 0)) > 0
VAR HasNeg  = COUNTROWS(FILTER(CFTable, [Amt] < 0)) > 0
RETURN
IF(NOT(HasData && HasPos && HasNeg), BLANK(), XIRR(CFTable, [Amt], [Dt], 0))
```

## Valeur vénale

```dax
ValeurVenale_modele = DIVIDE([VAN_modele], 1 + [FraisAcquisition_modele])
```

## VAN cumulée (trajectoire de valeur)

```dax
VAN_Cumulee_modele =
VAR r      = COALESCE([Valeur Param_TauxActual], SELECTEDVALUE(Hyp_valo_bureau[Taux d'actualisation]))
VAR startY = SELECTEDVALUE(Hyp_valo_bureau[AnnéeDeDépart])
VAR YearSel = SELECTEDVALUE(TECH_Annees_CF[Année])
RETURN
IF(
    ISBLANK(YearSel) || ISBLANK(startY) || YearSel < startY,
    BLANK(),
    SUMX(
        FILTER(ALL(TECH_Annees_CF), TECH_Annees_CF[Année] >= startY && TECH_Annees_CF[Année] <= YearSel),
        DIVIDE([Flux Net], POWER(1 + r, TECH_Annees_CF[Année] - startY + 1))
    )
)
```

## Année de retournement

```dax
AnneePayback_modele =
VAR startY = SELECTEDVALUE(Hyp_valo_bureau[AnnéeDeDépart])
VAR TableYears =
    FILTER(
        ADDCOLUMNS(
            FILTER(ALL(TECH_Annees_CF), TECH_Annees_CF[Année] >= startY),
            "VANCum", [VAN_Cumulee_modele]
        ),
        [VANCum] >= 0
    )
RETURN
MINX(TableYears, TECH_Annees_CF[Année])
```

## Couleur conditionnelle dynamique (et non figée)

```dax
Couleur_TRI_modele =
VAR tri = [TRI_modele]
VAR r = COALESCE([Valeur Param_TauxActual], SELECTEDVALUE(Hyp_valo_bureau[Taux d'actualisation]))
RETURN
IF(ISBLANK(tri) || ISBLANK(r), "#605E5C", IF(tri >= r, "#107C10", "#D13438"))
```

Le seuil de comparaison (`r`, le taux d'actualisation sélectionné) est lui-même une mesure, pas une valeur codée en dur — la couleur reste correcte quel que soit le scénario choisi sur les sliders, contrairement à une règle de mise en forme conditionnelle native basée sur un seuil fixe.
