# Team_707-PLM-Hackathon-ESILV
# E-Motion | Value Network monitoring

Tableau de bord Streamlit construit à partir d'un export XPDMXML (3DEXPERIENCE) : structure produit,
dépendances par pays, réutilisation des composants, analyse coût et risque.

## Lancer (Windows)

Double-cliquer sur `demarrer.bat`, ou dans un terminal ouvert dans ce dossier :

```powershell
Set-Location 'C:\Users\cyril\ecar_plm_app_v2\ecar_plm_app'
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m streamlit run app.py
```

Python 3.10 ou plus récent. Sur macOS ou Linux, remplacer `py` par `python3`.

Si l'environnement `.venv` existe déjà avec ses dépendances, seule la dernière commande est nécessaire.
`demarrer.bat` se place automatiquement dans son dossier et utilise cet environnement ; il installe les
dépendances si elles sont absentes. L'application est accessible à `http://localhost:8501`.
Si le port 8501 est occupé par une autre copie, lancer `.\demarrer.bat 8502`, puis ouvrir
`http://localhost:8502`. Le script utilise toujours le fichier `app.py` de son propre dossier.
Sur macOS/Linux, l'exécutable de l'environnement est `.venv/bin/python`.

## Analyse pays et impact des composants

- **Analyse par pays** remplace l'entrée « Suivi fournisseurs ». La vue principale regroupe les références
  par pays du site fournisseur, avec leurs noms, quantités, part des achats connus et parents concernés.
  Un filtre affiche les composants d'un même pays, même s'ils viennent de fournisseurs différents.
  Les informations fournisseurs restent accessibles dans « Détail fournisseurs ».
- **Analyse coût & risque** commence par les dépendances pays et un tableau de réutilisation.
  Le classement propose les parents directs distincts, les instances père–fils, les instances dans le
  produit et les parents impactés à tous les niveaux. Les pièces et assemblages peuvent être filtrés.
- Les instances père–fils sont la somme des quantités locales ; les instances produit tiennent compte
  des multiplicités de tous les assemblages. Exemple : 3 pièces dans un père présent 2 fois et 2 pièces
  dans un autre père présent 4 fois donnent 2 parents directs, 5 instances père–fils et 14 pièces par produit.
  Les ancêtres communs sont comptés une seule fois. Le tableau se recalcule après édition ou import XML.
- Les pièces sans sourcing apparaissent sous « Pays non renseigné ». Les objets de modélisation exclus
  du chiffrage ne sont pas traités comme des achats. Les regroupements utilisent les données existantes :
  l'exemple vis / stator américains n'est affiché que si les données associent ces pièces aux États-Unis.
- Les pays et indices de risque sont simulés. Une exposition commune indique une dépendance géographique,
  sans constituer une probabilité de rupture. Le pays du site fournisseur n'est pas une origine matière certifiée.

## Fonctionnalités ajoutées

- **Analyse coût & risque → Liens père–fils et impact d'une évolution** : ajouter un lien entre un assemblage
  et une référence existante, ou fixer sa quantité locale. Les cycles sont refusés. Les quantités se multiplient
  sur chaque chemin puis s'additionnent quand une référence est partagée. Les coûts, opérations d'assemblage,
  capacités et prévisions tiennent compte de ces quantités.
- **Graphe d'impact** : choisir une pièce ou un assemblage, afficher tous ses parents et filtrer la profondeur.
  Les flèches enfant → père portent la quantité locale. Une variation de coût par instance calcule l'effet
  par produit et par an. Les impacts des différents niveaux ne s'additionnent pas entre eux.
- **Fluctuation des prix matières et risques pays** : simuler une hausse ou baisse matière, un choc sur les
  achats d'un pays et un retard. Ces hypothèses se cumulent au risque sélectionné et à la hausse globale.
  Le tableau en bas de page regroupe les composants par pays fournisseur, matériau et parents concernés,
  avec les quantités et variations de coût. Les prix sont simulés ; aucune donnée de marché n'est récupérée.
- **Import / Export XML**, disponible au-dessus des trois pages : charger un export XPDMXML 3DEXPERIENCE,
  télécharger la nomenclature modifiée, ou revenir au fichier d'origine. Limite : 20 Mo. Le fichier XML
  d'origine n'est pas écrasé. Les changements restent propres à la session : exporter avant de la fermer.

L'export conserve les métadonnées et représentations du XML source. Les instances ajoutées ont de nouveaux
identifiants ; leur position copie celle du lien existant, sinon elle est à l'origine. Le placement CAO
et les masses déclarées restent à vérifier dans le PLM. Les contrôles de masse signalent les écarts.
Le fichier exporté peut être réimporté dans cette application ; son intégration dans 3DEXPERIENCE n'a pas été validée.

Les données achats restent dans `data/business/*.csv`, associées par ID PLM : un autre XML peut n'avoir
qu'un chiffrage partiel. Les pièces sans données sont signalées et restent visibles dans la structure et
le graphe d'impact. Les simulations de prix ne sont pas enregistrées dans le XML.

## Vérifications

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
```

## Structure

```
app.py                     navigation (3 onglets)
assets/logo.svg
data/XPDMXML_ECAR_PLM.xml  export PLM (le premier .xml de data/ est utilisé)
data/business/             données simulées, rattachées par ID PLM
  business_parameters.csv  prix de vente, volume
  sourcing.csv             pièce → fournisseur, prix, alternative, capacité
  suppliers.csv            fournisseurs (performances, site, ESG, contrat)
  countries.csv            pays (risque, mix électrique, transport)
  materials.csv            matériaux (prix/kg, CO₂/kg, volatilité)
  risks.csv                registre des risques → scénario
  assembly_operations.csv  coûts d'assemblage interne
  substitutions.csv        matériaux de substitution
plm/parser.py              lecture XPDMXML
plm/model.py               nomenclature, quantités, contrôles qualité
plm/business.py            coûts, CO₂, criticité, scénarios
plm/market.py              séries de prix matières et prévisions IF
views/structure.py         onglet Structure produit
views/tree.py              arborescence parent → enfant (coûts cumulés, vue locale d'un composant)
views/suppliers.py         onglet Analyse par pays et détail fournisseurs
views/countries.py         regroupements et filtre par pays
plm/dependencies.py        exposition pays et réutilisation des composants
views/impact.py            classement, édition et graphe d'impact
views/risk.py              onglet Analyse coût & risque
```

## Règles de calcul

- Prix d'achat = part matière (masse PLM × prix/kg) + transformation.
- Cours matière : agit sur la part matière. Droit de douane : agit sur tout le prix d'achat.
- Prévision IF : le choc démarre au 2e mois de prévision (montée sur 2 mois).
- Criticité sur 100 : coût 25, risque pays 20, source unique 15, concentration pays 10, délai 15, matière 15.

- Hausse du prix des composants (champ « Simulation » des onglets Structure et Analyse) : % saisi, appliqué au prix
  d'achat des composants (après droits de douane), sur tout le produit ou un sous-ensemble. 0 % = aucun effet.
  Dans l'onglet Analyse, elle se cumule au risque sélectionné dans la prévision IF.

Pour ajouter un risque : une ligne dans `risks.csv` et le scénario correspondant dans `PRESETS` (`plm/business.py`).
