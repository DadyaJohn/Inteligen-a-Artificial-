# Laborator — Modelarea comportamentului colectiv al agenților (Boids), varianta 3
## Documentație: `index.html`, `style.css` și `script.js`

**Student:** Vădănescu Ion
**Grupa:** DU2501

Acest README descrie ce fac cele trei fișiere ale implementării laboratorului:
`index.html` (structura paginii), `style.css` (aspectul) și `script.js` (simularea
propriu-zisă), precum și experimentele efectuate.

Parametrii variantei 3 (modelarea mișcării pietonilor):
- **Algoritm**: Boids (centrare, aliniere, separare) + regula **Goal Seeking** (mers spre țintă)
- **Mediu**: o încăpere, un coridor, o ieșire la capătul coridorului și o zonă restricționată (stâlp)
- **Agenți**: pietoni cu poziție și viteză inițială aleatoare
- **Particularitate**: se analizează formarea cozilor, a blocajelor și a aglomerărilor în
  locurile înguste, în funcție de numărul de pietoni, lățimea coridorului/ieșirii și
  ponderea regulii de separare.

---

## 1. `index.html` — structura paginii

Conține doar elementele vizibile și legăturile către celelalte două fișiere
(`<link rel="stylesheet" href="style.css">` și `<script src="script.js">`):
- `<canvas id="canvas">` (840 × 500 px) — aici se desenează simularea;
- un `<div id="stats">` — timpul curent, numărul de evacuați, timpul primului și al ultimului pieton;
- panoul cu butoanele **Start/Pauză** și **Resetare** și containerul `<div id="controls">`,
  în care `script.js` generează automat sliderele;
- tabelul **„Experimente efectuate"**, completat automat la finalul fiecărei rulări.

## 2. `style.css` — aspectul

Stilizare simplă: așezarea canvas-ului lângă panoul de control (`flex`), aspectul
sliderelor și al tabelului. Nu conține logică.

---

## 3. `script.js` — simularea

Tot codul de logică. Este împărțit în 10 secțiuni numerotate în comentarii.

### 3.1 `CFG` — parametrii (secțiunea 1)

Un singur obiect cu toți parametrii, grupați:

```js
const CFG = {
  env:     { count: 100, spawnWidth: 200, corridorWidth: 60, exitWidth: 40 },
  agent:   { radius: 4, maxSpeed: 30, maxAccel: 100, neighborRadius: 40, sepDist: 12 },
  weights: { cohesion: 2, alignment: 5, separation: 60, goal: 30, wall: 80, noise: 5 },
  sim:     { speed: 1, dt: 1 / 60, maxTime: 300 }
};
```

- **`env`** — numărul de pietoni, lățimea zonei de start (controlează densitatea),
  lățimea coridorului și a ieșirii.
- **`agent`** — raza agentului, viteza și accelerația maxime, raza în care vede vecinii
  (`neighborRadius`) și distanța sub care începe separarea (`sepDist`).
- **`weights`** — ponderile regulilor din formula `a = wc·C + wa·A + ws·S + wg·G + …`
- **`sim`** — viteza animației, pasul de timp (1/60 s) și timpul maxim după care
  simularea se oprește (300 s).

Scara folosită: **20 px = 1 m**; viteza maximă 30 px/s ≈ 1,5 m/s (mers normal).

### 3.2 `CONTROLS` (secțiunea 1 și 10)

Lista sliderelor: `[grup, cheie, text, min, max, pas, cere_reset]`. La pornire, codul
din secțiunea 10 parcurge lista și creează automat câte un slider pentru fiecare.
Parametrii mediului (`count`, `spawnWidth`, `corridorWidth`, `exitWidth`) resetează
simularea la modificare; ponderile se aplică imediat.

### 3.3 Mediul: `buildWalls`, `closestOnRect`, `hitsWall` (secțiunea 2)

- **`buildWalls()`** — construiește lista `walls`: pereții sunt dreptunghiuri
  `{x, y, w, h}`. Se creează încăperea (400 × 420 px), coridorul (360 px lungime),
  peretele de capăt cu deschiderea de ieșire și un stâlp (zona restricționată).
  Lățimile coridorului și ale ieșirii vin din `CFG.env`; ieșirea nu poate fi mai
  lată decât coridorul.
- **`closestOnRect(r, x, y)`** — cel mai apropiat punct al unui dreptunghi față de un punct.
- **`hitsWall(x, y, rad)`** — adevărat dacă un cerc de rază `rad` atinge vreun perete.

### 3.4 `resetSim()` — crearea agenților (secțiunea 3)

Golește listele, pune timpul pe 0 și creează `CFG.env.count` agenți. Fiecare primește
o poziție aleatoare în zona de start (încercată de până la 200 de ori, ca să nu cadă într-un perete
sau peste alt agent) și o viteză inițială mică (10 px/s) cu direcție aleatoare.
Dacă zona de start e prea mică pentru toți agenții, sunt creați mai puțini
(numărul real se salvează în `total`).

### 3.5 Funcții ajutătoare: `unit`, `limit`, `move` (secțiunea 4)

- **`unit(x, y)`** — vector de lungime 1 (doar direcția).
- **`limit(x, y, max)`** — scurtează vectorul dacă depășește `max`; folosită pentru
  accelerația și viteza maxime.
- **`move(a, dx, dy)`** — mută agentul; dacă mișcarea pe o axă l-ar băga într-un perete,
  mișcarea pe acea axă se anulează și viteza pe ea devine 0.

### 3.6 `computeAcceleration(a)` — regulile Boids (secțiunea 5)

Inima algoritmului. Pentru agentul `a`:
1. parcurge ceilalți agenți și îi reține pe cei din raza `neighborRadius` (vecinii);
   adună pozițiile și vitezele lor, iar pe cei prea apropiați (sub `sepDist`) îi folosește
   pentru separare; numără și câți agenți sunt la mai puțin de 14 px (`a.crowd`),
   folosit la desenarea zonelor de aglomerare;
2. **Centrare (C)** — direcția spre centrul de masă al vecinilor;
3. **Aliniere (A)** — diferența dintre viteza medie a vecinilor și viteza proprie;
4. **Separare (S)** — suma împingerilor în sens opus vecinilor prea apropiați, mai
   puternice cu cât vecinul e mai aproape;
5. **Țintă (G)** — direcția spre `WAYPOINT` (intrarea în coridor) cât timp `x < 440`,
   apoi spre `GOAL` (ieșirea). Punctul intermediar evită ca agenții din colțuri să
   încerce să treacă direct prin perete;
6. **Pereți** — împingere în sens opus oricărui perete aflat la mai puțin de 20 px,
   mai puternică cu cât agentul e mai aproape;
7. **Zgomot** — o mică împingere aleatoare, care ajută agenții să iasă din blocaje.

Suma ponderată dă accelerația, apoi limitată la `maxAccel`:

```
a = wc·C + wa·A + ws·S + wg·G + ww·Pereți + wn·Zgomot
```

### 3.7 `step()` — un pas de simulare (secțiunea 6)

1. calculează accelerația tuturor agenților (`computeAcceleration`);
2. actualizează viteza (`v += a·dt`), o limitează la `maxSpeed` și mută agentul (`move`);
3. **ciocniri dure** — dacă doi agenți se suprapun (distanță sub 2 × rază), sunt depărtați imediat;
4. crește timpul `t` cu `dt`;
5. agenții care au trecut de `x = 790` sunt considerați evacuați: timpul lor se salvează în
   `exitTimes` și sunt scoși din simulare;
6. dacă nu mai e niciun agent sau `t ≥ maxTime`, se apelează `finish()`.

### 3.8 `finish()`, `renderTable()`, `updateStats()` (secțiunea 7)

- **`finish()`** — oprește simularea și adaugă un rând în tabel: numărul de pietoni,
  densitatea (pietoni/m² în zona de start), lățimile coridorului și ale ieșirii,
  ponderea separării, câți au ieșit, **timpul grupului** (momentul ieșirii ultimului
  pieton) și **timpul mediu**. Dacă nu au ieșit toți până la 300 s, timpul grupului
  apare ca „nefinalizat".
- **`renderTable()`** — redesenează tabelul din lista `experiments`.
- **`updateStats()`** — actualizează textul cu timpul, evacuații, primul și ultimul pieton.

### 3.9 `draw()` și bucla principală (secțiunile 8 și 9)

- **`draw()`** desenează zona de ieșire (verde), pereții și stâlpul, **haloul portocaliu**
  în jurul agenților cu cel puțin 5 vecini la mai puțin de 14 px (zone de aglomerare) și
  agenții, colorați după viteză (verde = merge repede, roșu = stă pe loc).
- **`loop()`** rulează `CFG.sim.speed` pași de simulare pe cadru, apoi desenează și
  se reapelează cu `requestAnimationFrame`.

### 3.10 Interfața (secțiunea 10)

Generează sliderele din `CONTROLS` și leagă butoanele: **Start/Pauză**, **Resetare**
și **Șterge tabelul**.

---

## 4. Experimentele

### 4.1 Metodologie

Experimentele au fost rulate **fără interfață grafică**, într-un script auxiliar Node.js
care încarcă `script.js` și execută aceeași funcție `step()` până la `finish()`. Fiecare
configurație a fost rulată de **3 ori** (poziții de start aleatoare diferite; generatorul
aleator nu e fixat cu seed), deci rezultatele nu sunt perfect reproductibile.
Parametrii nemenționați au valorile implicite din `CFG`
(coridor 60 px, ieșire 40 px, zona de start 200 px, 100 pietoni, ponderi: centrare 2,
aliniere 5, separare 60, țintă 30, pereți 80, zgomot 5). Densitatea e calculată
în pietoni/m² în zona de start.

În tabele, **Timp grup** apare pentru fiecare dintre cele 3 rulări (secunde); „>300"
înseamnă că simularea s-a oprit la limita de 300 s cu pietoni rămași în încăpere.
**Timp mediu** e media timpilor de ieșire ai pietonilor (mediată pe cele 3 rulări).

> **Observație importantă despre măsurare:** timpul grupului depinde de ultimul pieton,
> iar acesta variază mult de la o rulare la alta (de exemplu 65 s, 126 s și 228 s la
> aceeași configurație). Timpul mediu este mult mai stabil și este indicatorul pe care
> se sprijină concluziile de mai jos.

### 4.2 Experiment A — numărul de pietoni (zona de start 200 px)

| Pietoni | Densitate (pers/m²) | Timp grup (3 rulări, s) | Timp mediu (s) |
|---|---|---|---|
| 50  | 0,26 | 126,3 / 228,4 / 65,5  | 29,2 |
| 100 | 0,53 | 48,6 / 49,7 / 43,8    | 24,2 |
| 150 | 0,79 | 88,4 / 51,6 / 193,3   | 25,7 |
| 200 | 1,05 | 169,9 / 56,7 / 131,2  | 26,1 |

### 4.3 Experiment B — densitatea inițială (100 pietoni, zona de start variabilă)

| Zona de start (px) | Densitate (pers/m²) | Timp grup (3 rulări, s) | Timp mediu (s) |
|---|---|---|---|
| 100 | 1,05 | 44,2 / 232,9 / 230,0 | 28,4 |
| 200 | 0,53 | 34,6 / 152,5 / 127,8 | 25,4 |
| 380 | 0,28 | 63,6 / 58,6 / 48,9   | 22,0 |

### 4.4 Experiment C — lățimea ieșirii (coridor 60 px, 100 pietoni)

| Ieșire (px) | Timp grup (3 rulări, s) | Evacuați | Timp mediu (s) |
|---|---|---|---|
| 10 | >300 / >300 / >300 | ~90% | 36,6 |
| 20 | 167,3 / 293,3 / 291,2 | 100% | 31,3 |
| 40 | 66,2 / 158,2 / 44,4 | 100% | 25,4 |
| 60 | 47,6 / 44,9 / 71,6 | 100% | 24,3 |

### 4.5 Experiment D — lățimea coridorului (ieșire 30 px, 100 pietoni)

| Coridor (px) | Timp grup (3 rulări, s) | Evacuați | Timp mediu (s) |
|---|---|---|---|
| 30 | 127,3 / 123,9 / >300 | ~99% | 28,0 |
| 60 | 56,7 / 48,1 / 59,3 | 100% | 24,7 |
| 90 | 60,1 / 39,9 / 70,2 | 100% | 25,1 |

### 4.6 Experiment E — ponderea separării (150 pietoni, zona de start 120 px, densitate 1,32 pers/m²)

| w separare | Timp grup (3 rulări, s) | Evacuați | Timp mediu (s) |
|---|---|---|---|
| 0   | 35,1 / 40,9 / >300 | ~99% | 27,0 |
| 10  | >300 / >300 / 171,1 | ~98% | 27,9 |
| 60  | >300 / 124,2 / 156,9 | ~99% | 27,3 |
| 100 | 39,4 / 37,5 / 72,5 | 100% | 26,3 |
| 150 | 209,6 / 198,5 / 133,6 | 100% | 28,4 |

### 4.7 Concluzii observate din date

- **Lățimea ieșirii** are cel mai clar efect. Timpul mediu crește constant pe măsură ce ieșirea se îngustează
  (24,3 s → 25,4 s → 31,3 s → 36,6 s pentru 60, 40, 20 și 10 px), iar la 10 px nici
  în 300 s nu ies toți pietonii. Ieșirea este blocajul principal: în fața ei se formează coada
  și zona de aglomerare (haloul portocaliu).
- **Lățimea coridorului** contează doar când devine foarte îngustă: la 30 px apar blocaje
  (o rulare din 3 nu s-a încheiat), iar între 60 și 90 px diferența e neglijabilă.
  Constrângerea cea mai strâmtă (ieșirea de 30 px) rămâne aceeași, deci coridorul mai larg nu ajută.
- **Densitatea inițială** mărește timpul mediu: 22,0 s (0,28 pers/m²), 25,4 s (0,53) și 28,4 s
  (1,05), la același număr de pietoni, doar prin înghesuirea lor într-o zonă mai mică.
- **Numărul de pietoni** (A) nu arată o tendință clară în timpul mediu (29,2 → 24,2 → 25,7 → 26,1 s):
  diferențele sunt mici față de variația dintre rulări.
- **Ponderea separării** (E) nu arată un efect clar în timpul mediu (26–28 s pentru toate valorile). Singura tendință
  vizibilă în timpul grupului este că **150 a fost constant mai lent** (133–210 s) decât 100 (37–72 s), compatibil
  cu ideea că o respingere prea puternică îi împiedică pe agenți să avanseze în mulțime. Dar
  cu valorile mici (0–60) apar blocaje la unii pietoni, deci nu se poate trage o concluzie
  fermă din doar 3 rulări per valoare.
- **Pietoni rămași în urmă:** în multe rulări, cei mai mulți pietoni ies în 40–60 s, iar timpul grupului e
  mărit de câțiva pietoni care rămân blocați mult timp (ceea ce explică variația mare
  între rulări). Cauza nu a fost verificată în detaliu; probabil sunt blocați în colțuri sau lângă stâlp,
  unde forța spre țintă este anulată de perete, iar doar zgomotul îi mai scoate de acolo. Pentru
  rezultate mai stabile ar fi nevoie de mai multe rulări per configurație.

---

## 5. Cum se rulează

Pune cele trei fișiere (`index.html`, `style.css`, `script.js`) în același folder și deschide
`index.html` în browser. Nu e nevoie de server sau de instalări.

1. Alege parametrii din panoul din dreapta.
2. Apasă **Start**. Poți folosi **Pauză** și **Resetare**.
3. La finalul fiecărei rulări, rezultatul apare automat în tabelul „Experimente efectuate".
4. Schimbă **un singur parametru odată** între rulări, ca să poți compara rândurile.