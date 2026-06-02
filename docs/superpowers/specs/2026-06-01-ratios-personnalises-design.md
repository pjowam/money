# Répartitions personnalisées par dépense — Design

**Date :** 2026-06-01
**Statut :** Validé

## Problème

L'app répartit toutes les dépenses selon un ratio fixe codé en dur (toi 55 % / Audrey 45 %).
On veut pouvoir attribuer un ratio différent à certaines dépenses (ex. vacances en 50/50),
y compris des ratios asymétriques, et que cela se reflète correctement sur le solde courant,
la clôture, l'historique et le recap.

Contraintes retenues :
- Les deux parts sont toujours > 0 (pas de 100/0 — dans ce cas la dépense n'est simplement pas saisie).
- Ratio par défaut conservé : 55/45 (toi/Audrey).

## Modèle de données

Chaque dépense gagne un champ optionnel `ratio: { me, her }` en **pourcentages entiers**
(somme = 100, les deux strictement > 0).

- Champ absent ou `null` → défaut 55/45. **Rétrocompatible** : les dépenses et snapshots
  déjà stockés (sans `ratio`) sont traités comme 55/45.
- À l'enregistrement, le champ `ratio` est **omis** quand il vaut le défaut (données propres).

## Cœur du calcul — helper partagé

On remplace le calcul global `Math.round(total * 0.55)` par une somme dépense par dépense :

```js
function computeShares(expenses) {
  let meS = 0, herS = 0, meT = 0, herT = 0;
  for (const e of expenses) {
    const rMe = (e.ratio?.me ?? 55) / 100;
    const my  = Math.round(e.amount * rMe);
    meS += my;  herS += e.amount - my;
    if (e.payer === 'me') meT += e.amount; else herT += e.amount;
  }
  return { meS, herS, meT, herT, total: meT + herT };
}
```

**Invariant conservé :** pour chaque dépense `part_toi + part_autre === montant`
(part_autre = montant − round(montant × ratioToi)), donc aucun centime perdu globalement.

Ce helper devient la **source unique** pour : `renderBalance`, `buildRecapText`,
le handler de clôture (`btn-settle`), et le rendu du détail d'un solde clôturé.

Le calcul du solde lui-même ne change pas : `diff = meT − meS` détermine qui doit combien à qui.

## Tag à la saisie (composer)

`parseExpense` reconnaît un tag de ratio dans le texte :

- `#50/50` — symétrique.
- `#P60/A40` — explicite (`P` = toi/Pierre, `A` = Audrey).
- `#60/40` — accepté, convention : **premier nombre = toi**.

Comportement :
- Le tag est **retiré du libellé**.
- Validation : exactement deux nombres, somme = 100, les deux > 0.
  Si invalide → le tag est **ignoré silencieusement** (ratio défaut appliqué).
- La preview du composer affiche une **puce de ratio** lorsque le ratio est non-défaut.

## Édition d'une dépense (override ponctuel)

Sur l'écran d'édition (`#scr-edit`/section formulaire), nouvelle section **« Répartition »**
placée sous « Payé par » :

- Présets en toggle : `55/45 (défaut)` · `50/50` · `Autre…`
- `Autre…` révèle deux champs nombre (toi / Audrey) avec validation live
  (somme = 100, les deux > 0). Le bouton **Enregistrer est désactivé** tant que c'est invalide.
- À l'ouverture, le sélecteur reflète le `ratio` courant de la dépense.
- À la sauvegarde, écrit `ratio` (ou l'omet si défaut).

## Affichage dans le fil de discussion

Les bulles de dépenses à ratio **non-défaut** portent un petit badge (ex. `50/50`, `60/40`).
Les bulles au ratio par défaut restent inchangées (épurées).

## Écran solde courant — décomposition (option C)

La ligne fixe actuelle « Sa part · 45 % / Ta part · 55 % » est remplacée par une décomposition
groupée par ratio :

```
Total payé      Toi 320 €        Audrey 180 €

Répartition
  55/45  · 3 dép.    Toi 220 €  ·  Audrey 180 €
  50/50  · 1 dép.    Toi 50 €   ·  Audrey 50 €
  ───────────────────────────────────────────
  Total parts        Toi 270 €  ·  Audrey 230 €

┌─────────────────────────────┐
│   Audrey te doit  50,00 €   │
└─────────────────────────────┘
```

- Une ligne par ratio distinct : `<ratio> · <n> dép.` + les parts sommées du groupe.
- Le défaut 55/45 affiché en premier, puis les autres.
- Divider, puis ligne **Total parts** (= `meS` / `herS`).
- La ligne « Total payé » (meT / herT) et la carte finale (qui doit à qui) sont inchangées.

**Fallback (décision validée) :** quand **toutes** les dépenses sont au ratio défaut
(cas le plus courant), on garde l'affichage actuel à deux colonnes (épuré). La décomposition
n'apparaît que dès qu'il y a **≥ 2 ratios distincts** dans la période.

## Clôture & historique

- Le snapshot de settlement stocke `ratio` par dépense (en plus de id/date/amount/label/payer).
- `meShare`/`herShare` du settlement sont calculés via `computeShares` (plus de `round(total*0.55)`).
- Le détail d'un solde clôturé réutilise le même rendu de décomposition.
- Snapshots anciens (sans `ratio` par dépense) → traités comme 55/45.

## Recap presse-papier

- Les lignes de dépenses non-défaut sont annotées de leur ratio, ex. `Hôtel 200 € (50/50)`.
- L'accroche (qui doit combien) utilise le nouveau `diff` issu de `computeShares`.

## Hors périmètre (YAGNI)

- Saisie groupée : reste au ratio par défaut / payeur sélectionné. Le tag de ratio n'y est pas géré.
- Pas de ratios 100/0.
- Pas de catégories nommées (les groupes sont identifiés par leur ratio seul).
- Pas de pourcentage effectif global (« ta part ≈ 52 % ») — écarté au profit de la décomposition.
