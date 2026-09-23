# Laborator IA — Mini-max cu tăiere alfa-beta (varianta 7)
## Documentație: `minimax_ab.js` și `experiment.js`

**Student:** Vădănescu Ion
**Grupa:** DU2501

Acest README descrie ce fac cele două fișiere JavaScript (Node.js) din implementarea
laboratorului: `minimax_ab.js` (algoritmul propriu-zis) și `experiment.js`
(experimentul statistic cerut de varianta 7).

Parametrii variantei 7:
- **Adâncimea arborelui**: 8 niveluri
- **Lățimea arborelui** (factor de ramificare): 2 (arbore binar)
- **Număr de frunze**: 2⁸ = 256
- **Particularitate**: o parte dintre frunze sunt egale cu 0 și se adaugă valori
  negative; se analizează influența acestui fapt asupra numărului de tăieri alfa-beta.

---

## 1. `minimax_ab.js` — algoritmul

Fișierul de bază, de care depinde `experiment.js`. Nu se rulează de obicei singur în
cadrul experimentului, dar poate fi rulat direct (`node minimax_ab.js`) pentru un test
rapid, pe un singur arbore.

### 1.1 Clasa `Node`

```js
class Node {
    constructor(value = null, children = []) { ... }
    get isLeaf() { return this.children.length === 0; }
}
```

Reprezintă un nod al arborelui de joc: rădăcina, un nod intermediar (MAX sau MIN) sau
o frunză. Dacă `children` e gol, nodul e frunză și poartă o valoare numerică (`value`).

### 1.2 `buildTree(depth, branching, leaves)`

Construiește recursiv arborele complet, plasând valorile din `leaves` (un array plat de
256 de numere) în frunzele finale, în ordine, de la stânga la dreapta. Rezultă un arbore
binar cu 9 niveluri (0..8), unde nivelul 8 conține frunzele.

### 1.3 Generatorul pseudo-aleator: `mulberry32`, `randInt`, `shuffleSample`

- **`mulberry32(seed)`** — generator de numere aleatoare **determinist**: pentru
  același `seed`, produce mereu exact aceeași secvență de numere. Necesar pentru
  reproductibilitate (a putea regenera identic același arbore, sau a compara corect
  configurații diferite pe același set de "semințe").
- **`randInt(rng, low, high)`** — un întreg aleator în intervalul `[low, high]`.
- **`shuffleSample(rng, n, k)`** — alege `k` indici distincți, aleatori, din
  `[0, n)` (echivalent cu `random.sample` din Python); folosit pentru a alege *care*
  frunze devin 0.

### 1.4 `generateLeaves(nLeaves, low, high, zeroFraction, seed)`

Generează cele `nLeaves` valori ale frunzelor:
1. generează `nLeaves` numere aleatoare în `[low, high]`;
2. calculează `nZeros = round(nLeaves * zeroFraction)`;
3. alege `nZeros` poziții aleatoare (distincte) și le forțează valoarea la 0.

Aceasta este particularitatea variantei 7: `zeroFraction` controlează procentul de
frunze = 0, iar `low < 0` permite includerea valorilor negative.

### 1.5 `minimax(node, depth, maximizingPlayer, stats)` — algoritmul clasic

Parcurge **exhaustiv** arborele, fără nicio optimizare:
- la fiecare apel incrementează `stats.nodes` (contorul de noduri vizitate);
- dacă nodul e frunză sau s-a atins adâncimea 0, întoarce valoarea frunzei;
- altfel, dacă e rândul lui MAX, ia maximul valorilor întoarse de toți copiii;
  dacă e rândul lui MIN, ia minimul.

Nu există nicio condiție de oprire timpurie — de aceea vizitează întotdeauna toate
cele 511 noduri ale arborelui (2⁹ − 1, pentru adâncime 8 și lățime 2).

### 1.6 `minimaxAB(node, depth, alpha, beta, maximizingPlayer, stats)` — cu tăiere

Aceeași structură ca `minimax`, dar cu doi parametri suplimentari:
- **`alpha`** — cea mai bună valoare garantată pentru MAX, pe drumul curent;
- **`beta`** — cea mai bună valoare garantată pentru MIN, pe drumul curent.

După evaluarea fiecărui copil, se actualizează `alpha` (la nodurile MAX) sau `beta`
(la nodurile MIN). Dacă `beta <= alpha`, restul copiilor nu mai pot influența
rezultatul final: se incrementează `stats.cuts` (o tăiere) și se oprește bucla
(`break`), fără a mai vizita copiii rămași.

Corectitudinea e garantată prin construcție: `minimaxAB` întoarce întotdeauna
**exact aceeași valoare** ca `minimax`, doar că vizitează mai puține noduri.

### 1.7 `runExperiment(depth, branching, leaves, repeats)`

Rulează ambii algoritmi (`minimax` și `minimaxAB`) pe **exact același arbore**
(aceleași frunze), astfel încât comparația să fie corectă. Pentru fiecare, măsoară
timpul de execuție cu `process.hrtime()`, mediat pe `repeats` rulări succesive (implicit
5), ca să reducă zgomotul de măsurare.

La final verifică `resultMM !== resultAB` și aruncă o eroare dacă rezultatele diferă —
o gardă de corectitudine: dacă implementarea alfa-beta ar fi greșită, experimentul se
oprește imediat, în loc să producă rezultate silențios greșite.

Returnează un obiect cu:
- `result` — valoarea optimă calculată la rădăcină;
- `nodesMinimax`, `nodesAlphabeta` — noduri vizitate de fiecare algoritm;
- `cutsAlphabeta` — numărul de tăieri (evenimente `break`);
- `timeMinimax`, `timeAlphabeta` — timp mediu de execuție (ms).

### 1.8 Rularea directă (`node minimax_ab.js`)

Blocul `if (require.main === module) { ... }` construiește un singur arbore de test
(adâncime 8, lățime 2, 256 de frunze, fără zerouri forțate, seed fix = 42) și afișează
în consolă rezultatul unei singure rulări — util pentru verificare rapidă, dar fără
nicio semnificație statistică (vezi `experiment.js` pentru asta).

---

## 2. `experiment.js` — experimentul statistic

Acest fișier folosește funcțiile din `minimax_ab.js` (prin `require('./minimax_ab.js')`)
pentru a răspunde efectiv la cerința variantei 7: *cum influențează procentul de
frunze = 0 (și prezența valorilor negative) numărul de tăieri alfa-beta?*

### 2.1 Parametrii experimentului

```js
const DEPTH = 8;
const BRANCHING = 2;
const N_LEAVES = 256;
const SEEDS = [0, 1, 2, ..., 29];              // 30 de arbori diferiți per configurație
const ZERO_FRACTIONS = [0.0, 0.1, 0.25, 0.5, 0.75, 0.9]; // 6 procente de testat
```

Un singur arbore aleator nu e concludent (poate ieși "norocos" sau "ghinionist" din
întâmplare). De aceea, pentru fiecare procent de zerouri se testează 30 de arbori
diferiți (semințe `0..29`), iar rezultatele se mediază.

### 2.2 `mean(arr)`

Funcție ajutătoare banală: media aritmetică a unui array de numere.

### 2.3 `averageOverSeeds(low, high, zeroFraction, seeds)`

Pentru un `zeroFraction` dat:
1. pentru fiecare `seed` din `seeds`, generează un arbore nou cu `generateLeaves(...)`
   și rulează `runExperiment(...)`;
2. adună într-un array rezultatele fiecărei rulări (noduri MM, noduri AB, tăieri,
   timpi);
3. calculează media pe toate cele 30 de rulări.

În plus, calculează:
```js
reductionPct = 100 * (1 - mNodesAB / mNodesMM)
```
adică procentul de noduri "economisite" de alfa-beta față de mini-max clasic — indicatorul
central al eficienței tăierii.

### 2.4 `printTable(results)`

Afișează în consolă un tabel aliniat, cu o linie pentru fiecare procent din
`ZERO_FRACTIONS`, cu coloanele:

| Coloană | Ce reprezintă |
|---|---|
| `%zero` | procentul de frunze forțate la 0 (parametrul testat pe acel rând) |
| `noduri MM` | media nodurilor vizitate de mini-max clasic (mereu ≈ 511) |
| `noduri AB` | media nodurilor vizitate de mini-max cu alfa-beta |
| `taieri AB` | media numărului de tăieri (`break`-uri) |
| `reducere%` | cât la sută mai puține noduri a vizitat alfa-beta față de MM |
| `t_MM(ms)` | timpul mediu de execuție al mini-max clasic |
| `t_AB(ms)` | timpul mediu de execuție al alfa-beta |

### 2.5 `main()`

Rulează întregul experiment de două ori, pentru a izola efectul zerourilor de efectul
valorilor negative:

- **Seria A** — frunze strict pozitive, `generateLeaves(N_LEAVES, 1, 100, zf, seed)`;
- **Seria B** — frunze cu valori negative incluse, `generateLeaves(N_LEAVES, -100, 100, zf, seed)`.

Pentru fiecare serie, parcurge toate cele 6 procente din `ZERO_FRACTIONS`, apelează
`averageOverSeeds`, și afișează tabelul corespunzător cu `printTable`.

### 2.6 Rularea directă (`node experiment.js`)

Rulează `main()` și tipărește în consolă cele două tabele (Seria A, Seria B). Durează
mai mult decât `minimax_ab.js` de sine stătător, pentru că rulează efectiv
`30 seeds × 6 procente × 2 serii = 360` de experimente complete (fiecare cu câte 3
repetări pentru cronometrare).

### 2.7 Concluzia observată din date

- Numărul de noduri vizitate de mini-max clasic e constant, 511, indiferent de `%zero`.
- Numărul de noduri vizitate de alfa-beta **scade** semnificativ pe măsură ce crește
  `%zero`: de la ~250 (49% din arbore) la 0% zerouri, până la ~100 (20% din arbore) la
  90% zerouri. `reducere%` crește de la ~51% la ~80%.
- Numărul brut de tăieri **scade** ușor (de la ~63 la ~37) pe măsură ce crește `%zero`,
  pentru că arborele efectiv explorat devine mult mai mic — există mai puține ocazii
  (noduri interne vizitate) pentru ca o tăiere să se producă. Indicatorul relevant
  pentru eficiență este `reducere%`, nu numărul brut de tăieri.
- Prezența valorilor negative (Seria B) are un efect secundar, mic, vizibil mai ales la
  `%zero` mic (0–25%), care se estompează la `%zero` mare (75–90%).
- Timpul de execuție urmează îndeaproape numărul de noduri vizitate: mai puține noduri
  → mai puțin timp.

---

## 3. Cum se rulează

```bash
node minimax_ab.js     # test rapid, un singur arbore
node experiment.js     # experimentul complet, cele două tabele
```

Fișierele trebuie să fie în același folder, pentru că `experiment.js` face
`require('./minimax_ab.js')`.
