# Vérificateur de conformité INCI (UE 1223/2009)

## Ce que fait l'outil

L'utilisateur saisit une formule (liste de noms INCI + % massique) et une
catégorie de produit. L'outil restitue, pour chaque ingrédient, les points de
contrôle réglementaires applicables :

- **interdit** — la substance figure à l'annexe II ;
- **restreint** — la substance figure à l'annexe III ; l'outil affiche chaque
  condition applicable et compare le % saisi à la limite ;
- **hors limite** — le % saisi dépasse une limite applicable à la catégorie
  et à la zone d'application déclarées ;
- **non autorisé pour cette fonction** — la fonction déclarée relève d'une
  liste positive (IV, V, VI) et la substance n'y figure pas ;
- **obligation d'étiquetage** — l'entrée impose une mention d'avertissement
  ou des conditions d'emploi ;
- **non déterminé** — la donnée n'est pas présente dans `/data`.

Utilisateur cible : formulateur cosmétique indépendant.
Auteur du projet : chimiste organicien, pas développeur. Le code doit rester
lisible et modifiable par lui : noms explicites, fonctions courtes,
commentaires en français sur la logique réglementaire.

## Stack imposée

- `index.html` : HTML, CSS et JS **inline**. Aucun framework, aucune
  dépendance, aucune étape de build. Le fichier s'ouvre directement dans un
  navigateur.
- Les données réglementaires vivent dans `/data/*.json`. **Aucune valeur
  réglementaire en dur dans le JS** : pas de nom de substance, pas de seuil,
  pas de mention d'avertissement, pas de numéro d'entrée. Le JS ne contient
  que de la mécanique (chargement, appariement, comparaison, affichage).
- Pas de backend au premier jet. Tout tourne côté client.
- Si une règle réglementaire ne peut pas s'exprimer dans le modèle de
  données, on étend le modèle de données — on ne la code pas en dur.

## Hiérarchie des sources — non négociable

1. Seuls le règlement (CE) n° 1223/2009 et ses annexes ont valeur juridique.
   C'est la seule source dont l'outil peut tirer un point de contrôle.
2. CosIng est une base d'information **non contraignante**. Elle sert à
   résoudre les noms INCI, les numéros CAS et EC. Elle ne sert jamais à
   statuer. Ne jamais présenter une information CosIng comme une règle.
3. Toute entrée de données porte obligatoirement :
   - `annexe_entree` au format `III/102` (numéro d'annexe / numéro d'entrée) ;
   - `reglement_modificatif` : la référence du règlement qui a introduit ou
     modifié l'entrée ;
   - `date_reglement` : sa date, au format ISO.
   **Sans ces trois champs, l'entrée est invalide et ne doit pas être
   chargée.** Le loader la rejette et la signale dans la console ; il ne la
   charge pas « en mode dégradé ».
4. CosIng contient l'historique depuis 1976. Ne charger que les entrées
   marquées actives (`actif: true`). Une entrée abrogée ou historique n'entre
   pas dans le moteur.

## Structure des annexes

| Annexe | Contenu | Nature |
|---|---|---|
| II | Substances interdites | liste négative |
| III | Substances soumises à restrictions | liste de conditions |
| IV | Colorants | **liste positive** |
| V | Conservateurs | **liste positive** |
| VI | Filtres UV | **liste positive** |

IV, V et VI sont des listes positives : une substance employée dans l'une de
ces fonctions et qui n'y figure pas est **non autorisée pour cette fonction**.

**Ne jamais conclure « autorisé » d'une absence.** L'absence d'une substance
de l'annexe II ne signifie pas qu'elle est autorisée. L'absence d'une
condition à l'annexe III ne signifie pas qu'il n'y a pas de limite. Toute
absence se restitue comme absence de donnée, pas comme feu vert.

## Pièges de lecture des restrictions — à modéliser explicitement

Ces points ne sont pas des détails d'implémentation : ils sont la raison
d'être de l'outil. Un modèle qui les écrase produit des résultats faux.

- **N conditions par substance, jamais une valeur unique.** Une limite
  d'annexe III dépend de la catégorie de produit et de la zone d'application
  (rincé / non rincé, capillaire, buccal, contour des yeux, muqueuses). Une
  même substance porte plusieurs limites dans une seule entrée. Le modèle est
  donc `substance → [conditions]`, chaque condition portant son propre champ
  d'application. Une substance dont plusieurs conditions s'appliquent au cas
  saisi les affiche toutes.
- **Base d'expression obligatoire.** Les concentrations sont exprimées sur des
  bases différentes : acide libre, base, sel, teneur en substance active, ou
  dans la « préparation prête à l'emploi ». `base_expression` est un champ
  obligatoire de chaque condition et **doit être affiché dans le résultat**.
  Ne jamais comparer un % saisi à une limite sans afficher la base : la
  comparaison n'a pas de sens sans elle. Si la base du % saisi et celle de la
  limite peuvent différer, le dire au lieu de convertir.
- **Limites de groupe.** Certaines limites portent sur la **somme** d'un
  groupe de substances, pas sur chacune. Modéliser les groupes
  (`/data/groupes.json`), calculer la somme des membres présents dans la
  formule, et indiquer quels membres ont été additionnés.
- **Mentions au mot.** Les mentions d'avertissement obligatoires et les
  conditions d'emploi sont restituées **verbatim**, sans reformulation, sans
  traduction, sans troncature, sans résumé. Champ dédié, affichage intégral.
- **Formes nano et critères de pureté : entrées distinctes.** Une forme nano
  ne partage pas l'entrée de la forme non nano. Idem pour les entrées portant
  des critères de pureté ou de spécification. Ne jamais fusionner.

## Modèle de données

`/data/` contient au minimum :

- `annexe-II.json`, `annexe-III.json`, `annexe-IV.json`, `annexe-V.json`,
  `annexe-VI.json` — les entrées réglementaires ;
- `groupes.json` — les groupes soumis à une limite de somme ;
- `categories.json` — les catégories de produit et zones d'application
  proposées à la saisie ;
- `meta.json` — `date_mise_a_jour` des données et périmètre couvert.

Forme d'une entrée (annexe III, la plus riche) :

```json
{
  "annexe_entree": "III/102",
  "inci": "NOM INCI",
  "synonymes": ["..."],
  "cas": "...",
  "ec": "...",
  "actif": true,
  "nano": false,
  "reglement_modificatif": "Règlement (UE) .../...",
  "date_reglement": "AAAA-MM-JJ",
  "groupes": ["id_de_groupe"],
  "conditions": [
    {
      "champ_application": {
        "categories": ["..."],
        "zones": ["non_rince"],
        "exclusions": ["..."]
      },
      "limite_pct": null,
      "base_expression": "acide libre",
      "mention_avertissement": "texte verbatim ou null",
      "conditions_emploi": "texte verbatim ou null",
      "purete": "texte verbatim ou null",
      "note": "..."
    }
  ]
}
```

`limite_pct: null` signifie **non déterminé**, jamais « pas de limite ».

Le loader valide chaque entrée à l'ouverture : présence de `annexe_entree`,
`reglement_modificatif`, `date_reglement`, `actif`, et pour chaque condition
de `champ_application` et `base_expression`. Toute entrée non conforme est
rejetée, comptée, et le nombre d'entrées rejetées est visible dans l'interface.

## Ne pas faire

- **Ne pas inventer, arrondir ni interpoler une valeur limite.** Donnée
  absente = « non déterminé, à vérifier dans l'annexe », avec le numéro
  d'entrée et le lien vers l'entrée. Aucune valeur ne doit apparaître dans le
  résultat sans exister dans `/data`.
- **Ne pas déduire une conformité globale.** L'outil signale des points de
  contrôle ; il ne délivre pas de verdict. Pas de « formule conforme », pas de
  score, pas de pastille verte globale, pas de « 0 problème détecté ».
- **Ne jamais présenter le résultat comme un substitut** au rapport de
  sécurité et au dossier d'information produit exigés par le règlement, qui
  relèvent d'un évaluateur qualifié. Ce rappel est **visible en permanence
  dans l'interface**, à hauteur des résultats — pas dans un pied de page, pas
  dans un coin, pas derrière un dépliant.
- **Ne pas traiter la couverture comme complète.** Afficher en permanence la
  `date_mise_a_jour` des données et le fait que les annexes sont amendées
  régulièrement. La couverture des données est partielle par construction.
- Ne pas reformuler une mention réglementaire, ne pas la traduire, ne pas la
  résumer.
- Ne pas convertir silencieusement entre bases d'expression.
- Ne pas ajouter de dépendance, de framework, d'étape de build ou de backend
  sans que ce soit demandé explicitement.

## Vocabulaire de l'interface

Employer : « point de contrôle », « à vérifier dans l'annexe », « non
déterminé », « non autorisé pour cette fonction », « restriction applicable ».

Éviter : « conforme », « validé », « approuvé », « sûr », « OK », « aucun
problème », « autorisé » (sauf pour dire qu'une substance figure bien dans une
liste positive pour la fonction déclarée, et uniquement avec le numéro
d'entrée à l'appui).

## Avant de considérer un changement terminé

- Aucune valeur réglementaire n'a été introduite dans le JS.
- Chaque nouvelle entrée de données porte ses trois champs de traçabilité.
- Chaque résultat affiche le numéro d'entrée `annexe/entrée`, le règlement
  modificatif et sa date.
- Chaque limite affichée est accompagnée de sa base d'expression.
- Les mentions réglementaires sont affichées verbatim.
- Aucun verdict global n'a été introduit.
- Le rappel « ceci ne remplace pas le rapport de sécurité » et la date de
  mise à jour des données sont toujours visibles.
- `index.html` s'ouvre et fonctionne sans serveur ni build.
