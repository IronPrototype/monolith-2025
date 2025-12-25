# Plan de mise en place du tableau de bord SAV / Atelier / Facturation

## 1. Sources de données
- **Tables attendues** :
  - `SAV` : dossier SAV, dates (ouverture, clôture), statut SAV, client, produit, indicateurs de retour.
  - `EntréesAtelier` : dates d'entrée, identifiants dossiers, statut d'avancement, marquage facturé/non facturé, client, produit.
  - `Facturation` : numéro de facture, date de facture, montant, statut (payée/en attente), lien dossier/commande, client, produit.
- **Champs clés à vérifier** : dates, numéro de dossier/commande/facture, statut SAV, montant facturé, statut facturation, client, produit.
- **Actualisation et accès** : privilégier une actualisation **quotidienne** (minimum hebdomadaire) via passerelle si données on-prem. Documenter les accès (service accounts, droits lecture).

## 2. Modélisation des données (Power BI Desktop)
- Charger les 3 tables dans Power BI Desktop.
- Modèle en étoile :
  - **Faits** : `F_SAV_KPI`, `F_Entrées_Atelier`, `F_Factures`.
  - **Dimensions** : `D_Date` (incluant semaine ISO), `D_Client`, `D_Produit`, `D_Statut` (SAV/facturation si partagée) ; éventuellement `D_Atelier` ou `D_Pays` si besoin de filtres supplémentaires.
- Relations :
  - `D_Date[Date]` en liaison 1-* vers chaque table de faits via la date d'événement (entrée, clôture, facture).
  - `D_Client` et `D_Produit` reliées 1-* aux faits.
- **Granularité** : niveau « dossier » pour SAV, « entrée atelier » pour atelier, « facture » pour facturation. S'assurer que chaque table de faits a des clés sur ce niveau et qu'aucune relation plusieurs-à-plusieurs n'est nécessaire.

## 3. Mesures DAX principales
- **SAV**
  - `Nb_Dossiers_SAV = DISTINCTCOUNT(F_SAV_KPI[IdDossier])`
  - `Taux_Clôture = DIVIDE(CALCULATE([Nb_Dossiers_SAV], F_SAV_KPI[Statut] = "Clôturé"), [Nb_Dossiers_SAV])`
  - `Temps_Moyen_Traitement = AVERAGEX(F_SAV_KPI, DATEDIFF(F_SAV_KPI[DateOuverture], F_SAV_KPI[DateCloture], DAY))`
  - `Taux_Retours = DIVIDE(CALCULATE([Nb_Dossiers_SAV], F_SAV_KPI[FlagRetour] = TRUE()), [Nb_Dossiers_SAV])`
- **Entrées atelier**
  - `Nb_Entrées_Semaine = CALCULATE(COUNTROWS(F_Entrées_Atelier), DATESWTD(D_Date[Date]))`
  - `Nb_Entrées_Facturées = CALCULATE(COUNTROWS(F_Entrées_Atelier), F_Entrées_Atelier[EstFacturé] = TRUE())`
  - `Nb_Entrées_Non_Facturées = CALCULATE(COUNTROWS(F_Entrées_Atelier), F_Entrées_Atelier[EstFacturé] = FALSE())`
- **Facturation**
  - `CA_Facturé = SUM(F_Factures[Montant])`
  - `CA_En_Attente = CALCULATE([CA_Facturé], F_Factures[Statut] = "En attente")`
  - `Montant_Moyen_Facture = AVERAGE(F_Factures[Montant])`
  - `Délai_Moyen_Paiement = AVERAGEX(F_Factures, DATEDIFF(F_Factures[DateFacture], F_Factures[DatePaiement], DAY))`
- **Périodes** : décliner les mesures YTD/MTD/WTD via `CALCULATE` + `DATESYTD`, `DATESMTD`, `DATESWTD` ou `WEEKNUM` dans `D_Date`.

## 4. Visuels et filtres
- **Cartes KPI** : Nb dossiers SAV, Taux de clôture, CA facturé, Nb entrées atelier.
- **Courbes/colonnes** : évolution hebdo/mensuelle/annuelle des entrées et du CA.
- **Table détaillée factures (semaine courante)** : numéro facture, client, montant, statut payé/non payé, date (filtre relatif sur semaine en cours).
- **Slicers** : hiérarchie `D_Date` (année → mois → semaine) + filtres Client/Produit/Statut.
- **Info-bulles** : détails SAV et factures sur survol.

## 5. Mise en forme et publication
- Palettes et branding harmonisés ; tester les interactions et filtres croisés.
- Publier sur le service Power BI, configurer l'actualisation (gateway si source locale) et les alertes si nécessaire.
- Partager avec le DG et parties prenantes identifiées.

## 6. Contrôles qualité
- Vérifier la cohérence des totaux (facturé vs non facturé) et des doublons de clés.
- Valider les plages temporelles (semaine ISO, mois fiscal le cas échéant).
- Ajouter la RLS si des restrictions d'accès sont requises.
