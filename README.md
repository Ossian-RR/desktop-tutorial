# Calculateur HLB

Outil de calcul du HLB requis d'une phase huileuse et du dosage d'un couple
d'émulsifiants, pour la formulation cosmétique.

- `index.html` — l'outil complet : HTML, CSS et JS inline, aucune dépendance.
- `data.json` — les valeurs chimiques (HLB requis des corps gras, HLB des
  émulsifiants), séparées du code pour être corrigées sans y toucher.
- `CLAUDE.md` — la spécification : chimie de référence, domaine de validité,
  contrat de données, cas de vérification.

## Utilisation

Le navigateur refuse de lire `data.json` quand la page est ouverte par
double-clic (`file://`). Servir le dossier :

```
python3 -m http.server 8000
```

puis ouvrir http://localhost:8000

## Portée

Le HLB de Griffin ne décrit que les tensioactifs non ioniques, et ignore la
PIT, le comportement de phase, les électrolytes et le procédé. Le résultat est
un point de départ à valider au laboratoire, pas une formule validée.
