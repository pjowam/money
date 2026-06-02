# Répartitions personnalisées par dépense — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Permettre d'attribuer un ratio de répartition différent du 55/45 par défaut à des dépenses individuelles (ex. vacances 50/50), avec reflet correct sur le solde courant, la clôture, l'historique et le recap.

**Architecture:** Chaque dépense gagne un champ optionnel `ratio: { me, her }` (pourcentages entiers, somme 100, absent = défaut 55/45). Un helper unique `computeShares()` calcule les parts dépense-par-dépense. Saisie via tag texte (`#50/50`) et override sur l'écran d'édition. L'écran solde affiche une décomposition groupée par ratio dès qu'il y a ≥ 2 ratios distincts.

**Tech Stack:** Application mono-fichier `index.html` (HTML + CSS + JavaScript vanilla, pas de build, pas de framework, persistance `localStorage`). Aucune dépendance.

**Note vérification :** Ce projet n'a ni framework de test ni git. Chaque tâche est vérifiée **manuellement dans le navigateur** (et via la console DevTools pour les fonctions pures). Ouvre `index.html` directement dans un navigateur. Les commits sont **optionnels** (le projet n'est pas versionné) — ignore les étapes de commit, ou initialise git au préalable si tu le souhaites.

**Convention de couleurs/noms existante :** `me` = Pierre (toi, violet/bleu, initiale `P`), `her` = Audrey (rose/orange, initiale `A`). Montants stockés en **centimes** (entiers). `fmtMoney(cents)` formate en `« 12,34 € »`. `escapeHtml(str)` échappe le HTML.

---

## File Structure

Tout est dans `index.html`. Aucune création de fichier. Les modifications par zone :

- **Bloc `<style>`** (~lignes 30-310) : nouvelles classes CSS pour le badge de ratio, la puce de preview et la décomposition.
- **Bloc HTML écran « add/edit »** (~lignes 370-388) : section « Répartition ».
- **Bloc HTML écran « solde »** (~lignes 405-412) : conteneur de décomposition.
- **Bloc HTML écran « solde clôturé »** (~lignes 465-467) : conteneur de décomposition.
- **Bloc `<script>`** : helpers (`computeShares`, etc.), `parseExpense`, `updatePreview`, `submitFromComposer`, `openAddScreen`, `btn-save`, `renderBubbles`, `renderBalance`, `renderSettlementDetail`, handler `btn-settle`, `buildRecapText`.

---

## Task 1 : Helpers de ratio et de calcul des parts

**Files:**
- Modify: `index.html` — ajouter les helpers juste avant `function parseExpense(text)` (~ligne 654).

- [ ] **Step 1 : Ajouter les helpers**

Insère ce bloc juste avant la ligne `function parseExpense(text) {` (~654) :

```javascript
  // ===== Ratios de répartition =====
  // Pourcentage de la part de "me" pour une dépense (défaut 55).
  function ratioMe(e) { return (e.ratio && typeof e.ratio.me === 'number') ? e.ratio.me : 55; }
  function ratioHer(e) { return 100 - ratioMe(e); }
  // Une dépense est au ratio par défaut si elle n'a pas de ratio explicite, ou exactement 55/45.
  function isDefaultRatio(e) { return ratioMe(e) === 55; }
  // Clé d'affichage/regroupement, ex. "50/50" (toi/Audrey).
  function ratioKey(e) { return ratioMe(e) + '/' + ratioHer(e); }

  // Parse un tag de ratio dans un texte libre : #50/50, #P60/A40, #A40/P60, #60/40.
  // Convention sans lettres : premier nombre = toi (me).
  // Retourne { ratio: {me,her}|null, rest } si un tag est présent (ratio=null si invalide), sinon null.
  function parseRatioTag(text) {
    const m = text.match(/#\s*([PApa]?)\s*(\d{1,3})\s*\/\s*([PApa]?)\s*(\d{1,3})/);
    if (!m) return null;
    const rest = (text.slice(0, m.index) + text.slice(m.index + m[0].length)).replace(/\s+/g, ' ').trim();
    const l1 = m[1].toUpperCase(), n1 = +m[2], l2 = m[3].toUpperCase(), n2 = +m[4];
    let me, her;
    if (l1 === 'A' || l2 === 'P') { her = n1; me = n2; }
    else { me = n1; her = n2; }
    if (me > 0 && her > 0 && me + her === 100) return { ratio: { me, her }, rest };
    return { ratio: null, rest };
  }

  // Calcule les parts en sommant dépense par dépense (préserve l'invariant : aucun centime perdu).
  function computeShares(expenses) {
    let meS = 0, herS = 0, meT = 0, herT = 0;
    for (const e of expenses) {
      const my = Math.round(e.amount * ratioMe(e) / 100);
      meS += my;
      herS += e.amount - my;
      if (e.payer === 'me') meT += e.amount; else herT += e.amount;
    }
    return { meS, herS, meT, herT, total: meT + herT };
  }
```

- [ ] **Step 2 : Vérifier dans la console du navigateur**

Ouvre `index.html`, puis dans la console DevTools :

```javascript
parseRatioTag('hôtel 200 #50/50')        // → { ratio: {me:50,her:50}, rest: 'hôtel 200' }
parseRatioTag('#P60/A40 resto')          // → { ratio: {me:60,her:40}, rest: 'resto' }
parseRatioTag('truc #A40/P60')           // → { ratio: {me:60,her:40}, rest: 'truc' }
parseRatioTag('truc #60/40')             // → { ratio: {me:60,her:40}, rest: 'truc' }
parseRatioTag('truc #60/30')             // → { ratio: null, rest: 'truc' }   (somme ≠ 100)
parseRatioTag('courses 12/05 truc')      // → null   (pas de #)
computeShares([{amount:1000,payer:'me'},{amount:500,payer:'her',ratio:{me:50,her:50}}])
// → { meS: 550+250=800? } vérifie : 1000*0.55=550 ; 500*0.50=250 ; meS=800, herS=700, total:1500
```

Attendu : chaque appel renvoie la valeur commentée. `computeShares` doit donner `meS:800, herS:700, meT:1000, herT:500, total:1500`.

- [ ] **Step 3 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): helpers ratio + computeShares"
```

---

## Task 2 : Brancher `computeShares` dans `renderBalance`

**Files:**
- Modify: `index.html` — `renderBalance` (~lignes 1302-1335).

But : remplacer le calcul global `0.55` par `computeShares`, **sans encore changer l'affichage** (la décomposition arrive en Task 7). Le solde de données existantes (toutes au défaut) doit rester identique.

- [ ] **Step 1 : Remplacer les 4 premières lignes de calcul**

Dans `renderBalance` (~1302), remplace :

```javascript
    const active = state.expenses.filter(e => !e.settled);
    const meT = active.filter(e => e.payer === 'me').reduce((s, e) => s + e.amount, 0);
    const herT = active.filter(e => e.payer === 'her').reduce((s, e) => s + e.amount, 0);
    const total = meT + herT;
    const meS = Math.round(total * 0.55), herS = total - meS, diff = meT - meS;
```

par :

```javascript
    const active = state.expenses.filter(e => !e.settled);
    const { meS, herS, meT, herT, total } = computeShares(active);
    const diff = meT - meS;
```

- [ ] **Step 2 : Vérifier dans le navigateur**

Avec des dépenses existantes (toutes au ratio défaut), ouvre l'écran « Solde courant ». Le total, les parts (« Sa part 45 % » / « Ta part 55 % ») et la carte « Audrey te doit / Tu dois » doivent être **identiques** à avant le changement.

Test rapide console : ajoute une dépense 50/50 manuellement puis ré-ouvre le solde :
```javascript
state.expenses.push({id: nextId++, date: fmt(today), amount: 10000, label: 'Test 50/50', payer: 'me', ratio:{me:50,her:50}}); save();
```
Rouvre le solde : la part « toi » doit refléter le mélange (pas un simple 55 % du total). Supprime ensuite la dépense de test.

- [ ] **Step 3 (optionnel) : Commit**

```bash
git add index.html && git commit -m "refactor(ratios): renderBalance via computeShares"
```

---

## Task 3 : Tag de ratio dans la saisie + puce de preview

**Files:**
- Modify: `index.html` — `parseExpense` (~654), `updatePreview` (~681), CSS (~ligne 160).

- [ ] **Step 1 : Parser le tag dans `parseExpense`**

Dans `parseExpense`, juste après la ligne :

```javascript
    let payer = state.composerDefaultPayer || 'me', date = fmt(today), dateLabel = "Aujourd'hui", amount = null, label = '';
```

ajoute :

```javascript
    let ratio = null;
    const rt = parseRatioTag(s);
    if (rt) { ratio = rt.ratio; s = rt.rest; }
```

> Important : ce strip se fait **avant** le parsing de date et de montant, sinon `#50/50` serait pris pour une date (`50/50`) ou un montant (`50`).

Puis, dans le `return` final de `parseExpense` (~678), ajoute `ratio` :

```javascript
    return { amount, date, dateLabel, label, payer, ratio, valid: amount !== null && amount > 0 };
```

- [ ] **Step 2 : Ajouter la puce de ratio dans la preview**

Dans `updatePreview` (~699), remplace la ligne qui construit `bubbleHtml` :

```javascript
    const bubbleHtml = '<div class="preview-bubble" id="preview-bubble">' + labelHtml + '<span class="pb-date">' + p.dateLabel + '</span><span class="pb-amount">' + fmtMoney(p.amount) + '</span></div>';
```

par :

```javascript
    const ratioHtml = (p.ratio && p.ratio.me !== 55) ? '<span class="pb-ratio">' + p.ratio.me + '/' + p.ratio.her + '</span>' : '';
    const bubbleHtml = '<div class="preview-bubble" id="preview-bubble">' + labelHtml + ratioHtml + '<span class="pb-date">' + p.dateLabel + '</span><span class="pb-amount">' + fmtMoney(p.amount) + '</span></div>';
```

- [ ] **Step 3 : Ajouter le CSS de la puce**

Juste après la règle `.preview-chip.label { ... }` (~ligne 158), ajoute :

```css
.pb-ratio { display: inline-flex; align-items: center; font-size: 9.5px; font-weight: 600; padding: 1px 6px; border-radius: 999px; background: rgba(255,180,122,0.18); color: #FFB47A; letter-spacing: 0.2px; }
.ratio-badge { display: inline-flex; align-items: center; font-size: 9px; font-weight: 600; padding: 1px 5px; border-radius: 999px; background: rgba(255,180,122,0.16); color: #FFB47A; margin-top: 4px; letter-spacing: 0.2px; }
```

(la classe `.ratio-badge` servira en Task 6.)

- [ ] **Step 4 : Vérifier dans le navigateur**

Dans la barre de saisie, tape `hôtel 200 #50/50`. La preview doit afficher la bulle « Hôtel · 200,00 € » avec une **puce orange `50/50`**, et le libellé doit être « Hôtel » (sans le `#50/50`). Tape `resto 30 #P70/A30` → puce `70/30`. Tape `truc 10 #60/30` (invalide) → pas de puce, libellé « Truc ».

- [ ] **Step 5 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): tag de saisie + puce preview"
```

---

## Task 4 : Persister le ratio à la création via le composer

**Files:**
- Modify: `index.html` — `submitFromComposer` (~706-717).

- [ ] **Step 1 : Stocker le ratio sur la dépense créée**

Dans `submitFromComposer`, remplace :

```javascript
    state.expenses.push({ id: newId, date: p.date, amount: p.amount, label: p.label || 'Sans nom', payer: p.payer });
```

par :

```javascript
    const exp = { id: newId, date: p.date, amount: p.amount, label: p.label || 'Sans nom', payer: p.payer };
    if (p.ratio && p.ratio.me !== 55) exp.ratio = p.ratio;
    state.expenses.push(exp);
```

- [ ] **Step 2 : Vérifier dans le navigateur**

Tape `hôtel 200 #50/50` puis valide (Entrée). Dans la console :
```javascript
state.expenses[state.expenses.length - 1]
```
La dépense doit contenir `ratio: { me: 50, her: 50 }` et `label: 'Hôtel'`. Tape ensuite `pain 5` (sans tag) → la dépense suivante ne doit **pas** avoir de champ `ratio`.

- [ ] **Step 3 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): persistance du ratio depuis le composer"
```

---

## Task 5 : Sélecteur « Répartition » sur l'écran d'édition

**Files:**
- Modify: `index.html` — HTML écran add (~377-385), `openAddScreen` (~852-878), `btn-save` (~1037-1055), + nouvelle fonction `setRatioMode`.

- [ ] **Step 1 : Ajouter le HTML de la section « Répartition »**

Dans l'écran `#scr-add`, entre le bloc « Payé par » (se terminant ligne 381 par `</div>`) et le `form-field` de la date (ligne 382), insère :

```html
      <p class="payer-section-label">Répartition</p>
      <div class="payer-toggle" id="ratio-toggle">
        <button class="payer-seg active" data-ratio="default">55 / 45</button>
        <button class="payer-seg" data-ratio="5050">50 / 50</button>
        <button class="payer-seg" data-ratio="custom">Autre…</button>
      </div>
      <div class="form-field" id="ratio-custom-field" style="display:none">
        <div class="form-field-content" style="display:flex; gap:12px; align-items:center">
          <span class="form-label">Toi</span>
          <input class="form-input" id="input-ratio-me" type="number" inputmode="numeric" min="1" max="99" style="width:52px; flex:0 0 auto">
          <span class="form-label">Audrey</span>
          <input class="form-input" id="input-ratio-her" type="number" style="width:52px; flex:0 0 auto" disabled>
        </div>
      </div>
```

- [ ] **Step 2 : Ajouter l'état et la fonction `setRatioMode`**

Juste après la fonction `setPayer` (~1033) et son `document.querySelectorAll(...)` (1034) et `setPayer('me')` (1035), ajoute :

```javascript
  // ===== Sélecteur de répartition (écran édition) =====
  let editRatio = null; // null = défaut 55/45 ; sinon { me, her }
  function setRatioMode(mode) {
    const customField = document.getElementById('ratio-custom-field');
    document.querySelectorAll('#ratio-toggle .payer-seg').forEach(s => {
      s.classList.toggle('active', s.dataset.ratio === mode);
      s.removeAttribute('style');
      if (s.dataset.ratio === mode) {
        s.style.cssText = 'background:linear-gradient(135deg,rgba(255,180,122,0.28) 0%,rgba(255,138,91,0.28) 100%);border-color:rgba(255,180,122,0.5);color:#FFF';
      }
    });
    if (mode === 'default') { editRatio = null; customField.style.display = 'none'; }
    else if (mode === '5050') { editRatio = { me: 50, her: 50 }; customField.style.display = 'none'; }
    else { // custom
      customField.style.display = 'flex';
      const meVal = (editRatio && editRatio.me !== 55) ? editRatio.me : 60;
      document.getElementById('input-ratio-me').value = meVal;
      document.getElementById('input-ratio-her').value = 100 - meVal;
      editRatio = { me: meVal, her: 100 - meVal };
    }
    updateSaveEnabled();
  }
  // Sélection d'un préset
  document.querySelectorAll('#ratio-toggle .payer-seg').forEach(s =>
    s.addEventListener('click', () => setRatioMode(s.dataset.ratio)));
  // Saisie du ratio "toi" en mode custom : Audrey = 100 - toi
  document.getElementById('input-ratio-me').addEventListener('input', () => {
    let me = parseInt(document.getElementById('input-ratio-me').value, 10);
    if (isNaN(me) || me < 1 || me > 99) { editRatio = null; document.getElementById('input-ratio-her').value = ''; updateSaveEnabled(); return; }
    editRatio = { me, her: 100 - me };
    document.getElementById('input-ratio-her').value = 100 - me;
    updateSaveEnabled();
  });
  // Active/désactive le bouton Enregistrer selon la validité du ratio custom
  function updateSaveEnabled() {
    const customShown = document.getElementById('ratio-custom-field').style.display !== 'none';
    const me = parseInt(document.getElementById('input-ratio-me').value, 10);
    const invalid = customShown && (isNaN(me) || me < 1 || me > 99);
    document.getElementById('btn-save').disabled = invalid;
  }
```

- [ ] **Step 3 : Charger le ratio à l'ouverture de l'écran**

Dans `openAddScreen` (~858-874), dans la branche `if (editing) { ... }` ajoute après `setPayer(editing.payer);` :

```javascript
      editRatio = (editing.ratio && editing.ratio.me !== 55) ? { me: editing.ratio.me, her: editing.ratio.her } : null;
      setRatioMode(!editRatio ? 'default' : (editRatio.me === 50 ? '5050' : 'custom'));
```

et dans la branche `else { ... }` (nouvelle dépense), ajoute après `setPayer('me');` :

```javascript
      editRatio = null;
      setRatioMode('default');
```

- [ ] **Step 4 : Sauvegarder le ratio dans `btn-save`**

Dans le handler `btn-save` (~1042-1049), remplace :

```javascript
    if (state.editingId) {
      const exp = state.expenses.find(e => e.id === state.editingId);
      if (exp) { exp.amount = amount; exp.label = label; exp.date = date; exp.payer = state.currentPayer; }
    } else {
      const newId = nextId++;
      state.expenses.push({ id: newId, date, amount, label, payer: state.currentPayer });
      state.justAddedId = newId;
    }
```

par :

```javascript
    const ratioToSave = (editRatio && editRatio.me !== 55) ? { me: editRatio.me, her: 100 - editRatio.me } : null;
    if (state.editingId) {
      const exp = state.expenses.find(e => e.id === state.editingId);
      if (exp) {
        exp.amount = amount; exp.label = label; exp.date = date; exp.payer = state.currentPayer;
        if (ratioToSave) exp.ratio = ratioToSave; else delete exp.ratio;
      }
    } else {
      const newId = nextId++;
      const exp = { id: newId, date, amount, label, payer: state.currentPayer };
      if (ratioToSave) exp.ratio = ratioToSave;
      state.expenses.push(exp);
      state.justAddedId = newId;
    }
```

- [ ] **Step 5 : Vérifier dans le navigateur**

1. Ajoute une dépense simple, puis appui long dessus → « Modifier ». L'écran d'édition montre la section « Répartition » avec « 55 / 45 » actif.
2. Clique « 50 / 50 », Enregistrer. Ré-édite : « 50 / 50 » doit être actif. Console : la dépense a `ratio: {me:50,her:50}`.
3. Clique « Autre… » → deux champs ; tape `70` dans « Toi » → « Audrey » affiche `30`. Enregistrer. Ré-édite : mode « Autre… » avec 70/30.
4. Repasse sur « 55 / 45 », Enregistrer → la dépense n'a plus de champ `ratio` (console).
5. Mode « Autre… », vide le champ « Toi » → le bouton Enregistrer est désactivé.

- [ ] **Step 6 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): sélecteur de répartition à l'édition"
```

---

## Task 6 : Badge de ratio sur les bulles non-défaut

**Files:**
- Modify: `index.html` — `renderBubbles` (~780). (CSS `.ratio-badge` déjà ajouté en Task 3.)

- [ ] **Step 1 : Ajouter le badge dans la bulle**

Dans `renderBubbles` (~780), remplace la ligne qui construit `bub` :

```javascript
      const bub = '<div class="bubble ' + cls + (options.readOnly ? ' read-only' : '') + '" data-expense-id="' + e.id + '"><div class="label">' + escapeHtml(e.label) + '</div><div class="amount">' + fmtMoney(e.amount) + '</div></div>';
```

par :

```javascript
      const badge = !isDefaultRatio(e) ? '<div class="ratio-badge">' + ratioKey(e) + '</div>' : '';
      const bub = '<div class="bubble ' + cls + (options.readOnly ? ' read-only' : '') + '" data-expense-id="' + e.id + '"><div class="label">' + escapeHtml(e.label) + '</div><div class="amount">' + fmtMoney(e.amount) + '</div>' + badge + '</div>';
```

- [ ] **Step 2 : Vérifier dans le navigateur**

Sur l'écran d'accueil (fil de discussion), une dépense 50/50 doit afficher un petit badge orange `50/50` sous le montant ; une dépense 70/30 affiche `70/30` ; une dépense par défaut n'affiche **aucun** badge.

- [ ] **Step 3 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): badge de ratio sur les bulles"
```

---

## Task 7 : Décomposition sur l'écran solde courant

**Files:**
- Modify: `index.html` — HTML écran solde (~406-409), nouvelle fonction `buildBreakdownHtml`, `renderBalance` (~1310-1311), CSS (~ligne 236).

- [ ] **Step 1 : Remplacer la ligne de parts statique par un conteneur**

Dans `#scr-balance`, remplace le bloc `gc-split` des parts (lignes 406-409) :

```html
        <div class="gc-split">
          <div class="gc-split-col" style="cursor:default"><div class="gc-head"><div class="av-mini her sm">A</div> Sa part · 45 %</div><div class="gc-val" id="share-her">0,00 €</div></div>
          <div class="gc-split-col right" style="cursor:default"><div class="gc-head right">Ta part · 55 % <div class="av-mini me sm">P</div></div><div class="gc-val right" id="share-me">0,00 €</div></div>
        </div>
```

par :

```html
        <div id="bal-shares"></div>
```

- [ ] **Step 2 : Ajouter la fonction `buildBreakdownHtml`**

Juste avant `function renderBalance() {` (~1302), ajoute :

```javascript
  // Construit le HTML des parts pour l'écran solde / le détail clôturé.
  // - allDefault : affichage simple à deux colonnes (avec les "%" fixes 45/55).
  // - sinon : décomposition groupée par ratio + ligne "Total parts".
  function buildBreakdownHtml(expenses) {
    const allDefault = expenses.every(isDefaultRatio);
    const { meS, herS } = computeShares(expenses);
    if (allDefault) {
      return '<div class="gc-split">'
        + '<div class="gc-split-col" style="cursor:default"><div class="gc-head"><div class="av-mini her sm">A</div> Sa part · 45 %</div><div class="gc-val">' + fmtMoney(herS) + '</div></div>'
        + '<div class="gc-split-col right" style="cursor:default"><div class="gc-head right">Ta part · 55 % <div class="av-mini me sm">P</div></div><div class="gc-val right">' + fmtMoney(meS) + '</div></div>'
        + '</div>';
    }
    // Regrouper par ratio (clé = me), défaut (55) d'abord, puis ratio "me" décroissant.
    const groups = new Map();
    for (const e of expenses) {
      const k = ratioMe(e);
      if (!groups.has(k)) groups.set(k, []);
      groups.get(k).push(e);
    }
    const keys = [...groups.keys()].sort((a, b) => (a === 55 ? -1 : b === 55 ? 1 : b - a));
    let rows = '<div class="bd-title">Répartition</div>';
    for (const k of keys) {
      const list = groups.get(k);
      const g = computeShares(list);
      rows += '<div class="bd-row">'
        + '<div class="bd-ratio">' + k + '/' + (100 - k) + '<span class="bd-count">' + list.length + ' dép.</span></div>'
        + '<div class="bd-vals"><span>Toi ' + fmtMoney(g.meS) + '</span> · <span>Audrey ' + fmtMoney(g.herS) + '</span></div>'
        + '</div>';
    }
    rows += '<div class="bd-divider"></div>'
      + '<div class="bd-row bd-total">'
      + '<div class="bd-ratio">Total parts</div>'
      + '<div class="bd-vals"><span>Toi ' + fmtMoney(meS) + '</span> · <span>Audrey ' + fmtMoney(herS) + '</span></div>'
      + '</div>';
    return '<div class="bal-breakdown">' + rows + '</div>';
  }
```

- [ ] **Step 3 : Remplir le conteneur dans `renderBalance`**

Dans `renderBalance`, remplace les deux lignes (~1310-1311) :

```javascript
    document.getElementById('share-me').textContent = fmtMoney(meS);
    document.getElementById('share-her').textContent = fmtMoney(herS);
```

par :

```javascript
    document.getElementById('bal-shares').innerHTML = buildBreakdownHtml(active);
```

> Note : `meS`/`herS` restent utilisés plus bas dans `renderBalance` pour le texte explicatif (`bal-explain`) — ne pas les supprimer du calcul.

- [ ] **Step 4 : Ajouter le CSS de la décomposition**

Juste après la règle `.gc-divider { ... }` (~ligne 236), ajoute :

```css
.bal-breakdown { padding: 2px 0 4px; }
.bd-title { font-size: 11px; color: #8C8CA8; font-weight: 600; margin-bottom: 8px; }
.bd-row { display: flex; align-items: baseline; justify-content: space-between; gap: 10px; padding: 5px 0; }
.bd-ratio { font-size: 12.5px; color: #C9C9DD; font-weight: 600; font-variant-numeric: tabular-nums; white-space: nowrap; }
.bd-count { font-size: 10px; color: #8C8CA8; font-weight: 500; margin-left: 7px; }
.bd-vals { font-size: 12px; color: #C9C9DD; text-align: right; font-variant-numeric: tabular-nums; }
.bd-divider { height: 0.5px; background: rgba(255,255,255,0.08); margin: 8px 0; }
.bd-total .bd-ratio, .bd-total .bd-vals { color: #FFF; font-weight: 600; }
```

- [ ] **Step 5 : Vérifier dans le navigateur**

1. Avec uniquement des dépenses au défaut : l'écran solde affiche l'ancien rendu « Sa part · 45 % » / « Ta part · 55 % » (inchangé).
2. Ajoute une dépense 50/50 : l'écran solde affiche désormais la décomposition « Répartition » avec une ligne `55/45 · n dép.`, une ligne `50/50 · 1 dép.`, le séparateur et « Total parts ». Les montants « Toi »/« Audrey » de chaque ligne doivent additionner correctement (la somme des lignes = ligne « Total parts »).
3. La carte finale « Audrey te doit / Tu dois » reste correcte.

- [ ] **Step 6 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): décomposition par ratio sur le solde"
```

---

## Task 8 : Clôture (snapshot) + décomposition dans le détail clôturé

**Files:**
- Modify: `index.html` — handler `btn-settle` (~1462-1499), HTML écran clôturé (~465-467), `renderSettlementDetail` (~1269-1300).

- [ ] **Step 1 : Calculer les parts et stocker le ratio dans le snapshot**

Dans le handler `btn-settle` (~1462), remplace :

```javascript
    const meT = active.filter(e => e.payer === 'me').reduce((s, e) => s + e.amount, 0);
    const herT = active.filter(e => e.payer === 'her').reduce((s, e) => s + e.amount, 0);
    const total = meT + herT;
    if (total === 0) return;
    const meS = Math.round(total * 0.55);
    const herS = total - meS;
    const diff = meT - meS;
```

par :

```javascript
    const { meS, herS, meT, herT, total } = computeShares(active);
    if (total === 0) return;
    const diff = meT - meS;
```

Puis, dans l'objet poussé dans `state.settlements`, remplace la ligne `expenses:` (~1490) :

```javascript
      expenses: active.map(e => ({ id: e.id, date: e.date, amount: e.amount, label: e.label, payer: e.payer }))
```

par :

```javascript
      expenses: active.map(e => {
        const o = { id: e.id, date: e.date, amount: e.amount, label: e.label, payer: e.payer };
        if (e.ratio && e.ratio.me !== 55) o.ratio = e.ratio;
        return o;
      })
```

> La ligne `shareRatio: { me: 0.55, her: 0.45 }` (~1488) devient indicative seulement ; on peut la laisser telle quelle (non utilisée par l'affichage, qui recalcule depuis les dépenses).

- [ ] **Step 2 : Ajouter un conteneur de décomposition dans l'écran clôturé**

Dans `#scr-settlement-detail`, dans la deuxième `group-card` (~465-467), remplace :

```html
      <div class="group-card">
        <div class="bal-final-card" id="settlement-card"><div class="gc-center"><p class="bal-final-label" id="settlement-label">—</p><p class="bal-final-amount" id="settlement-amount">0,00<span class="cents"> €</span></p></div></div>
      </div>
```

par :

```html
      <div class="group-card">
        <div id="settlement-shares"></div>
        <div class="bal-final-card" id="settlement-card"><div class="gc-center"><p class="bal-final-label" id="settlement-label">—</p><p class="bal-final-amount" id="settlement-amount">0,00<span class="cents"> €</span></p></div></div>
      </div>
```

- [ ] **Step 3 : Remplir la décomposition dans `renderSettlementDetail` (seulement si non-défaut)**

Dans `renderSettlementDetail`, juste avant la dernière ligne `renderBubbles(...)` (~1299), ajoute (le divider est inclus **dans** le HTML, donc le rendu reste idempotent même réappelé) :

```javascript
    const shares = document.getElementById('settlement-shares');
    const exps = s.expenses || [];
    if (exps.some(e => !isDefaultRatio(e))) {
      shares.innerHTML = buildBreakdownHtml(exps) + '<div class="gc-divider"></div>';
      shares.style.display = 'block';
    } else {
      shares.innerHTML = '';
      shares.style.display = 'none';
    }
```

- [ ] **Step 4 : Vérifier dans le navigateur**

1. Avec une période contenant une dépense 50/50, ouvre le solde et clique « Clôturer cette période ».
2. Va dans l'historique, ouvre le solde clôturé : il doit afficher la décomposition « Répartition » (55/45 + 50/50 + Total parts) au-dessus de la carte « Audrey te devait / Tu devais ». Le montant dû doit correspondre à celui affiché avant la clôture.
3. Console : `state.settlements[state.settlements.length-1].expenses` — la dépense 50/50 doit avoir conservé `ratio: {me:50,her:50}`.
4. Ouvre un ancien solde clôturé (sans ratios) : aucune section décomposition, look inchangé.

- [ ] **Step 5 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): snapshot + décomposition à la clôture"
```

---

## Task 9 : Annoter les ratios dans le recap presse-papier

**Files:**
- Modify: `index.html` — `buildRecapText` (~1338-1377).

- [ ] **Step 1 : Utiliser `computeShares` pour le calcul du recap**

Dans `buildRecapText` (~1341-1345), remplace :

```javascript
    const meT = active.filter(e => e.payer === 'me').reduce((s, e) => s + e.amount, 0);
    const herT = active.filter(e => e.payer === 'her').reduce((s, e) => s + e.amount, 0);
    const total = meT + herT;
    const meS = Math.round(total * 0.55);
    const diff = meT - meS;
```

par :

```javascript
    const { meS, meT, herT, total } = computeShares(active);
    const diff = meT - meS;
```

- [ ] **Step 2 : Annoter les lignes de dépenses non-défaut**

Dans `buildRecapText` (~1372-1374), remplace :

```javascript
      personExpenses.forEach(e => {
        lines.push('• ' + e.label + ' : ' + fmtMoney(e.amount));
      });
```

par :

```javascript
      personExpenses.forEach(e => {
        const tag = !isDefaultRatio(e) ? ' (' + ratioKey(e) + ')' : '';
        lines.push('• ' + e.label + ' : ' + fmtMoney(e.amount) + tag);
      });
```

- [ ] **Step 3 : Vérifier dans le navigateur**

Avec une période contenant une dépense 50/50, ouvre le solde et clique sur la carte finale (« Audrey te doit … ») pour copier le recap. Colle-le dans un éditeur de texte : la ligne de la dépense 50/50 doit se terminer par ` (50/50)`, les dépenses au défaut n'ont pas d'annotation, et le montant dû (accroche) doit correspondre à l'écran.

- [ ] **Step 4 (optionnel) : Commit**

```bash
git add index.html && git commit -m "feat(ratios): annotation des ratios dans le recap"
```

---

## Récapitulatif de la couverture du spec

| Section du spec | Tâche(s) |
|---|---|
| Modèle de données (`ratio` optionnel, rétrocompat) | 1, 4, 5, 8 |
| Helper `computeShares` partagé | 1, 2, 8, 9 |
| Tag de saisie (`#50/50`, `#P60/A40`) + preview | 3, 4 |
| Sélecteur de répartition à l'édition | 5 |
| Badge sur les bulles non-défaut | 6 |
| Décomposition sur le solde courant (+ fallback ≥2 ratios) | 7 |
| Clôture & historique | 8 |
| Recap presse-papier | 9 |
| Hors périmètre (bulk inchangé, pas de 100/0, pas de catégories) | — (aucun changement) |
