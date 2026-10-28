# Wealth Score Fuel Gauge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the Wealth Score fuel gauge (a live monthly maturity-ladder instrument) beside the speedometer, plus food home/outside classification, green-dot history, price-tag flags, and two removals (earn-card Edit button, header streak chip).

**Architecture:** Everything lives in the single file `C:\landing-pages\gap\index.html` (inline CSS + JS, no build step). New pure functions (`foodIsHome`, `duelFor`, `debtDaysStr`) compute from `S.txs`; render functions consume them. Two new persisted keys on `S` merge safely through `defaultState()`.

**Tech Stack:** Vanilla JS/CSS in one HTML file. No tests framework, no git — verification is done in a browser served via `python -m http.server` from `C:\landing-pages\gap`, using the DevTools console with demo state.

## Global Constraints

- Colour law: green = safe, red = danger, amber = waiting; purple = repayments; no emoji (SVG icons only)
- Copy is income-first — never tells the user to cut expenses; price tags are exchange rates, not judgments
- The gauge is an instrument (may fall mid-month); green dots are permanent awards
- Repayment counts only when `type==='repayment' && !t.rotating`; investment = expense category `invest`
- Essentials = home food + `family`; non-essentials = `personal` + `friends` + outside food; `business`/`other` excluded
- All money figures use `fmt()` (₹, en-IN); tabular numerals
- Verify each task by loading `http://localhost:<port>/index.html`, hiding overlays (`authOverlay`/`setupOverlay` → `display='none'`), injecting demo `S`, and asserting in the console

**Demo-state snippet used by every verification step (paste once per session):**

```js
document.getElementById('authOverlay').style.display='none';
document.getElementById('setupOverlay').style.display='none';
const _d=new Date(), _y=_d.getFullYear(), _m=String(_d.getMonth()+1).padStart(2,'0');
S=defaultState();
S.settings={setup:true,totalDebt:180000,monthlyFixed:14500};
S.monthTargets[mkey()]=90000;
S.txs=[
 {id:'i1',date:`${_y}-${_m}-01`,amount:45000,type:'income',cat:'retainer',note:'Adroit retainer'},
 {id:'v1',date:`${_y}-${_m}-03`,amount:6000,type:'expense',cat:'invest',fund:'own',note:'SIP'},
 {id:'r1',date:`${_y}-${_m}-05`,amount:8000,type:'repayment',fund:'own',cat:'repayment',rotating:false,note:'CC repayment'},
 {id:'f1',date:`${_y}-${_m}-02`,amount:1200,type:'expense',cat:'food',fund:'own',note:'Groceries'},
 {id:'f2',date:`${_y}-${_m}-04`,amount:900,type:'expense',cat:'food',fund:'own',note:'Biryani zomato'},
 {id:'p1',date:`${_y}-${_m}-06`,amount:2500,type:'expense',cat:'personal',fund:'own',note:'Shopping'},
 {id:'fam1',date:`${_y}-${_m}-07`,amount:3000,type:'expense',cat:'family',fund:'own',note:'Home'},
];
go('dash');
```

Expected duel for this state: repay 8000, invest 6000, ess 1200+3000=4200, non 900+2500=3400 → L1 ✓(8000>3400) L2 ✓(8000>4200) L3 ✓(6000>3400) L4 ✗(6000 ≤ 4200+3400=7600) → **level 3 (low green)**.

---

### Task 1: Duel engine — `foodIsHome` + `duelFor` + state keys

**Files:**
- Modify: `C:\landing-pages\gap\index.html` — `defaultState()` (~line 570) and the helpers area right after `incByUsage()` (~line 700)

**Interfaces:**
- Produces: `foodIsHome(key:string)->bool`; `duelFor(y:number, m0:number)->{repay,invest,ess,non,level}` (m0 = 0-based month); `S.foodMode` object; `S.greenMonths` array; `HOME_WORDS` const

- [ ] **Step 1: Add the two state keys.** In `defaultState()`, after the `tagMap:{},` line add:

```js
  foodMode:{}, // food tag → 'home' | 'out' (overrides the keyword seed)
  greenMonths:[], // months that ended green — permanent dots
```

- [ ] **Step 2: Add the engine functions** directly after the `incByUsage()` function:

```js
// WEALTH SCORE — the monthly duel: repayment/investment vs indulgence
const HOME_WORDS=['grocer','vegetable','veggie','rice','milk','provision','market','maligai','saman','fruit','curd','atta','wheat','oil','dal','sugar','egg'];
function foodIsHome(key){
  const fm=(S.foodMode||{})[key];
  if(fm) return fm==='home';
  return HOME_WORDS.some(w=>key.includes(w));
}
function duelFor(y,m0){
  let repay=0,invest=0,ess=0,non=0;
  S.txs.forEach(t=>{
    const d=new Date(t.date);
    if(d.getMonth()!==m0||d.getFullYear()!==y) return;
    if(t.type==='repayment'&&!t.rotating){repay+=t.amount;return;}
    if(t.type!=='expense') return;
    const key=(t.note||'').trim().toLowerCase();
    if(t.cat==='invest') invest+=t.amount;
    else if(t.cat==='family') ess+=t.amount;
    else if(t.cat==='food'){ if(foodIsHome(key)) ess+=t.amount; else non+=t.amount; }
    else if(t.cat==='personal'||t.cat==='friends') non+=t.amount;
    // business + other: excluded from the duel entirely
  });
  let level=0;
  if(repay>0&&repay>non) level=1;
  if(repay>0&&repay>non&&repay>ess) level=2;
  if(invest>0&&invest>non) level=Math.max(level,3);
  if(invest>0&&invest>ess+non) level=4;
  return {repay,invest,ess,non,level};
}
```

- [ ] **Step 3: Verify in console** (serve, load, paste demo snippet, then):

```js
duelFor(new Date().getFullYear(), new Date().getMonth())
// expect {repay:8000, invest:6000, ess:4200, non:3400, level:3}
foodIsHome('groceries')            // true
foodIsHome('biryani zomato')       // false
S.foodMode['biryani zomato']='home'; foodIsHome('biryani zomato') // true — override wins
delete S.foodMode['biryani zomato'];
```

---

### Task 2: Fuel gauge UI in the earn card + remove the Edit button

**Files:**
- Modify: `C:\landing-pages\gap\index.html` — earn-card HTML (`id="earnCard"` block), `renderEarnSpeed()`, CSS block, and delete `toggleEdit`/`saveTarget` + the `editRow` markup

**Interfaces:**
- Consumes: `duelFor`, `S.greenMonths`
- Produces: `fuelGaugeHTML(level:0|1|2|3|4, dots:number)->string`; gauge element `id="fuelWrap"` with `onclick="go('goal')"`

- [ ] **Step 1: Remove the Edit button and edit row.** In the earn-card HTML, delete the `<button class="edit-btn" onclick="toggleEdit()">Edit</button>` line and the entire `<div class="edit-row" id="editRow">…</div>` block that follows the card. In JS, delete the `toggleEdit` and `saveTarget` functions (Settings keeps its own target input via `st_target`/`saveSettings` — untouched).

- [ ] **Step 2: Add gauge CSS** (after the `.freedom-ends` rule):

```css
/* WEALTH FUEL GAUGE */
.fuel-wrap{display:flex;flex-direction:column;align-items:center;gap:5px;cursor:pointer;flex-shrink:0;width:56px;}
.fuel{display:flex;flex-direction:column-reverse;gap:3px;width:34px;}
.fseg{height:9px;border-radius:3px;background:var(--card2);border:1px solid var(--border);transition:background 0.4s,box-shadow 0.4s;}
.fseg.on-r{background:var(--red);border-color:var(--red);box-shadow:0 0 6px rgba(255,96,87,0.5);}
.fseg.on-a{background:var(--yellow);border-color:var(--yellow);box-shadow:0 0 6px rgba(255,178,36,0.5);}
.fseg.on-g{background:var(--green);border-color:var(--green);box-shadow:0 0 6px rgba(61,220,132,0.55);}
.fuel-lbl{font-size:9px;font-weight:800;letter-spacing:1px;color:var(--muted);}
.fuel-dots{display:flex;gap:3px;flex-wrap:wrap;justify-content:center;max-width:52px;}
.fuel-dots span{width:6px;height:6px;border-radius:50%;background:var(--green);box-shadow:0 0 4px rgba(61,220,132,0.5);}
```

- [ ] **Step 3: Add `fuelGaugeHTML` before `renderEarnSpeed`:**

```js
function fuelGaugeHTML(level,dots){
  const lit=[1,3,4,6,8][level]; // rung → segments lit (bottom-up)
  let segs='';
  for(let i=1;i<=8;i++){
    const on=i<=lit;
    const zone=i<=2?'on-r':i<=4?'on-a':'on-g';
    segs+=`<div class="fseg${on?' '+zone:''}"></div>`;
  }
  const dotSpans=Array.from({length:Math.min(dots,12)},()=>'<span></span>').join('');
  return `<div class="fuel-wrap" id="fuelWrap" onclick="go('goal')" aria-label="Wealth Score — tap for details">
    <div class="fuel">${segs}</div>
    <div class="fuel-lbl">WEALTH</div>
    ${dots?`<div class="fuel-dots">${dotSpans}</div>`:''}
  </div>`;
}
```

- [ ] **Step 4: Cluster layout.** In `renderEarnSpeed`, compute the duel and wrap the existing speedometer `<svg …>` in a flex row with the gauge. Replace the start of the `w.innerHTML=` template — the `<svg viewBox="0 0 200 116" …>` open tag becomes:

```js
  const duel=duelFor(S.vy,S.vm);
  const dots=(S.greenMonths||[]).length;
  w.innerHTML=`
  <div style="display:flex;align-items:center;gap:4px;">
  <svg viewBox="0 0 200 116" style="display:block;flex:1;min-width:0;max-width:230px;margin:8px auto 0;">
```

and the closing `</svg>` becomes:

```js
  </svg>
  ${fuelGaugeHTML(duel.level,dots)}
  </div>
```

(Everything between — arcs, ticks, needle — is unchanged.)

- [ ] **Step 5: Verify.** Reload, paste demo snippet. Expect: gauge right of the speedo with **6 segments lit** (2 red + 2 amber + 2 green = level 3), `WEALTH` label, no Edit button anywhere in the earn card, tapping the gauge lands on the Goal tab. Console: `document.querySelectorAll('#fuelWrap .fseg.on-g').length===2 && document.querySelectorAll('#fuelWrap .fseg[class*=on-]').length===6`.

---

### Task 3: Green-dot awards at month close

**Files:**
- Modify: `C:\landing-pages\gap\index.html` — new `awardGreenDots()` above `renderDash`, one call inside `renderDash`

**Interfaces:**
- Consumes: `duelFor`
- Produces: `awardGreenDots()` — idempotent; fills `S.greenMonths` with `'YYYY-MM'` keys for closed months at level ≥ 3

- [ ] **Step 1: Add the function** directly above `function renderDash(){`:

```js
function awardGreenDots(){
  if(!S.greenMonths) S.greenMonths=[];
  const now=new Date(); let changed=false;
  for(let i=1;i<=12;i++){ // closed months only, up to a year back
    const d=new Date(now.getFullYear(),now.getMonth()-i,1);
    const key=`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}`;
    if(S.greenMonths.includes(key)) continue;
    if(duelFor(d.getFullYear(),d.getMonth()).level>=3){ S.greenMonths.push(key); changed=true; }
  }
  if(changed) save();
}
```

- [ ] **Step 2: Call it** as the first line inside `renderDash()`:

```js
  awardGreenDots();
```

- [ ] **Step 3: Verify.** Paste demo snippet, then add a green *previous* month and re-render:

```js
const pm=new Date(new Date().getFullYear(),new Date().getMonth()-1,1);
const pms=`${pm.getFullYear()}-${String(pm.getMonth()+1).padStart(2,'0')}`;
S.txs.push({id:'pv1',date:`${pms}-10`,amount:5000,type:'expense',cat:'invest',fund:'own',note:'SIP'});
renderDash();
S.greenMonths // expect ['<pms>'] — and one green dot visible under the gauge
renderDash(); S.greenMonths.length===1 // idempotent — still 1
```

---

### Task 4: Remove the streak chip from the header

**Files:**
- Modify: `C:\landing-pages\gap\index.html` — `.habit-bar` HTML (~line 300) and `updateHabit()`

- [ ] **Step 1: HTML.** Delete `<div class="streak-chip cold" id="streakChip">Streak 0</div>` from the habit-bar. Leave `lastLog` and `consistChip` untouched.

- [ ] **Step 2: JS.** In `updateHabit()`, delete the three `sc` lines:

```js
  const sc=document.getElementById('streakChip');
  sc.textContent=`Streak ${streak}d`;
  sc.className='streak-chip '+(streak>=7?'hot':streak>=3?'warm':'cold');
```

and change the destructuring line to `const {dAgo,consist}=habitData();` (leave `habitData` itself alone — `streak` simply goes unused there).

- [ ] **Step 3: Verify.** Reload + demo snippet: header shows only "Logged today ✓ / Nd ago" text and the % chip; console `document.getElementById('streakChip')===null`; no JS errors.

---

### Task 5: Home/Outside toggle for Food tags in Settings

**Files:**
- Modify: `C:\landing-pages\gap\index.html` — `renderSettings()` tags card rows, new `toggleFoodMode(i)` next to `setTagCat`

**Interfaces:**
- Consumes: `foodIsHome`, `_setTags`
- Produces: `toggleFoodMode(i:number)` writing `S.foodMode[key]`

- [ ] **Step 1: Add the function** after `setTagCat`:

```js
function toggleFoodMode(i){
  const t=_setTags[i]; if(!t) return;
  if(!S.foodMode) S.foodMode={};
  S.foodMode[t.key]=foodIsHome(t.key)?'out':'home';
  save(); renderSettings();
  toast(t.label+' → '+(foodIsHome(t.key)?'Home food':'Outside food'));
}
```

- [ ] **Step 2: Add the button to each Food-tag row.** In the `renderSettings` tags-card template, right after the `</select>` and before the ✕ button, insert:

```js
      ${t.cat==='food'?`<button onclick="toggleFoodMode(${i})" style="background:${foodIsHome(t.key)?'var(--green-soft)':'var(--yellow-soft)'};border:1px solid ${foodIsHome(t.key)?'var(--green-line)':'var(--yellow-line)'};border-radius:8px;color:${foodIsHome(t.key)?'var(--green)':'var(--yellow)'};font-size:11px;font-weight:700;padding:8px 9px;cursor:pointer;">${foodIsHome(t.key)?'Home':'Out'}</button>`:''}
```

- [ ] **Step 3: Verify.** Demo snippet → Settings tab: the two Food tags show toggles — Groceries = green "Home", "Biryani zomato" = amber "Out". Tap Out→Home; console `S.foodMode['biryani zomato']==='home'`; back on Home tab the gauge recomputes (level may rise since non-essentials shrink).

---

### Task 6: Price tags — big-ticket at logging + everyday line on the dashboard

**Files:**
- Modify: `C:\landing-pages\gap\index.html` — new helpers near `calcSnapshot`, `finalizeTx` flash line, `renderDash` categories area

**Interfaces:**
- Consumes: `calcSnapshot`, `duelFor`, `fmt`
- Produces: `debtDaysStr(amount:number)->string|null`; `avgMonthlyIncome()->number`

- [ ] **Step 1: Add helpers** directly after `calcSnapshot`:

```js
function avgMonthlyIncome(){
  const snap=calcSnapshot();
  return snap.n?snap.avgRecv:0;
}
function debtDaysStr(amount){
  const snap=calcSnapshot();
  const surp=snap.n?snap.avgSurplus:snap.currentSurplus;
  if(surp>0){
    const days=Math.round(amount/(surp/30));
    if(days<1) return null;
    return days>=30?('~'+Math.round(days/30*10)/10+' month'+(days>=60?'s':'')+' of debt progress'):(days+' day'+(days===1?'':'s')+' of debt progress');
  }
  const inc=avgMonthlyIncome();
  if(inc>0) return Math.round(amount/inc*100)+'% of a month\'s income';
  return null;
}
```

- [ ] **Step 2: Big-ticket flash line.** In `finalizeTx`, replace the flash label line:

```js
  const lbl=old?'Updated':(curType==='expense'?'Spent — logged':curType==='income'?incomeLbl:tx.rotating?'Card rotation — logged':'Repayment — logged');
```

with:

```js
  let lbl=old?'Updated':(curType==='expense'?'Spent — logged':curType==='income'?incomeLbl:tx.rotating?'Card rotation — logged':'Repayment — logged');
  if(curType==='expense'&&(S.settings.totalDebt||0)>0){
    const ami=avgMonthlyIncome();
    if(ami>0&&amt>ami*0.1){ const dd=debtDaysStr(amt); if(dd) lbl='Spent — logged · '+dd; }
  }
```

- [ ] **Step 3: Everyday dashboard line.** In `renderDash`, inside the categories `else` branch, extend the existing nudge block: after the `const nudge=…` statement add:

```js
    const duelNow=duelFor(S.vy,S.vm);
    const hardTimes=(state==='bad'||state==='wait');
    const dd=duelNow.non>0?debtDaysStr(duelNow.non):null;
    const priceLine=(hardTimes&&dd)
      ?`<div style="font-size:12px;font-weight:600;color:var(--muted);background:var(--card);border:1px solid var(--border);border-radius:10px;padding:9px 12px;margin-bottom:8px;">Non-essentials this month: ${fmt(duelNow.non)} — that's ${dd}</div>`
      :'';
```

and change `cg.innerHTML=nudge+…` to `cg.innerHTML=priceLine+nudge+…`.

- [ ] **Step 4: Verify.** Demo snippet, then: log a ₹6,000 expense (>10% of ₹45k income) → flash shows "Spent — logged · N days of debt progress". Force a hard month (`S.txs=S.txs.filter(t=>t.type!=='income'); renderDash();`) → the non-essentials price line appears above the category bars; restore income → line disappears.

---

### Task 7: Wealth Score breakdown card in the Goal tab

**Files:**
- Modify: `C:\landing-pages\gap\index.html` — `renderGoal()`

**Interfaces:**
- Consumes: `duelFor`, `fuelGaugeHTML` levels semantics, `debtDaysStr`, `S.greenMonths`

- [ ] **Step 1: Build the card.** In `renderGoal()`, before `el.innerHTML=`, add:

```js
  const duel=duelFor(S.vy,S.vm);
  const rungNames=['Red — indulgence is winning','Low amber — repayment beats non-essentials','Mid amber — repayment beats essentials too','Green — investment beats non-essentials','Top green — investment beats everything'];
  let nextHint='';
  if(duel.level===0) nextHint=duel.repay>0?`Repay ${fmt(duel.non-duel.repay+1)} more (own money) to beat non-essentials`:'Log a repayment or investment to start filling the tank';
  else if(duel.level===1) nextHint=`Repay ${fmt(duel.ess-duel.repay+1)} more to beat essentials too`;
  else if(duel.level===2) nextHint=`Invest ${fmt(duel.non-duel.invest+1)} more to enter green`;
  else if(duel.level===3) nextHint=`Invest ${fmt(duel.ess+duel.non-duel.invest+1)} more to top the tank`;
  else nextHint='Tank full — hold the line till month end';
  const dotCount=(S.greenMonths||[]).length;
  const wealthCard=`<div class="card">
    <div class="sec">Wealth Score</div>
    <div style="font-size:14px;font-weight:700;margin-bottom:8px;color:${duel.level>=3?'var(--green)':duel.level>=1?'var(--yellow)':'var(--red)'};">${rungNames[duel.level]}</div>
    <div class="row"><span class="rl">Repayment (own money)</span><span class="rv" style="color:var(--purple)">${fmt(duel.repay)}</span></div>
    <div class="row"><span class="rl">Investment</span><span class="rv cg">${fmt(duel.invest)}</span></div>
    <div class="row"><span class="rl">Non-essentials</span><span class="rv cr">${fmt(duel.non)}</span></div>
    <div class="row"><span class="rl">Essentials (never judged)</span><span class="rv" style="color:var(--muted)">${fmt(duel.ess)}</span></div>
    <div style="font-size:12px;font-weight:600;color:var(--accent);margin-top:10px;">${nextHint}</div>
    ${dotCount?`<div style="font-size:11px;color:var(--muted);margin-top:8px;">Green months: ${dotCount} — every month that ended green, forever.</div>`:''}
  </div>`;
```

- [ ] **Step 2: Insert it** into the existing template: change `gapCard+beCard+accCard+` to `gapCard+wealthCard+beCard+accCard+`.

- [ ] **Step 3: Verify.** Demo snippet → Goal tab: card shows "Green — investment beats non-essentials", the four rows (₹8,000 / ₹6,000 / ₹3,400 / ₹4,200), and the hint "Invest ₹1,601 more to top the tank". No console errors on a fresh empty state either (`S=defaultState(); renderGoal();`).

---

### Task 8: End-to-end pass + copy to testing

- [ ] **Step 1:** Full manual sweep with demo state: Home (gauge + dots + speedo intact, no Edit button, no streak chip), Goal (wealth card), Settings (food toggles), Add (log invest → gauge rises on return; log a party expense that flips the duel → gauge falls — instrument behaviour).
- [ ] **Step 2:** Fresh-state sweep: `S=defaultState()` + setup → no errors, gauge at level 0 with red base segment, caption behaviour via Goal-tab hint.
- [ ] **Step 3:** `Copy-Item C:\landing-pages\gap\index.html C:\landing-pages\gap\testing\index.html -Force` and confirm hashes match.
