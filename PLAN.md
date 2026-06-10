# ものさし（長さ読みアプリ）実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ものさしの指す位置を「何cm／何cm何mm」で読み取る、横向き専用の子ども向け学習アプリを単一 index.html で作る。最初のホーム画面で3ステージ（ものさしをよむ＝★1〜4／ミリでこたえる／とちゅうからよむ）に分岐する。

**Architecture:** 既存たねまきシリーズと同じく `index.html` 1枚に HTML/CSS/JS をまとめる。JS は「ロジック層（問題生成・判定・換算）」「ものさし描画層（やさしい定規=SVG／木のものさし=画像、共通の `mmToX` 座標系で部品化）」「入力・UI制御層」「画面遷移層（ホーム↔3ステージ）」に論理分割する。描画層は第2弾の計算アプリで再利用できるよう関数境界を明確にする。テストランナーは導入せず、ロジックはページ内 `selftest()` の `console.assert` で検証し、見た目・操作はローカルプレビューで目視確認する。

**Tech Stack:** プレーン HTML / CSS / Vanilla JS（フレームワーク・ビルドなし）。SVG でやさしい定規を描画。竹尺は PNG 画像。GitHub Pages で公開。

検証ツール = `preview_*`（ローカルサーバ起動・スナップショット・コンソールログ）。

**設計書:** `~/Documents/たねまき/ものさし/SPEC.md`

---

## ファイル構成

- 作成: `~/Documents/たねまき/ものさし/index.html` … アプリ本体（HTML/CSS/JS 全部入り）
- 既存: `~/Documents/たねまき/ものさし/assets/takejaku-crop-11cm.png` … 木のものさし（横向き・左0から約11cm・790×733px）
- 既存: `~/Documents/たねまき/ものさし/assets/takejaku-horizontal.png` … 横向き全長（較正の参考）
- 既存: `~/Documents/たねまき/ものさし/assets/takejaku-original.png` … 元画像（縦・参考）
- 作成: `~/Documents/たねまき/ものさし/README.md` … シリーズ慣例の説明ファイル

index.html 内の JS は以下のセクションコメントで区切る（同一ファイル内で責務分離）:
`// === LOGIC ===` / `// === RULER (drawing) ===` / `// === OBJECT ===` / `// === INPUT/UI ===` / `// === MODE ===` / `// === NAV ===` / `// === SELFTEST ===`

座標系の合意（全タスク共通）:
- `X0` = ものさし0の画面x座標、`PER_MM` = 1mmあたりpx。`mmToX(mm) = X0 + mm * PER_MM`
- SVG viewBox は `0 0 600 130`、`X0 = 46`、`PER_MM = 5`（0〜100mm = x46〜546）
- 竹尺画像はこの同じ `mmToX` に画像の0と100mmが一致するよう、Task 9 で `WOOD_PX0` と `WOOD_PX100` を測って合わせる

ステージとレベルの合意:
- `ST.stage` = `'home'` | `'read'` | `'mm'` | `'offset'`
- `'read'` のみ `ST.level`（1〜4）を持つ。`'mm'`・`'offset'` は単発（★なし）
- 問題は `generateProblem(stage, level)` が `{start, end, ask}` を返す（mm単位、0〜99）
- `ask` = `'cm'`（□cm）/ `'cmmm'`（□cm □mm）/ `'mm'`（□□mm だけ）

---

## Task 0: 雛形と横向きゲート

**Files:**
- Create: `~/Documents/たねまき/ものさし/index.html`

- [ ] **Step 1: index.html の骨格を作る**

```html
<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>ものさし（ながさ）</title>
  <style>
    *{ box-sizing:border-box; -webkit-tap-highlight-color:transparent; }
    body{ margin:0; font-family: system-ui,-apple-system,"Hiragino Kaku Gothic ProN","Noto Sans JP",sans-serif;
      background:#f4f6fb; color:#1b1f2a; user-select:none; }
    .wrap{ max-width:980px; margin:0 auto; padding:10px 12px 18px; }
    .card{ background:#fff; border:1px solid #e7eaf3; border-radius:16px; padding:12px; }
    #rotate{ position:fixed; inset:0; z-index:99999; background:#0f172a; color:#fff;
      display:none; align-items:center; justify-content:center; text-align:center; padding:24px; }
    #rotate .inner{ font-size:20px; font-weight:800; line-height:1.7; }
    @media (orientation:portrait){ #rotate{ display:flex; } }
  </style>
</head>
<body>
  <div id="rotate"><div class="inner">📱➡️<br>よこむきに してね</div></div>
  <div class="wrap">
    <div class="card" id="app"></div>
  </div>
  <script>
  (function(){
    "use strict";
    // === LOGIC ===
    // === RULER (drawing) ===
    // === OBJECT ===
    // === INPUT/UI ===
    // === MODE ===
    // === NAV ===
    // === SELFTEST ===
  })();
  </script>
</body>
</html>
```

- [ ] **Step 2: ローカルプレビューで骨格を確認**

`preview_start`（このフォルダを root）→ `preview_snapshot`。
期待: 横向きで空カード表示。縦長にすると「よこむきにしてね」が全面表示。

---

## Task 1: ロジック層（問題生成・判定・自己テスト）

純粋関数だけ。DOMに触れない。第2弾でも流用できる中核。

**Files:**
- Modify: `index.html`（`// === LOGIC ===` と `// === SELFTEST ===`）

- [ ] **Step 1: ロジック関数を実装する**

`// === LOGIC ===` 直下に追加:

```js
function ri(a,b){ return Math.floor(Math.random()*(b-a+1))+a; }

// stage: 'read'(level 1..4) / 'mm' / 'offset' → { start, end, ask }  (mm単位 0..99)
function generateProblem(stage, level){
  if(stage==='mm'){ return { start:0, end:ri(11,99), ask:'mm' }; }
  if(stage==='offset'){
    var s=ri(5,38), e=s+ri(15,55); if(e>99) e=99;
    return { start:s, end:e, ask:'cmmm' };
  }
  // stage 'read'
  if(level===1){ return { start:0, end:ri(1,9)*10, ask:'cm' }; }       // 何cmぴったり
  if(level===2){ return { start:0, end:ri(2,18)*5, ask:'cmmm' }; }     // ◯cm5mm（5mm刻み）
  if(level===3){                                                       // 同じcm台で1mm刻み（0cm台も含む）
    var base=ri(0,9); return { start:0, end:base*10+ri(1,9), ask:'cmmm' };
  }
  // level 4: cmをまたぐ・境界（◯cm9mm/次のcm直前）を厚めに
  var end = (Math.random()<0.5) ? ri(1,9)*10+ri(8,9) : ri(11,99);
  return { start:0, end:end, ask:'cmmm' };
}

function splitLen(lenMm){ return { cm:Math.floor(lenMm/10), mm:lenMm%10, totalMm:lenMm }; }

// 入力(文字列) が正解か。未入力は ''
function judge(prob, inCm, inMm){
  var len = prob.end - prob.start;
  if(prob.ask==='cm'){ return inCm!=='' && parseInt(inCm,10)===len/10; }
  if(prob.ask==='mm'){ return inMm!=='' && parseInt(inMm,10)===len; }
  return inCm!=='' && inMm!=='' &&
    parseInt(inCm,10)===Math.floor(len/10) && parseInt(inMm,10)===len%10;
}
```

- [ ] **Step 2: 自己テストを書く**

`// === SELFTEST ===` 直下に追加:

```js
function selftest(){
  var ok=true, A=function(c,m){ if(!c){ ok=false; console.error('SELFTEST FAIL:',m); } };
  A(splitLen(74).cm===7 && splitLen(74).mm===4, 'splitLen 74 = 7cm4mm');
  A(splitLen(90).cm===9 && splitLen(90).mm===0, 'splitLen 90 = 9cm0mm');
  var p={start:0,end:74,ask:'cmmm'};
  A(judge(p,'7','4')===true,  'judge 7cm4mm correct');
  A(judge(p,'7','5')===false, 'judge 7cm5mm wrong');
  A(judge(p,'7','')===false,  'judge missing mm wrong');
  A(judge({start:0,end:50,ask:'cm'},'5','')===true, 'judge 5cm correct');
  A(judge({start:0,end:74,ask:'mm'},'','74')===true,'judge 74mm correct');
  A(judge({start:20,end:85,ask:'cmmm'},'6','5')===true,'judge start!=0 → 6cm5mm');
  // generateProblem ranges
  for(var i=0;i<300;i++){
    var r1=generateProblem('read',1); A(r1.ask==='cm' && r1.end%10===0 && r1.end>=10 && r1.end<=90,'read L1');
    var r2=generateProblem('read',2); A(r2.ask==='cmmm' && r2.end%5===0 && r2.end<=90,'read L2');
    var r3=generateProblem('read',3); A(r3.ask==='cmmm' && r3.end>=1 && r3.end<=99,'read L3');
    var r4=generateProblem('read',4); A(r4.ask==='cmmm' && r4.end>=11 && r4.end<=99,'read L4');
    var rm=generateProblem('mm');     A(rm.ask==='mm' && rm.end>=11 && rm.end<=99 && rm.start===0,'mm stage');
    var ro=generateProblem('offset'); A(ro.ask==='cmmm' && ro.start>0 && ro.end<=99 && ro.end>ro.start,'offset stage');
  }
  console.log(ok ? 'SELFTEST PASS' : 'SELFTEST FAIL');
  return ok;
}
selftest();
```

- [ ] **Step 3: プレビューのコンソールで自己テスト合格を確認**

`preview_eval: window.location.reload()` → `preview_console_logs`。
期待: `SELFTEST PASS`。FAIL が出たら該当行を修正して再確認。

---

## Task 2: やさしい定規の描画（SVG・部品化）

**Files:**
- Modify: `index.html`（`// === RULER (drawing) ===`）

- [ ] **Step 1: ステージSVGと座標系・描画部品を実装**

`// === RULER (drawing) ===` 直下に追加:

```js
var SVGNS='http://www.w3.org/2000/svg';
var X0=46, PER_MM=5;
function mmToX(mm){ return X0 + mm*PER_MM; }
var stage; // <svg> 要素（ステージ画面のとき DOM にある）
function svgAdd(tag, attrs){ var e=document.createElementNS(SVGNS,tag);
  for(var k in attrs) e.setAttribute(k, attrs[k]); stage.appendChild(e); return e; }
function clearStage(){ while(stage.firstChild) stage.removeChild(stage.firstChild); }

function drawEasyRuler(){
  svgAdd('rect',{x:X0-22,y:64,width:100*PER_MM+44,height:48,rx:3,
    fill:'#eef1f7',stroke:'#c7ccda','stroke-width':1});
  for(var mm=0; mm<=100; mm++){
    var h=(mm%10===0)?17:(mm%5===0)?11:7;
    svgAdd('line',{x1:mmToX(mm),x2:mmToX(mm),y1:64,y2:64+h,
      stroke:'#5b6478','stroke-width':mm%10===0?1.3:0.7});
  }
  for(var c=0; c<=10; c++){
    var t=svgAdd('text',{x:mmToX(c*10),y:104,'text-anchor':'middle','font-size':13,fill:'#1b1f2a'});
    t.textContent=c;
  }
}
```

- [ ] **Step 2: （一時）ステージSVG単体表示で目盛りを確認**

`// === NAV ===` に仮の表示関数を置いて確認する（Task 6 で本実装に置換）:

```js
function enterStage(){
  var app=document.getElementById('app');
  app.innerHTML='<div style="background:#fff;border:1px solid #e7eaf3;border-radius:12px;padding:8px 4px">'+
    '<svg id="stage" viewBox="0 0 600 130" style="width:100%;display:block"></svg></div>';
  stage=document.getElementById('stage');
  clearStage(); drawEasyRuler();
}
enterStage();
```

- [ ] **Step 3: プレビューで目盛りを目視確認**

`preview_eval: window.location.reload()` → `preview_screenshot`。
期待: 0〜10cmの数字つき、1mm刻みが等間隔。左端は0の手前に余白。

---

## Task 3: 測るもの＋三角マーカーの描画

**Files:**
- Modify: `index.html`（`// === OBJECT ===`）

- [ ] **Step 1: 品物のランダム選択とマーカー描画を実装**

`// === OBJECT ===` 直下に追加:

```js
var ITEMS=[
  {name:'りぼん', color:'#ED93B1'},
  {name:'えんぴつ',color:'#F2B441'},
  {name:'ひも',   color:'#7FB5E6'},
  {name:'テープ', color:'#9BD17F'}
];
function pickItem(){ return ITEMS[Math.floor(Math.random()*ITEMS.length)]; }

function drawObject(prob, item){
  var xs=mmToX(prob.start), xe=mmToX(prob.end);
  svgAdd('rect',{x:xs,y:42,width:Math.max(xe-xs,2),height:12,rx:2,fill:item.color});
  svgAdd('path',{d:'M '+xe+' 62 l -7 -13 h 14 z', fill:'#D85A30'});       // 終点・赤
  if(prob.start>0){ svgAdd('path',{d:'M '+xs+' 62 l -6 -11 h 12 z', fill:'#888780'}); } // 始点・灰
}
```

- [ ] **Step 2: enterStage の確認用描画に品物を一時追加**

`enterStage()` の `drawEasyRuler();` の後に一時追加:

```js
drawObject({start:0,end:74,ask:'cmmm'}, ITEMS[0]);
```

- [ ] **Step 3: プレビューで重なりを確認**

`preview_eval: window.location.reload()` → `preview_screenshot`。
期待: リボンが0〜7.4cm、赤三角が7cm4mmちょうどを指す。確認後この一時行は Task 5 で本実装に置換。

---

## Task 4: 答え枠＋テンキー＋入力制御

**Files:**
- Modify: `index.html`（`// === INPUT/UI ===`、`<style>`）

- [ ] **Step 1: 答え枠・テンキーのCSSを追加**

`<style>` に追加:

```css
.abox{ width:54px;height:50px;border:2px solid #c7ccda;border-radius:10px;
  display:inline-flex;align-items:center;justify-content:center;font-size:26px;font-weight:800;
  vertical-align:middle; }
.abox.act{ border-color:#2b66ff;background:#eef2ff; }
.kp{ width:54px;height:50px;font-size:22px;font-weight:800;border:1px solid #c7ccda;
  background:#fff;border-radius:10px;cursor:pointer; }
.kp:active{ transform:translateY(1px); }
.kp.ok{ background:#E1F5EE;color:#085041;border-color:#5DCAA5; }
```

- [ ] **Step 2: 状態オブジェクト・入力ハンドラ・テンキー生成を実装**

`// === INPUT/UI ===` 直下に追加:

```js
var ST={ stage:'home', level:1, ruler:'easy', mode:'practice',
  prob:null, item:null, inCm:'', inMm:'', active:'cm', locked:false,
  streak:0, chN:0, chHit:0 };

function buildKeypad(){
  var kp=document.getElementById('keypad');
  var keys=['1','2','3','4','5','6','7','8','9','back','0','ok'];
  kp.innerHTML='';
  keys.forEach(function(k){
    var b=document.createElement('button'); b.className='kp';
    if(k==='back'){ b.innerHTML='⌫'; b.onclick=onBack; }
    else if(k==='ok'){ b.className='kp ok'; b.innerHTML='OK→'; b.onclick=onSubmit; }
    else { b.textContent=k; b.onclick=function(){ onPress(k); }; }
    kp.appendChild(b);
  });
  document.getElementById('boxCm').onclick=function(){ if(ST.prob.ask!=='mm'){ ST.active='cm'; updateBoxes(); } };
  document.getElementById('boxMm').onclick=function(){ if(ST.prob.ask!=='cm'){ ST.active='mm'; updateBoxes(); } };
}

function layoutBoxes(){
  var ask=ST.prob.ask;
  document.getElementById('wrapCm').style.display=(ask==='mm')?'none':'';
  document.getElementById('wrapMm').style.display=(ask==='cm')?'none':'';
  document.getElementById('boxMm').style.width=(ask==='mm')?'78px':'54px';
}
function updateBoxes(){
  var bc=document.getElementById('boxCm'), bm=document.getElementById('boxMm');
  bc.textContent=ST.inCm; bm.textContent=ST.inMm;
  bc.className='abox'+(ST.active==='cm'?' act':'');
  bm.className='abox'+(ST.active==='mm'?' act':'');
}
function onPress(d){
  if(ST.locked) return;
  if(ST.active==='cm'){ ST.inCm=d; if(ST.prob.ask==='cmmm') ST.active='mm'; }
  else { var max=(ST.prob.ask==='mm')?2:1; if(ST.inMm.length<max) ST.inMm+=d; }
  updateBoxes();
}
function onBack(){
  if(ST.locked) return;
  if(ST.active==='mm'){ if(ST.inMm.length>0) ST.inMm=ST.inMm.slice(0,-1);
    else if(ST.prob.ask==='cmmm') ST.active='cm'; }
  else { ST.inCm=''; }
  updateBoxes();
}
```

注: `onSubmit` は Task 5 で定義する。本Task終了時点では未定義参照のため、Step 3 の確認は Task 5 実装後にまとめて行ってもよい。

- [ ] **Step 3: （確認は Task 5 と合わせて実施）**

この Task はUI部品とハンドラの定義のみ。実際の画面差し込みは Task 6（ステージ画面）で行う。

---

## Task 5: 出題ループ・判定・モード・カウンタ

**Files:**
- Modify: `index.html`（`// === MODE ===`）

- [ ] **Step 1: 再描画と新規出題 newProblem を実装**

`// === MODE ===` 直下に追加:

```js
function renderStage(){
  clearStage();
  if(ST.ruler==='easy') drawEasyRuler(); else drawWoodRuler(); // 木は Task 9
  drawObject(ST.prob, ST.item);
}
function newProblem(){
  ST.prob=generateProblem(ST.stage, ST.level);
  ST.item=pickItem();
  ST.inCm=''; ST.inMm=''; ST.active=(ST.prob.ask==='mm')?'mm':'cm'; ST.locked=false;
  document.getElementById('fb').textContent='';
  layoutBoxes(); renderStage(); updateBoxes(); updateCounter();
}
function updateCounter(){
  var c=document.getElementById('counter');
  if(ST.mode==='practice') c.textContent='れんぞく せいかい: '+ST.streak+'もん';
  else c.textContent='チャレンジ '+ST.chN+'/10　せいかい '+ST.chHit;
}
```

- [ ] **Step 2: 判定とモード分岐 onSubmit を実装**

`// === MODE ===` に追加:

```js
function onSubmit(){
  if(ST.locked) return;
  var ok=judge(ST.prob, ST.inCm, ST.inMm);
  var fb=document.getElementById('fb');
  if(ST.mode==='practice'){
    if(ok){ fb.style.color='#1D9E75'; fb.textContent='○ せいかい！';
      ST.streak++; ST.locked=true; updateCounter(); setTimeout(newProblem,900); }
    else { fb.style.color='#E24B4A'; fb.textContent='ざんねん！ もういっかい';
      ST.streak=0; ST.inCm=''; ST.inMm=''; ST.active=(ST.prob.ask==='mm')?'mm':'cm';
      updateBoxes(); updateCounter(); }
  } else {
    ST.chN++; if(ok){ ST.chHit++; fb.style.color='#1D9E75'; fb.textContent='○'; }
    else { fb.style.color='#E24B4A'; fb.textContent='✗'; }
    ST.locked=true; updateCounter();
    if(ST.chN>=10){ setTimeout(function(){ fb.style.color='#1b1f2a';
      fb.textContent='10もんちゅう '+ST.chHit+'もん せいかい！';
      ST.chN=0; ST.chHit=0; setTimeout(newProblem,1600); }, 700); }
    else setTimeout(newProblem,800);
  }
}
```

- [ ] **Step 3: 確認は Task 6（ステージ画面が組み上がってから）**

newProblem/onSubmit は DOM（#stage,#boxCm 等）を前提とする。Task 6 で画面を作ってから通しで確認する。

---

## Task 6: ホーム画面と3ステージのナビゲーション

**Files:**
- Modify: `index.html`（`// === NAV ===` を本実装に、`<style>`）

- [ ] **Step 1: ホーム画面のCSSを追加**

`<style>` に追加:

```css
.home-title{ font-size:18px;font-weight:800;text-align:center;margin:6px 0 16px; }
.home-grid{ max-width:560px;margin:0 auto;display:flex;flex-direction:column;gap:12px; }
.home-main{ background:#fff;border:2px solid #2b66ff;border-radius:16px;padding:38px 16px;
  text-align:center;display:flex;align-items:center;justify-content:center;gap:14px;cursor:pointer; }
.home-main .t{ font-size:26px;font-weight:800;color:#1b3a8a; }
.home-row{ display:flex;gap:12px; }
.home-sub{ flex:1;background:#fff;border:1px solid #c7ccda;border-radius:12px;padding:13px 12px;
  text-align:center;display:flex;align-items:center;justify-content:center;gap:8px;cursor:pointer; }
.home-sub .t{ font-size:15px;font-weight:700;color:#3a4150; }
.home-sub:active,.home-main:active{ transform:translateY(1px); }
.backbtn{ border:1px solid #c7ccda;background:#fff;border-radius:10px;padding:6px 12px;
  font-size:14px;font-weight:700;cursor:pointer; }
```

- [ ] **Step 2: NAV を本実装（ホーム表示・ステージ入場・もどる）に置き換え**

`// === NAV ===` の仮 `enterStage`（Task 2 で置いた確認用）を削除し、次に置き換え:

```js
function showHome(){
  ST.stage='home';
  var app=document.getElementById('app');
  app.innerHTML=
   '<div class="home-title">どれを えらぶ？</div>'+
   '<div class="home-grid">'+
    '<div class="home-main" data-s="read"><span class="t">ものさしを よむ</span></div>'+
    '<div class="home-row">'+
     '<div class="home-sub" data-s="mm"><span class="t">ミリで こたえる</span></div>'+
     '<div class="home-sub" data-s="offset"><span class="t">とちゅうから よむ</span></div>'+
    '</div></div>';
  app.querySelectorAll('[data-s]').forEach(function(el){
    el.onclick=function(){ enterStage(el.dataset.s); };
  });
}

function enterStage(stageName){
  ST.stage=stageName; ST.level=1; ST.streak=0; ST.chN=0; ST.chHit=0;
  var app=document.getElementById('app');
  app.innerHTML=
   '<div id="controls" style="margin-bottom:7px"></div>'+
   '<div style="background:#fff;border:1px solid #e7eaf3;border-radius:12px;padding:8px 4px">'+
   '<svg id="stage" viewBox="0 0 600 130" style="width:100%;display:block"></svg></div>'+
   '<div style="display:flex;gap:12px;margin-top:10px;align-items:center">'+
    '<div style="flex:1;min-width:0">'+
     '<div style="display:flex;align-items:center;gap:7px;font-size:18px">'+
      '<span id="wrapCm"><span class="abox act" id="boxCm"></span> cm</span>'+
      '<span id="wrapMm"><span class="abox" id="boxMm"></span> mm</span></div>'+
     '<div id="fb" style="height:28px;margin-top:8px;font-size:18px;font-weight:800"></div>'+
     '<div id="counter" style="font-size:14px;color:#5b6478"></div>'+
    '</div>'+
    '<div id="keypad" style="display:grid;grid-template-columns:repeat(3,auto);gap:6px;flex-shrink:0"></div>'+
   '</div>';
  stage=document.getElementById('stage');
  buildControls();   // Task 7
  buildKeypad();
  newProblem();
}

showHome();
```

注: `buildControls` は Task 7 で定義する。Task 6 単体確認のため、暫定的に `function buildControls(){}` を `// === INPUT/UI ===` に置いておき、Task 7 で本実装に差し替える。

- [ ] **Step 3: プレビューでホーム→各ステージ→問題表示を確認**

`preview_eval: window.location.reload()` → `preview_screenshot`（ホーム）。
`preview_click` で「ものさしを よむ」→ `preview_screenshot`（ステージ画面・問題が出る）。
`preview_console_logs` でエラーなしを確認。
期待: ホームは上段大・下段2つ。クリックでステージに入り、ものさし＋リボン＋答え枠＋テンキーが出る。

---

## Task 7: ステージ内の操作バー（★1〜4・ものさし・モード・もどる）

**Files:**
- Modify: `index.html`（`// === INPUT/UI ===` の暫定 `buildControls` を本実装に、`<style>`）

- [ ] **Step 1: セグメントのCSSを追加**

`<style>` に追加:

```css
.bar{ display:flex;flex-wrap:wrap;gap:6px;align-items:center;margin-bottom:7px; }
.bar2{ display:flex;flex-wrap:wrap;gap:14px;align-items:center; }
.seglabel{ font-size:12px;color:#8a90a2;margin-right:5px; }
.lvbtn{ padding:6px 11px;font-size:15px;font-weight:800;border:1px solid #c7ccda;
  background:#fff;border-radius:10px;cursor:pointer; }
.lvbtn.on{ background:#eef2ff;color:#2b66ff;border-color:#2b66ff; }
.seg{ display:inline-flex;border:1px solid #c7ccda;border-radius:10px;overflow:hidden; }
.seg button{ border:none;border-radius:0;font-size:14px;font-weight:700;padding:7px 13px;
  background:#fff;color:#5b6478;cursor:pointer; }
.seg button+button{ border-left:1px solid #c7ccda; }
.seg button.on{ background:#eef2ff;color:#2b66ff; }
```

- [ ] **Step 2: buildControls を本実装に差し替え**

`// === INPUT/UI ===` の暫定 `function buildControls(){}` を次に置き換え:

```js
function buildControls(){
  var c=document.getElementById('controls');
  var lvHtml='';
  if(ST.stage==='read'){
    lvHtml='<div class="bar"><span class="seglabel">レベル</span>'+
      '<button class="lvbtn on" data-lv="1">★1</button>'+
      '<button class="lvbtn" data-lv="2">★2</button>'+
      '<button class="lvbtn" data-lv="3">★3</button>'+
      '<button class="lvbtn" data-lv="4">★4</button></div>';
  }
  c.innerHTML=
   lvHtml+
   '<div class="bar2">'+
    '<span style="display:inline-flex;align-items:center"><span class="seglabel">ものさし</span>'+
     '<span class="seg" id="segRuler">'+
      '<button class="on" data-r="easy">やさしい定規</button>'+
      '<button data-r="wood">木のものさし</button></span></span>'+
    '<span style="display:inline-flex;align-items:center"><span class="seglabel">モード</span>'+
     '<span class="seg" id="segMode">'+
      '<button class="on" data-m="practice">れんしゅう</button>'+
      '<button data-m="challenge">チャレンジ</button></span></span>'+
    '<span style="flex:1"></span>'+
    '<button class="backbtn" id="backHome">← もどる</button></div>';

  if(ST.stage==='read'){
    c.querySelectorAll('.lvbtn').forEach(function(b){ b.onclick=function(){
      c.querySelectorAll('.lvbtn').forEach(function(x){ x.classList.remove('on'); });
      b.classList.add('on'); ST.level=parseInt(b.dataset.lv,10);
      ST.streak=0; ST.chN=0; ST.chHit=0; newProblem(); }; });
  }
  c.querySelectorAll('#segRuler button').forEach(function(b){ b.onclick=function(){
    c.querySelectorAll('#segRuler button').forEach(function(x){ x.classList.remove('on'); });
    b.classList.add('on'); ST.ruler=b.dataset.r; renderStage(); }; });
  c.querySelectorAll('#segMode button').forEach(function(b){ b.onclick=function(){
    c.querySelectorAll('#segMode button').forEach(function(x){ x.classList.remove('on'); });
    b.classList.add('on'); ST.mode=b.dataset.m;
    ST.streak=0; ST.chN=0; ST.chHit=0; newProblem(); }; });
  document.getElementById('backHome').onclick=showHome;
}
```

- [ ] **Step 3: プレビューで操作バーとステージ別表示を確認**

`preview_eval: window.location.reload()`。
- 「ものさしを よむ」に入る → `preview_screenshot`: レベル★1〜4＋ものさし＋モード＋もどる
- 各ボタンを `preview_click` して塗りが移動するか
- 「もどる」でホームへ戻るか
- ホーム→「ミリで こたえる」に入る → `preview_screenshot`: ★が出ず、ものさし＋モード＋もどるだけ。答え枠は □□mm

---

## Task 8: ステージ別・★別の難易度を目視チェック

**Files:**
- 変更なし（動作確認のみ。問題があれば Task 1 の `generateProblem` を調整）

- [ ] **Step 1: ものさしをよむ ★1〜4 を確認**

「ものさしを よむ」で ★1→★4 を `preview_click` 切替、各 `preview_screenshot`。
期待:
- ★1: 何cmぴったり（mm枠なし）
- ★2: 5mm刻み（mmは0か5）
- ★3: 同じcm台の1mm刻み（cmが固定気味で mm が1〜9。ときどき0cm台＝ミリだけ）
- ★4: cmをまたぐ1mm刻み。◯cm9mm・次のcm直前が多め

- [ ] **Step 2: ミリでこたえる／とちゅうから を確認**

- 「ミリで こたえる」: 答え枠が「□□mm」1つ。2桁入力で答えられる（`preview_click` で2桁入れて OK）
- 「とちゅうから よむ」: 灰色マーカーの位置から始まる（start>0）。0〜終点ではなく差を読む

`preview_snapshot` で各確認。

---

## Task 9: 木のものさし（竹尺画像）と切替・較正

**Files:**
- Modify: `index.html`（`// === RULER (drawing) ===` に `drawWoodRuler`）
- 使用: `assets/takejaku-crop-11cm.png`

- [ ] **Step 1: 竹尺画像を背景に描く drawWoodRuler を仮の較正値で実装**

`// === RULER (drawing) ===` に追加（href + xlink:href 併用ヘルパも追加）:

```js
function svgImage(attrs){
  var e=document.createElementNS(SVGNS,'image');
  for(var k in attrs){ if(k!=='href') e.setAttribute(k,attrs[k]); }
  e.setAttribute('href',attrs.href);
  e.setAttributeNS('http://www.w3.org/1999/xlink','xlink:href',attrs.href);
  stage.appendChild(e); return e;
}

var WOOD_IMG='assets/takejaku-crop-11cm.png';
var WOOD_IMG_W=790, WOOD_IMG_H=733;
var WOOD_PX0=8, WOOD_PX100=722;   // 画像内の 0mm/100mm のpx（Step3で実測して上書き）
function drawWoodRuler(){
  var pxPerMm=(WOOD_PX100-WOOD_PX0)/100; // 画像px/mm
  var scale=PER_MM/pxPerMm;              // 画像→SVG倍率
  var imgW=WOOD_IMG_W*scale, imgH=WOOD_IMG_H*scale;
  var imgX=mmToX(0) - WOOD_PX0*scale;    // 画像の0を mmToX(0) に合わせる
  var imgY=64;
  svgImage({ x:imgX, y:imgY, width:imgW, height:imgH,
    href:WOOD_IMG, 'preserveAspectRatio':'none' });
}
```

- [ ] **Step 2: プレビューで木のものさしを表示**

`preview_eval: window.location.reload()` → ステージで「木のものさし」を `preview_click` → `preview_screenshot`。
期待: 黄色い竹尺が出る（この時点でズレは可）。

- [ ] **Step 3: 較正する（重要）**

竹尺の印刷目盛りと `mmToX` を一致させる。較正確認用に一時の検証線を引く:

```js
// 較正確認用（合ったら削除）
for(var c=0;c<=10;c++){ svgAdd('line',{x1:mmToX(c*10),x2:mmToX(c*10),
  y1:60,y2:118,stroke:'rgba(0,120,255,.6)','stroke-width':1}); }
```

手順:
1. `preview_screenshot` で青い検証線が竹尺の各cm目盛り（黒点／マス境界）に乗っているか確認
2. ズレていれば `WOOD_PX0`（画像内0のpx）と `WOOD_PX100`（画像内100mmのpx）を数px単位で調整して再確認
3. 全cmで重なったら検証線のコードを削除

- [ ] **Step 4: 同じ問題を easy⇔wood で読み比べ**

★3で1問出し、ものさしを切替えて `preview_screenshot` 2枚。
期待: 同じ赤三角の位置が、両ものさしで同じmmを指す（較正OK）。

---

## Task 10: 横向きレイアウトの仕上げ

**Files:**
- Modify: `index.html`（`<style>` レスポンシブ）

- [ ] **Step 1: 横向き実寸での詰めを調整**

`preview_resize: 740x360` → `preview_screenshot`。
ものさし・テンキー・答え枠・操作バーが収まり、左右が詰まっているか確認。はみ出す場合は追加:

```css
@media (max-height:420px){
  .wrap{ padding:6px 10px 10px; }
  .card{ padding:8px; }
  .abox,.kp{ width:46px;height:44px;font-size:20px; }
  .seglabel{ display:none; }
  .home-main{ padding:26px 16px; }
}
```

- [ ] **Step 2: タブレット横でも確認**

`preview_resize: 1024x768` → `preview_screenshot`。
期待: 間延びせず中央に収まる。ホーム画面も破綻しない。

---

## Task 11: ホーム追加バナー・README・公開準備

**Files:**
- Modify: `index.html`（A2HS バナー。`とけい/るーれっと/index.html` の `.a2hs` を流用）
- Create: `README.md`

- [ ] **Step 1: 「ホームに追加」バナーを移植**

`~/Documents/たねまき/とけい/るーれっと/index.html` の `.a2hs`〜`/A2HS` のCSSと、バナーHTML・表示制御JSを参考に上部へ追加。文言は「ものさし（ながさ）をホームに追加」に変更。

- [ ] **Step 2: README.md を作る**

```markdown
# ものさし（ながさ）

ものさしの指す位置の長さを「何cm・何cm何mm」で読み取る学習アプリ。
たねまき授業デモツール。横向き専用。

最初に3つから選ぶ:
- ものさしを よむ（★1〜4・メイン）
- ミリで こたえる（何十何mm）
- とちゅうから よむ（0からじゃない長さ）

ものさしは「やさしい定規」「木のものさし（竹尺）」を切替。れんしゅう／チャレンジ（10問連続）。
長さの「読む」専用。計算（足し算引き算）は第2弾の別アプリ。

設計: SPEC.md ／ 計画: PLAN.md
```

- [ ] **Step 3: 全ステージ通し確認**

`preview_eval: window.location.reload()`。ホーム→3ステージ × easy/wood × practice/challenge を一通り操作。
`preview_console_logs` でエラーなし＋`SELFTEST PASS` を確認。

- [ ] **Step 4: 公開とレジストリ追記（ぽこぴぃ確認後）**

GitHub リポジトリ作成 → GitHub Pages 公開（既存シリーズ同手順）。
公開後 `~/Workspace/apps/REGISTRY.md` に1行追記:
`| monosashi-nagasa | ~/Documents/たねまき/ものさし/ | <Pages URL> | ものさし（ながさ）読み取り。ホームで3分岐（よむ★1〜4/ミリでこたえる/とちゅうから）。やさしい定規/竹尺切替・れんしゅう/チャレンジ。読む専用、計算は第2弾 |`

---

## Self-Review（計画チェック結果）

- **Spec coverage:** §1概要=Task0,6 / §2画面（ホーム＋ステージ）=Task0,4,6,10 / §3ものさし2種=Task2,9 / §4の3ステージ＝Task6・★1〜4=Task1,7,8・ミリ=Task1,8・とちゅうから=Task1,8 / §5モード=Task5,7 / §6入力=Task4 / §7判定=Task1,5 / §8測るもの=Task3 / §9部品化=Task2,3,9 / §11公開=Task11。全項目に対応タスクあり。
- **Placeholder scan:** 曖昧指示なし。各コードステップに実コードを記載。暫定 `buildControls(){}`→Task7本実装、仮 `enterStage`→Task6本実装、の置換タイミングを明記済み。
- **Type consistency:** `ST.stage`/`ST.level`/`generateProblem(stage,level)`/`judge`/`splitLen`/`mmToX`/`PER_MM`/`X0`/`drawEasyRuler`/`drawWoodRuler`/`drawObject`/`renderStage`/`newProblem`/`onSubmit`/`buildControls`/`buildKeypad`/`showHome`/`enterStage`/`svgAdd`/`svgImage` は全タスクで同名・同シグネチャ。
- **依存順の注意:** newProblem/onSubmit（Task5）は #stage 等のDOM（Task6）が前提のため、通し確認は Task6 以降。Task5 Step3 でその旨明記。
- **較正リスク:** 竹尺較正（Task9 Step3）は実画像を見ながらの調整。検証線手順を明記済み。
