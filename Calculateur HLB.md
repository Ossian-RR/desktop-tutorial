# Calculateur HLB — instructions projet

Outil de calcul du HLB requis d'une phase huileuse et du dosage d'un couple
d'émulsifiants.

- **Utilisateur cible** : formulateur cosmétique indépendant.
- **Auteur** : chimiste organicien, pas développeur. Le code doit rester
  lisible et modifiable par quelqu'un qui connaît la chimie, pas le tooling JS.

---

## 1. Stack — non négociable

- Un seul fichier `index.html` : HTML, CSS et JS **inline**.
- Zéro dépendance, zéro framework, zéro CDN, zéro étape de build.
  Pas de npm, pas de bundler, pas de TypeScript, pas de service worker.
- Les données chimiques vivent dans `data.json`, à côté de `index.html`,
  jamais recopiées dans le JS. Corriger une valeur de HLB ne doit jamais
  demander de toucher au code.
- Le projet complet = 2 fichiers versionnés : `index.html` + `data.json`
  (+ ce `CLAUDE.md`).

### Chargement de `data.json` — piège connu

`fetch('data.json')` échoue en `file://` (politique d'origine du navigateur).
L'outil doit donc :

1. tenter `fetch('./data.json')` au chargement ;
2. en cas d'échec, afficher un message explicite qui donne la commande
   `python3 -m http.server 8000` et le lien `http://localhost:8000` ;
3. proposer en secours un `<input type="file">` pour charger `data.json`
   à la main.

Ne **jamais** résoudre ce problème en inlinant les données dans le JS.
Ne jamais laisser l'outil démarrer avec un catalogue vide et silencieux.

---

## 2. Chimie — définitions de référence

Ce sont les seules formules autorisées. Aucune autre corrélation, aucun
modèle alternatif, aucune heuristique maison.

**HLB de Griffin (tensioactif non ionique)**

```
HLB = 20 × Mh / M
```

`Mh` = masse de la portion hydrophile, `M` = masse molaire totale.
Échelle 0–20.

**HLB requis d'une phase huileuse** — moyenne des HLB requis des corps gras,
**pondérée par la masse** :

```
HLB_requis = Σ(mᵢ × HLBrᵢ) / Σ(mᵢ)
```

La phase huileuse n'inclut **que** les corps gras, cires et esters.
Elle exclut l'eau, les actifs hydrosolubles, les conservateurs hydrosolubles
et **les émulsifiants eux-mêmes**. Un ingrédient marqué émulsifiant ne peut
pas entrer dans la somme de la phase huileuse, même si l'utilisateur le range
là : le signaler et l'exclure du calcul.

**HLB d'un mélange d'émulsifiants** — moyenne pondérée par la masse, même
formule.

**Dosage d'un couple** (A = HLB haut, B = HLB bas) pour atteindre une cible :

```
fraction_A = (HLB_cible − HLB_B) / (HLB_A − HLB_B)
fraction_B = 1 − fraction_A
```

Si `fraction_A` sort de `[0, 1]`, **la cible est hors d'atteinte avec ce
couple**. Le dire explicitement, nommer l'intervalle atteignable
`[HLB_B, HLB_A]`, et proposer de changer d'émulsifiant. **Ne pas tronquer,
ne pas clamper, ne pas afficher 0 % ou 100 % comme si c'était une solution.**

Si `HLB_A == HLB_B`, la division est impossible : refuser, ne pas afficher
`Infinity` ni `NaN`.

---

## 3. Repères de dosage (indicatifs, pas des règles)

- Émulsion **H/E** : HLB du système émulsifiant 8–18, en pratique 10–16.
- Émulsion **E/H** : 3–6.
- Solubilisation : 15–18.
- Total émulsifiants : 3–6 % de la formule, ou ~20–25 % de la masse de
  phase huileuse.

Les afficher comme des repères de départ, jamais comme une validation.
Une valeur hors repère mérite une note, pas une erreur bloquante.

---

## 4. Domaine de validité — ce que l'outil doit refuser

Le modèle de Griffin ne décrit **que** les tensioactifs non ioniques.
L'outil **refuse**, il n'estime pas :

- tensioactifs **ioniques** (anioniques, cationiques, amphotères) ;
- émulsifiants **polymériques** (acrylates, carbomères, gommes,
  polyglycéryles réticulés, émulsifiants « à froid » de type polymère) ;
- tout ingrédient dont le champ de type n'est pas `non_ionique`.

Le refus est un message clair : « le HLB de Griffin ne décrit pas ce type
d'émulsifiant », pas un avertissement discret ni une valeur grisée.

**Valeur manquante = erreur, jamais une estimation.** Si un corps gras n'a pas
de HLB requis renseigné pour le type d'émulsion choisi, l'outil s'arrête sur
cet ingrédient et le nomme. Interdiction absolue d'inventer, d'interpoler,
de prendre la valeur d'un ingrédient « proche », ou de retomber sur une
moyenne de catégorie.

---

## 5. Contrat de données — `data.json`

Structure attendue. Tout champ absent est traité comme manquant (→ erreur),
jamais comme zéro.

```json
{
  "version": "2026-09-01",
  "corps_gras": [
    {
      "nom": "Huile de jojoba",
      "inci": "Simmondsia Chinensis Seed Oil",
      "categorie": "huile",
      "hlb_requis_he": 6.5,
      "hlb_requis_eh": null,
      "source": "référence bibliographique"
    }
  ],
  "emulsifiants": [
    {
      "nom": "Polysorbate 80",
      "inci": "Polysorbate 80",
      "type": "non_ionique",
      "hlb": 15.0,
      "source": "référence bibliographique"
    },
    {
      "nom": "Gomme xanthane",
      "inci": "Xanthan Gum",
      "type": "polymerique",
      "hlb": null,
      "source": "hors domaine Griffin"
    }
  ]
}
```

- `categorie` ∈ `huile` | `cire` | `ester` | `beurre` | `silicone`.
- `type` ∈ `non_ionique` | `anionique` | `cationique` | `amphotere` |
  `polymerique`. Seul `non_ionique` est calculable.
- `hlb_requis_he` / `hlb_requis_eh` : `null` autorisé et signifie **inconnu**,
  pas zéro.
- Au chargement, valider le fichier : signaler les entrées mal formées,
  les HLB hors `[0, 20]`, les doublons de `nom`. Un `data.json` invalide
  doit produire un message qui nomme la ligne fautive, pas un plantage.

---

## 6. Comportement de l'outil

Entrées :

1. Type d'émulsion : H/E, E/H, solubilisation.
2. Table de la phase huileuse : ingrédient (liste depuis `data.json`) + masse.
   Unité affichée explicitement (g ou % — une seule, cohérente).
3. Choix du couple d'émulsifiants A et B, depuis `data.json`.
4. Masse totale d'émulsifiants (avec le repère 3–6 % / 20–25 % suggéré,
   modifiable).

Sorties :

- HLB requis de la phase huileuse, avec le détail de la pondération
  (chaque ingrédient : masse, fraction, HLB requis, contribution).
- Fraction et **masse** de A et de B.
- HLB effectif du mélange obtenu, comme contrôle.
- Comparaison au repère du type d'émulsion choisi.

Le calcul doit être vérifiable à la main : montrer les nombres intermédiaires,
pas seulement le résultat.

**Arrondi** : aucun arrondi dans les calculs. On arrondit uniquement à
l'affichage final, à **deux décimales**.

---

## 7. Cadrage du résultat

Le résultat n'est **pas** une formule validée. Le HLB ignore la PIT, le
comportement de phase, les électrolytes, le pH, la viscosité et le procédé.
C'est un point de départ à valider au labo.

Cet avertissement est affiché en permanence à côté du résultat. Pas une
modale que l'on ferme, pas un `<footer>` en gris clair, pas un texte de 10 px.

---

## 8. Interdits

- Inventer, interpoler ou extrapoler une valeur de HLB requis manquante.
- Appliquer Griffin hors du domaine non ionique.
- Tronquer une fraction hors `[0, 1]`.
- Arrondir avant l'affichage final.
- Afficher `NaN`, `Infinity` ou `undefined` à l'écran.
- Inliner `data.json` dans le JS.
- Ajouter une dépendance, un CDN, un build.
- Ajouter une fonctionnalité non demandée (comptes, export cloud, sauvegarde
  distante, télémétrie, analytics). L'outil est local et hors ligne.

---

## 9. Conventions de code

- JS vanilla, fonctions de calcul **pures** et séparées du DOM
  (`hlbRequisPhaseHuileuse(...)`, `dosageCouple(...)`), pour qu'elles soient
  relisibles isolément.
- Commenter la **chimie**, pas la syntaxe : chaque formule porte en commentaire
  sa définition de référence.
- Nommage en français, cohérent avec le vocabulaire du formulateur.
- Pas d'abstraction spéculative : pas de couche de plugins, pas de state
  manager maison. Du code plat et long est préférable à du code court et malin.
- Erreurs : une fonction de calcul lève ou retourne un objet d'erreur nommé ;
  l'UI traduit en message lisible. Jamais de `console.log` comme seul retour.

---

## 10. Cas de vérification

Toute modification du calcul doit reproduire ces cas.

1. **Pondération** — 70 g d'un corps gras à HLBr 6.0 + 30 g d'un corps gras
   à HLBr 12.0 → HLB requis = `7.80`.
2. **Dosage nominal** — A = 15.0, B = 4.3, cible = 12.0 →
   `fraction_A = 0.7196…` → affiché `71.96 %`, `fraction_B = 28.04 %`.
   Contrôle : HLB du mélange = `12.00`.
3. **Cible hors d'atteinte** — A = 15.0, B = 4.3, cible = 16.0 →
   pas de dosage, message nommant l'intervalle atteignable `[4.30, 15.00]`.
4. **Couple dégénéré** — A = 10.0, B = 10.0 → refus, pas de division.
5. **Valeur manquante** — un corps gras avec `hlb_requis_he: null` en mode
   H/E → erreur nommant l'ingrédient, aucun résultat partiel affiché.
6. **Hors domaine** — émulsifiant `type: "polymerique"` → refus explicite.
7. **Émulsifiant dans la phase huileuse** — exclu de la somme, signalé.
