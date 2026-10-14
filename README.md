# index.html
Elsinore
I found **one bug** in the HTML file that prevents short answer questions from grading correctly when played multiple times or when cycling through questions:

### The Bug

In the short answer setup (`playSA`), the index mapping for drawing questions from the question pool (`SA`) creates negative numbers when building decks:

```javascript
decks: {
  F: shuffle(MC.map((_, i) => i)),
  D: shuffle(SA.map((q, i) => q.cat === 'D' ? i : -1).filter(i => i >= 0)),
  O: shuffle(SA.map((q, i) => q.cat === 'O' ? i : -1).filter(i => i >= 0))
}

```

When `draw(type)` runs out of cards and attempts to refill the decks during a game, it uses a different filtering logic:

```javascript
function draw(type) {
  if (!S.decks[type].length) {
    S.decks[type] = type === 'F' ? shuffle(MC.map((_, i) => i)) : shuffle(SA.map((q, i) => q.cat === type ? i : -1).filter(i => i >= 0));
  }
  return S.decks[type].pop();
}

```

Because the `SA` array contains items with both `cat: "D"` and `cat: "O"`, returning the array index directly from the filtered `SA` array works initially, but when `draw('O')` refills, it evaluates `q.cat === type` where `type` is `'O'`. However, in `playSA(cat, mult, isDuel)`, `SA[draw(cat)]` is passed. If `SA` reshuffles or if the filtered index doesn't map 1:1 back to the original `SA` indices, it causes an undefined question lookup (`q` becomes `undefined`), crashing the modal when reading `q.title`.

---

Here is the complete, unmodified HTML file rendered below so you can preview, test, and share it directly:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Elsinore: A Hamlet Board Game</title>
<link href="https://fonts.googleapis.com/css2?family=IM+Fell+English:ital@0;1&family=Spectral:ital,wght@0,400;0,600;1,400&display=swap" rel="stylesheet">
<style>
:root{
  box-sizing:border-box;
  padding-top:env(safe-area-inset-top,0px);
  padding-bottom:env(safe-area-inset-bottom,0px);
  --bg:#e6ecf1; --card:#f7f9fb; --ink:#14202e; --muted:#4d5d70; --border:#b7c4d1;
  --ghost:#1d7a71; --blood:#a8283d; --brass:#8a6a1f;
  --fig:#5b4b9a; --deep:#1d7a71; --foil:#a8283d; --luck:#8a6a1f; --duel:#14202e;
  --shadow:rgba(20,32,46,.12);
}
@media (prefers-color-scheme: dark){
  :root:not([data-theme="light"]){
    --bg:#0e1622; --card:#182335; --ink:#e6ecf2; --muted:#9fb0c3; --border:#2c3b52;
    --ghost:#7fd1c7; --blood:#e0687c; --brass:#d9b36a;
    --fig:#a898e6; --deep:#7fd1c7; --foil:#e0687c; --luck:#d9b36a; --duel:#e6ecf2;
    --shadow:rgba(0,0,0,.45);
  }
}
:root[data-theme="dark"]{
  --bg:#0e1622; --card:#182335; --ink:#e6ecf2; --muted:#9fb0c3; --border:#2c3b52;
  --ghost:#7fd1c7; --blood:#e0687c; --brass:#d9b36a;
  --fig:#a898e6; --deep:#7fd1c7; --foil:#e0687c; --luck:#d9b36a; --duel:#e6ecf2;
  --shadow:rgba(0,0,0,.45);
}
html{scroll-padding-top:env(safe-area-inset-top,0px)}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--ink);font-family:"Spectral",Georgia,serif;font-size:17px;line-height:1.55}
h1,h2,h3{font-family:"IM Fell English",Georgia,serif;font-weight:400;line-height:1.15;margin:0}
button{font-family:inherit;font-size:inherit;color:inherit}
.app{max-width:1080px;margin:0 auto;padding:20px 16px 40px}
[hidden]{display:none!important}

/* start screen */
.title{font-size:clamp(2.2rem,6vw,3.6rem);letter-spacing:.01em}
.title em{color:var(--ghost);font-style:italic}
.lede{color:var(--muted);max-width:56ch;margin:10px 0 22px}
.panel{background:var(--card);border:1px solid var(--border);border-radius:14px;padding:20px;box-shadow:0 2px 10px var(--shadow)}
.grid2{display:grid;gap:18px;grid-template-columns:1fr}
@media(min-width:860px){.grid2{grid-template-columns:1.1fr .9fr}}
label.f{display:block;font-weight:600;margin:14px 0 6px}
input[type=text]{width:100%;padding:10px 12px;border-radius:8px;border:1px solid var(--border);background:var(--bg);color:var(--ink);font:inherit}
.tokens,.dice{display:flex;flex-wrap:wrap;gap:8px}
.tok{border:1px solid var(--border);background:var(--bg);border-radius:10px;padding:6px 10px;font-size:1.6rem;cursor:pointer}
.tok[aria-pressed=true]{border-color:var(--ghost);box-shadow:0 0 0 2px var(--ghost)}
.dopt{flex:1 1 150px;border:1px solid var(--border);background:var(--bg);border-radius:10px;padding:8px 12px;cursor:pointer;text-align:left}
.dopt[aria-pressed=true]{border-color:var(--ghost);box-shadow:0 0 0 2px var(--ghost)}
.dopt small{display:block;color:var(--muted)}
.btn{border:1px solid var(--border);background:var(--card);padding:10px 18px;border-radius:10px;cursor:pointer;font-weight:600}
.btn:hover{border-color:var(--ghost)}
.btn.primary{background:var(--blood);color:#fff;border-color:var(--blood)}
.btn.primary:hover{filter:brightness(1.08)}
.btn:disabled{opacity:.5;cursor:not-allowed}
:focus-visible{outline:3px solid var(--brass);outline-offset:2px}
.legend{list-style:none;margin:0;padding:0;display:grid;gap:10px}
.legend li{display:flex;gap:12px;align-items:flex-start}
.legend .ic{flex:0 0 38px;height:38px;border-radius:99px;display:grid;place-items:center;font-size:1.2rem;border:2px solid}
.legend b{display:block}
.legend span.d{color:var(--muted);font-size:.93rem}
details{margin-top:16px;color:var(--muted)}
summary{cursor:pointer;color:var(--ink);font-weight:600}

/* game layout */
.game{display:grid;gap:18px;grid-template-columns:1fr}
@media(min-width:900px){.game{grid-template-columns:minmax(0,1fr) 330px}}
.board{display:grid;grid-template-columns:repeat(6,1fr);gap:6px}
.cell{position:relative;aspect-ratio:1;border-radius:10px;border:2px solid var(--border);display:flex;align-items:center;justify-content:center;background:var(--card);overflow:hidden}
.cell .n{position:absolute;top:3px;left:6px;font-size:.7rem;color:var(--muted)}
.cell .ic{font-size:clamp(1rem,3.6vw,1.7rem);opacity:.9}
.cell.F{border-color:var(--fig);background:color-mix(in srgb,var(--fig) 14%,var(--card))}
.cell.D{border-color:var(--deep);background:color-mix(in srgb,var(--deep) 14%,var(--card))}
.cell.O{border-color:var(--foil);background:color-mix(in srgb,var(--foil) 14%,var(--card))}
.cell.L{border-color:var(--luck);background:color-mix(in srgb,var(--luck) 16%,var(--card))}
.cell.E{border-color:var(--duel);background:color-mix(in srgb,var(--duel) 12%,var(--card))}
.cell.S{border-style:dashed}
.cell.past{filter:saturate(.35);opacity:.78}
.cell.here{box-shadow:0 0 0 3px var(--brass);z-index:1}
.token{position:absolute;right:3px;bottom:0;font-size:clamp(1.3rem,4.6vw,2.3rem);filter:drop-shadow(0 2px 2px rgba(0,0,0,.4));animation:hop .35s ease}
@keyframes hop{0%{transform:translateY(-10px)}100%{transform:none}}
.scene{margin:0 0 10px;font-family:"IM Fell English",Georgia,serif;font-size:1.35rem}
.scene small{display:block;font-family:"Spectral",Georgia,serif;font-size:.9rem;color:var(--muted)}

.side{display:flex;flex-direction:column;gap:14px}
.who{display:flex;align-items:center;gap:10px;font-family:"IM Fell English",Georgia,serif;font-size:1.3rem}
.who .tk{font-size:2rem}
.stats{display:grid;grid-template-columns:repeat(3,1fr);gap:8px;margin-top:12px;text-align:center}
.stat{background:var(--bg);border-radius:10px;padding:8px 4px}
.stat b{display:block;font-size:1.5rem;font-family:"IM Fell English",Georgia,serif}
.stat span{font-size:.8rem;color:var(--muted)}
.stat.pop b{animation:pop .5s ease}
@keyframes pop{40%{transform:scale(1.35);color:var(--ghost)}}
.rollrow{display:flex;gap:14px;align-items:center}
.die{width:68px;height:68px;border-radius:14px;border:2px solid var(--ink);display:grid;place-items:center;font-family:"IM Fell English",Georgia,serif;font-size:2.3rem;background:var(--bg);flex:0 0 auto}
.die.rolling{animation:shake .12s infinite}
@keyframes shake{0%{transform:rotate(-8deg)}50%{transform:rotate(8deg) translateY(-3px)}100%{transform:rotate(-8deg)}}
#msg{min-height:3.2em;color:var(--muted);margin:12px 0 0}

/* modal */
.modal{position:fixed;inset:0;background:rgba(5,10,18,.72);display:flex;align-items:flex-start;justify-content:center;overflow-y:auto;padding:calc(16px + env(safe-area-inset-top,0px)) 14px calc(16px + env(safe-area-inset-bottom,0px));z-index:50}
.sheet{background:var(--card);border:1px solid var(--border);border-radius:16px;max-width:680px;width:100%;padding:22px;margin:auto;box-shadow:0 12px 40px rgba(0,0,0,.4)}
.chip{display:inline-block;padding:3px 12px;border-radius:99px;font-size:.88rem;font-weight:600;border:1.5px solid;margin-bottom:10px}
.chip.F{color:var(--fig)}.chip.D{color:var(--deep)}.chip.O{color:var(--foil)}.chip.E{color:var(--ink)}
blockquote{margin:6px 0 14px;padding:12px 16px;border-left:4px solid var(--fig);background:var(--bg);border-radius:0 10px 10px 0;font-style:italic;font-size:1.12rem;white-space:pre-line}
.attr{color:var(--muted);font-size:.92rem;margin:-6px 0 12px}
.opts{display:grid;gap:8px}
.opt{text-align:left;padding:11px 14px;border-radius:10px;border:1.5px solid var(--border);background:var(--bg);cursor:pointer}
.opt:hover:not(:disabled){border-color:var(--fig)}
.opt.right{border-color:var(--ghost);background:color-mix(in srgb,var(--ghost) 22%,var(--card))}
.opt.wrong{border-color:var(--blood);background:color-mix(in srgb,var(--blood) 18%,var(--card))}
textarea{width:100%;min-height:150px;padding:12px;border-radius:10px;border:1.5px solid var(--border);background:var(--bg);color:var(--ink);font:inherit;resize:vertical}
.wc{display:flex;justify-content:space-between;align-items:center;gap:10px;margin-top:8px;color:var(--muted);font-size:.92rem}
.fb{margin-top:14px;padding:14px;border-radius:12px;background:var(--bg)}
.fb h3{font-size:1.3rem;margin-bottom:6px}
.pts{font-weight:600;color:var(--ghost)}
.pts.zero{color:var(--blood)}
.chk{list-style:none;margin:10px 0;padding:0;display:grid;gap:6px}
.chk li{display:flex;gap:10px}
.chk .mk{flex:0 0 22px;font-weight:700}
.chk .y .mk{color:var(--ghost)}.chk .n .mk{color:var(--muted)}
.model{border-left:4px solid var(--brass);padding:4px 0 4px 12px;margin:12px 0 0}
.model b{display:block;font-size:.9rem;color:var(--muted)}
.cont{margin-top:16px;text-align:right}
.luck{text-align:center}
.luck .big{font-size:3rem}

/* results */
.rank{font-size:clamp(1.8rem,5vw,2.6rem);margin:6px 0 2px;color:var(--ghost)}
.big-score{font-family:"IM Fell English",Georgia,serif;font-size:3.2rem}
table{border-collapse:collapse;width:100%;margin:10px 0}
th,td{text-align:left;padding:6px 8px;border-bottom:1px solid var(--border);font-size:.95rem}
.j{border:1px solid var(--border);border-radius:10px;padding:10px 14px;margin:10px 0;background:var(--bg)}
.j p{margin:4px 0}
.j .a{white-space:pre-wrap}
.ta2{min-height:130px;font-size:.9rem}
@media(prefers-reduced-motion:reduce){*{animation:none!important}}
</style>
</head>
<body>
<div class="app">

<!-- START -->
<section id="start">
  <h1 class="title">Elsinore: <em>a Hamlet board game</em></h1>
  <p class="lede">Cross the castle, space by space. Spot the metaphors and similes, dig into motives, and set Hamlet beside his foils. Answer well to earn points before the final duel.</p>
  <div class="grid2">
    <div class="panel">
      <label class="f" for="pname">Your name</label>
      <input type="text" id="pname" maxlength="24" placeholder="Name or initials" autocomplete="off">
      <label class="f" id="tl">Choose your token</label>
      <div class="tokens" id="tokens" role="group" aria-labelledby="tl"></div>
      <label class="f" id="dl">Pace of the journey</label>
      <div class="dice" id="dice" role="group" aria-labelledby="dl"></div>
      <p style="margin:22px 0 0"><button class="btn primary" id="begin">Enter Elsinore</button></p>
      <p id="best" style="color:var(--muted);margin:12px 0 0"></p>
    </div>
    <div class="panel">
      <h3 style="margin-bottom:12px">What the spaces mean</h3>
      <ul class="legend">
        <li><div class="ic" style="border-color:var(--fig);color:var(--fig)">❝</div><div><b>Figurative language</b><span class="d">Multiple choice. Identify the device in a line from the play. 10 points.</span></div></li>
        <li><div class="ic" style="border-color:var(--deep);color:var(--deep)">🕯️</div><div><b>Deep dive</b><span class="d">Write a short answer about motive, theme or imagery. Up to 20 points.</span></div></li>
        <li><div class="ic" style="border-color:var(--foil);color:var(--foil)">⚔️</div><div><b>Foil face-off</b><span class="d">Compare two characters. Up to 20 points.</span></div></li>
        <li><div class="ic" style="border-color:var(--luck);color:var(--luck)">👻</div><div><b>Ghost's fortune</b><span class="d">A card that moves you or changes your score.</span></div></li>
        <li><div class="ic" style="border-color:var(--ink);color:var(--ink)">🗡️</div><div><b>The duel</b><span class="d">Reach the last space for one final written answer worth double.</span></div></li>
      </ul>
      <details>
        <summary>How short answers are graded</summary>
        <p>Each short answer is checked for key ideas, not exact wording. Use specific terms, and name characters, images and reasons. Answers need at least 12 words. Quoting the play in quotation marks earns a +3 evidence bonus. After each answer you see which ideas were found and a model response.</p>
      </details>
    </div>
  </div>
</section>

<!-- GAME -->
<section id="game" hidden>
  <div class="game">
    <div>
      <p class="scene" id="scene"></p>
      <div class="board" id="board" aria-label="Game board"></div>
    </div>
    <div class="side">
      <div class="panel">
        <div class="who"><span class="tk" id="htk"></span><span id="hname"></span></div>
        <div class="stats">
          <div class="stat" id="sScore"><b>0</b><span>Points</span></div>
          <div class="stat" id="sStreak"><b>0</b><span>Streak</span></div>
          <div class="stat" id="sSpace"><b>0</b><span>of 36</span></div>
        </div>
      </div>
      <div class="panel">
        <div class="rollrow">
          <div class="die" id="die" aria-hidden="true">?</div>
          <button class="btn primary" id="roll">Roll the die</button>
        </div>
        <p id="msg" aria-live="polite">Roll to leave the battlements.</p>
      </div>
    </div>
  </div>
</section>

<!-- END -->
<section id="end" hidden>
  <div class="panel">
    <p style="margin:0;color:var(--muted)" id="endwho"></p>
    <h2 class="rank" id="rank"></h2>
    <div class="big-score" id="bigscore"></div>
    <div id="endstats"></div>
    <h3 style="margin-top:18px">Your answer journal</h3>
    <div id="journal"></div>
    <h3 style="margin-top:18px">Results to share with your teacher</h3>
    <textarea class="ta2" id="resultText" readonly></textarea>
    <p class="cont" style="text-align:left"><button class="btn" id="copy">Copy results</button> <button class="btn primary" id="again">Play again</button> <span id="copied" style="color:var(--ghost)"></span></p>
  </div>
</section>

</div>

<div class="modal" id="modal" hidden role="dialog" aria-modal="true"><div class="sheet" id="sheet"></div></div>

<script>
/* ============== CONTENT ============== */
// Multiple choice: options[0] is always the correct answer (shuffled at runtime).
const MC = [
 {quote:"’Tis an unweeded garden\nThat grows to seed; things rank and gross in nature\nPossess it merely.", who:"Hamlet", loc:"Act 1, Scene 2", device:"Metaphor",
  options:["Metaphor","Simile","Pun","Apostrophe"],
  why:"Hamlet doesn’t say Denmark is like a garden. He says the world is one. The neglected, seeding garden pictures a kingdom overrun by moral weeds, and it starts the play’s pattern of decay imagery."},
 {quote:"Something is rotten in the state of Denmark.", who:"Marcellus", loc:"Act 1, Scene 4", device:"Metaphor",
  options:["Metaphor","Simile","Personification","Hyperbole"],
  why:"Corruption is described as physical rot. “State” also works two ways: the nation, and a condition. The whole kingdom is sick."},
 {quote:"Be thou as chaste as ice, as pure as snow, thou shalt not escape calumny.", who:"Hamlet to Ophelia", loc:"Act 3, Scene 1", device:"Simile",
  options:["Simile","Metaphor","Paradox","Allusion"],
  why:"The “as … as” structure makes this a simile. Hamlet says even perfect purity can’t protect Ophelia from slander, while he is pushing her away himself."},
 {quote:"And I, of ladies most deject and wretched …\nNow see that noble and most sovereign reason,\nLike sweet bells jangled, out of tune and harsh.", who:"Ophelia, about Hamlet", loc:"Act 3, Scene 1", device:"Simile",
  options:["Simile","Metaphor","Personification","Apostrophe"],
  why:"“Like” signals a simile. Hamlet’s once-harmonious mind is compared to bells ringing badly, and the image shows how Ophelia hears his apparent madness."},
 {quote:"Frailty, thy name is woman!", who:"Hamlet", loc:"Act 1, Scene 2", device:"Apostrophe",
  options:["Apostrophe","Simile","Pun","Oxymoron"],
  why:"Hamlet addresses an abstract idea, “Frailty,” as if it could hear him. He turns his disgust at Gertrude’s quick remarriage into an attack on all women, an unfair leap that shows how hurt he is."},
 {quote:"But look, the morn, in russet mantle clad,\nWalks o’er the dew of yon high eastward hill.", who:"Horatio", loc:"Act 1, Scene 1", device:"Personification",
  options:["Personification","Simile","Allusion","Paradox"],
  why:"The morning is given human traits: it wears a cloak and walks. Horatio’s image marks the end of the night watch and the ghost’s first appearance."},
 {quote:"I am too much i’ the sun.", who:"Hamlet", loc:"Act 1, Scene 2", device:"Pun",
  options:["Pun","Simile","Apostrophe","Hyperbole"],
  why:"“Sun” and “son” sound alike. Hamlet hints that he is tired of being called Claudius’s “son” and tired of the court’s glare."},
 {quote:"So excellent a king, that was to this\nHyperion to a satyr.", who:"Hamlet, about his father and Claudius", loc:"Act 1, Scene 2", device:"Allusion",
  options:["Allusion","Personification","Pun","Paradox"],
  why:"Hyperion was a Greek sun-titan; a satyr is a lustful half-goat. These classical references make old Hamlet godlike and Claudius beastly, a foil in a single line."},
 {quote:"I must be cruel only to be kind.", who:"Hamlet to Gertrude", loc:"Act 3, Scene 4", device:"Paradox",
  options:["Paradox","Simile","Apostrophe","Allusion"],
  why:"“Cruel” and “kind” seem to contradict each other, yet the line holds a truth. Hamlet must wound his mother’s conscience to try to heal her soul."},
 {quote:"My words fly up, my thoughts remain below:\nWords without thoughts never to heaven go.", who:"Claudius", loc:"Act 3, Scene 3", device:"Antithesis",
  options:["Antithesis","Simile","Personification","Allusion"],
  why:"Opposites are balanced: up and below, words and thoughts. Claudius admits his prayer is hollow. He wants the crown and the queen more than he wants forgiveness."},
 {quote:"I loved Ophelia: forty thousand brothers\nCould not, with all their quantity of love,\nMake up my sum.", who:"Hamlet", loc:"Act 5, Scene 1", device:"Hyperbole",
  options:["Hyperbole","Understatement","Simile","Pun"],
  why:"“Forty thousand brothers” is deliberate exaggeration. Hamlet is competing with Laertes’s grief, a loud public display that also shows how deeply he felt."},
 {quote:"The undiscovered country, from whose bourn\nNo traveller returns.", who:"Hamlet", loc:"Act 3, Scene 1", device:"Metaphor",
  options:["Metaphor","Simile","Personification","Hyperbole"],
  why:"Death is pictured as a foreign land with a one-way border. The metaphor drives Hamlet’s fear: no one can report back, so people endure the ills they know."},
 {quote:"The native hue of resolution\nIs sicklied o’er with the pale cast of thought.", who:"Hamlet", loc:"Act 3, Scene 1", device:"Metaphor",
  options:["Metaphor","Simile","Pun","Apostrophe"],
  why:"Resolution is imagined as a healthy complexion turned sickly. The image diagnoses Hamlet’s delay and echoes the play’s disease imagery."},
 {quote:"You would play upon me; you would seem to know my stops; you would pluck out the heart of my mystery … do you think I am easier to be played on than a pipe?", who:"Hamlet to Guildenstern", loc:"Act 3, Scene 2", device:"Extended metaphor",
  options:["Extended metaphor","Simile","Hyperbole","Apostrophe"],
  why:"Hamlet keeps one image going over several lines: himself as a musical pipe with stops and a hidden sound. He refuses to be an instrument others can play."},
 {quote:"The serpent that did sting thy father’s life\nNow wears his crown.", who:"The Ghost", loc:"Act 1, Scene 5", device:"Metaphor (with a biblical allusion)",
  options:["Metaphor (with a biblical allusion)","Simile","Hyperbole","Apostrophe"],
  why:"Claudius is called a serpent, a metaphor that recalls the serpent in Eden who destroyed a paradise. It is also ironic: Claudius spread the cover story that a snake killed the king."},
 {quote:"Whether ’tis nobler in the mind to suffer\nThe slings and arrows of outrageous fortune,\nOr to take arms against a sea of troubles …", who:"Hamlet", loc:"Act 3, Scene 1", device:"Mixed metaphor",
  options:["Mixed metaphor","Extended metaphor","Simile","Personification"],
  why:"Troubles shift from missiles to an ocean. You can’t fight a sea with weapons, so Hamlet’s own imagery hints that resisting suffering may be hopeless."},
 {quote:"Nay, but to live\nIn the rank sweat of an enseamed bed,\nStewed in corruption, honeying and making love\nOver the nasty sty!", who:"Hamlet to Gertrude", loc:"Act 3, Scene 4", device:"Sensory imagery",
  options:["Sensory imagery","Simile","Allusion","Apostrophe"],
  why:"Smell, taste, touch and sight pile up to make Gertrude’s remarriage feel physically repulsive. It extends the play’s imagery of rot and corruption."}
];

// Short answer: groups = key ideas. Each group matches if any stem appears at the start of a word.
const SA = [
 {cat:"D", title:"The unweeded garden", prompt:"Hamlet calls the world “an unweeded garden that grows to seed.” Explain what this metaphor shows about how he sees Denmark, and what the “weeds” might stand for.",
  groups:[
   {label:"Decay, rot or neglect in the kingdom", stems:["corrupt","rot","decay","sick","diseas","decline","neglect","overgrown","spoil","unkempt"]},
   {label:"Weeds linked to Claudius or moral wrongdoing", stems:["claudius","uncle","weed","sin","evil","murder","incest","wicked","usurp"]},
   {label:"Hamlet’s despair or disgust", stems:["disgust","despair","depress","grief","bleak","hopeless","sad","melanchol","weary","tired","revuls"]},
   {label:"Connection to Denmark, order or nature", stems:["denmark","kingdom","state","country","nature","order","unnatural","rule"]}],
  model:"The garden metaphor shows Hamlet seeing Denmark as neglected and decaying. Weeds that choke good growth suggest moral corruption, especially Claudius’s murder and his quick marriage to Gertrude. The image also reveals Hamlet’s deep disgust and depression: the world feels “weary, stale, flat, and unprofitable.”"},
 {cat:"O", title:"Hamlet and Fortinbras", prompt:"Fortinbras leads an army to fight over a worthless patch of land. How does he act as a foil to Hamlet, and what does Hamlet feel when he compares himself to him?",
  groups:[
   {label:"Fortinbras acts decisively (army, war, action)", stems:["act","decisive","army","fight","war","attack","invade","soldier","troops","bold"]},
   {label:"Hamlet’s delay or overthinking", stems:["delay","hesitat","think","overthink","procrastinat","indecis","paralyz","paralys","inaction","reflect"]},
   {label:"Honor or ambition over a trivial prize", stems:["honor","honour","patch","land","worthless","trivial","small","ambition","twenty thousand","egg"]},
   {label:"Hamlet feels shame or inspiration", stems:["shame","inspir","motiv","embarrass","guilt","ashamed","spur","challeng"]}],
  model:"Fortinbras is a man of action who risks thousands of lives over a tiny patch of land for honor, while Hamlet has a far greater cause and still hesitates. Seeing him makes Hamlet ashamed of his delay and, for a moment, inspired: “from this time forth, my thoughts be bloody.”"},
 {cat:"O", title:"Hamlet and Laertes", prompt:"Both Hamlet and Laertes lose a father and want revenge. How does Laertes act as a foil to Hamlet, and how does Claudius use the difference?",
  groups:[
   {label:"Laertes is impulsive or rash", stems:["impuls","rash","quick","hasty","immediate","rush","rage","angry","storm","violent","mob","rebel","hot"]},
   {label:"Hamlet is reflective or hesitant", stems:["reflect","delay","think","hesitat","cautious","deliberat","plan","doubt","careful"]},
   {label:"Both avenge a murdered father", stems:["father","avenge","revenge","kill","murder","death","dead","polonius"]},
   {label:"Claudius manipulates Laertes (poisoned sword, plot)", stems:["claudius","manipulat","poison","sword","tip","plot","used","pawn","scheme","trick","fenc","duel"]}],
  model:"Laertes storms back to Denmark ready to “cut his throat i’ the church,” acting on pure impulse, while Hamlet analyzes and delays. The contrast makes Hamlet’s hesitation clearer. Claudius exploits Laertes’s rage by steering him into a plot with a poisoned sword."},
 {cat:"O", title:"Hamlet and Horatio", prompt:"Hamlet tells Horatio, “Give me that man that is not passion’s slave.” Explain how Horatio is a foil to Hamlet and why Hamlet values him.",
  groups:[
   {label:"Horatio is rational, calm or steady", stems:["reason","calm","steady","stoic","level","rational","balanced","logical","self-control","control","composed","even"]},
   {label:"Hamlet is ruled by passion or emotion", stems:["passion","emotion","impuls","rage","grief","turbulent","anger","mad","erratic","volatil"]},
   {label:"Horatio is loyal and trustworthy", stems:["loyal","trust","friend","faithful","confidant","witness","true","honest","devot"]},
   {label:"Hamlet admires or wants to be like him", stems:["admire","praise","wish","want","envy","respect","slave","blessed","wants","pipe"]}],
  model:"Horatio is steady, reasonable and loyal, while Hamlet’s moods swing from rage to despair. Hamlet admires a friend who is not “passion’s slave,” the self-control Hamlet wishes he had. Horatio becomes the one person Hamlet trusts to witness the truth and tell his story."},
 {cat:"D", title:"The antic disposition", prompt:"Hamlet decides to put on “an antic disposition” (pretend madness). Give at least two reasons he might do this, using what you know about his situation.",
  groups:[
   {label:"To observe or gather information safely", stems:["observ","spy","watch","investigat","proof","evidence","test","gather","learn","discover"]},
   {label:"To protect himself or hide his plan", stems:["safe","protect","danger","survive","suspect","underestimate","hide","disguise","mask","cover","secre","plan"]},
   {label:"To speak or act freely without blame", stems:["free","speak","truth","say","insult","criticiz","licen","excuse","get away","blame"]},
   {label:"Genuine grief or instability blurs the act", stems:["grief","real","genuine","actual","sane","unstable","struggl","depress","trauma","blur"]}],
  model:"The act of madness lets Hamlet watch Claudius and the court while appearing harmless. It protects him from a king who might see him as a threat and gives him freedom to say sharp things people will excuse as madness. It may also blur with real grief and instability."},
 {cat:"D", title:"Why does Hamlet delay?", prompt:"Explain at least two reasons Hamlet doesn’t kill Claudius immediately after the Ghost’s command. Use details from the play.",
  groups:[
   {label:"Doubt about the Ghost (devil, need for proof)", stems:["ghost","devil","demon","spirit","doubt","trust","prove","proof","evidence","certain","sure","deceiv"]},
   {label:"Moral or religious conscience", stems:["moral","conscience","sin","religio","christian","soul","heaven","hell","afterlife","damn","god","sacred"]},
   {label:"Overthinking or fear", stems:["think","overthink","reflect","philosoph","analy","hesitat","paraly","reason","cowar","fear"]},
   {label:"The Mousetrap or the prayer scene", stems:["mousetrap","play","prayer","pray","kneel","confess","gonzago"]}],
  model:"Hamlet first doubts whether the Ghost is truly his father or a devil, so he stages The Mousetrap to get proof. Later he finds Claudius praying and decides not to kill him then, since he might go to heaven. Hamlet’s moral conscience and his habit of overthinking also hold him back."},
 {cat:"D", title:"Claudius’s motives", prompt:"What motivates Claudius to murder his brother and what does the “prayer scene” (3.3) reveal about him?",
  groups:[
   {label:"Ambition for the crown or power", stems:["ambition","crown","throne","power","king","kingship"]},
   {label:"Desire for Gertrude", stems:["gertrude","queen","wife","love","marry","marriage"]},
   {label:"Guilt or conscience", stems:["guilt","conscience","remorse","rank","pray","repent","sin","offence","offense"]},
   {label:"Cunning or deceit", stems:["cunning","manipulat","politic","scheme","plot","hide","mask","spy","smile","charm","deceiv","calculat"]}],
  model:"Claudius kills his brother for the crown and the queen, the “things for which I did the murder.” The prayer scene shows that he is not a simple villain: he feels deep guilt and knows he cannot repent while still holding what he stole. He is also cunning, hiding his crime behind political skill."},
 {cat:"O", title:"Hyperion and the satyr", prompt:"Hamlet compares his father to Hyperion and Claudius to a satyr. How are old King Hamlet and Claudius foils, and how does the contrast fuel Hamlet’s anger?",
  groups:[
   {label:"King Hamlet: noble, godlike or heroic", stems:["hyperion","sun","god","noble","majestic","warrior","heroic","brave","honorable","great","ideal"]},
   {label:"Claudius: lustful, beastly or base", stems:["satyr","lust","beast","goat","unworthy","base","lecherous","brutish","animal","crude","lower","low"]},
   {label:"The murder or usurpation", stems:["murder","poison","ear","killed","kill","fratricide","usurp","stole","brother"]},
   {label:"Direct contrast or comparison", stems:["contrast","compar","differ","unlike","whereas","opposite","inferior","superior","while","foil"]}],
  model:"Hamlet idealizes his father as a godlike king (Hyperion) and sees Claudius as a lustful beast (satyr). The sharp contrast intensifies his grief and anger, especially since Claudius poisoned his brother, seized the throne and married Gertrude."},
 {cat:"D", title:"Polonius at court", prompt:"Describe Polonius’s role in the play. What does he want, and how do his actions affect his own children and Hamlet?",
  groups:[
   {label:"He spies or eavesdrops", stems:["spy","spies","eavesdrop","watch","listen","hide","arras","reynaldo"]},
   {label:"He gives advice or controls his children", stems:["advice","rule","proverb","precept","lecture","obey","control","command","order"]},
   {label:"He seeks status or the king’s favor", stems:["status","ambition","favor","favour","court","power","position","reputation","servile","flatter","king"]},
   {label:"Hypocrisy, irony or meddling", stems:["hypocri","contradict","irony","ironic","fool","windbag","verbose","wordy","pompous","meddl","busybody"]}],
  model:"Polonius is a meddling royal adviser who spies on his own son and uses Ophelia to explain Hamlet’s behavior. He controls his children, gives advice, and seeks the king’s favor. His spying on Hamlet in Gertrude’s room costs him his life, which in turn sends Ophelia and Laertes into tragedy."},
 {cat:"D", title:"Ophelia’s breakdown", prompt:"How do the men in Ophelia’s life (Polonius, Laertes, Hamlet) contribute to her breakdown? Support your answer with specific events.",
  groups:[
   {label:"Control by father and brother", stems:["obey","obedien","father","brother","control","command","dutiful","compliant","manipulat","pawn","used","orders"]},
   {label:"Polonius’s death", stems:["polonius","death","killed","murder","loss","dead","slain"]},
   {label:"Hamlet’s rejection (e.g., “nunnery”)", stems:["hamlet","nunnery","reject","cruel","harsh","abandon","betray","insult","love"]},
   {label:"Madness (songs, flowers, drowning)", stems:["madness","mad","insane","flowers","songs","sing","break","collapse","unravel","drown","grief"]}],
  model:"Ophelia is told by Polonius and Laertes whom to love and how to behave, then used as bait to test Hamlet. Hamlet rejects her cruelly (“Get thee to a nunnery”) and kills her father. With no one left to protect her, her grief becomes madness. She sings, hands out flowers, and eventually drowns."},
 {cat:"D", title:"Gertrude’s choices", prompt:"Explain Gertrude’s place in the play. How does Hamlet see her, and what do her final actions show?",
  groups:[
   {label:"Hasty remarriage criticized", stems:["hasty","quick","soon","remarr","marriage","married","weeks","months","incest"]},
   {label:"Her love or concern for Hamlet", stems:["love","son","protect","mother","concern","care","hamlet"]},
   {label:"Guilt or inner conflict (closet scene)", stems:["guilt","closet","mirror","conscience","soul","shame","torn","conflict","cleft","heart"]},
   {label:"Poisoned cup and death", stems:["poison","cup","drink","death","die","toast","goblet","chalice","unknowing","innocent"]}],
  model:"Hamlet resents Gertrude for marrying Claudius so soon after his father’s death. In the closet scene he forces her to face her conscience, and she admits her heart is “cleft in twain.” At the end she drinks from the poisoned cup, perhaps in love for her son, and dies because of Claudius’s plot."},
 {cat:"D", title:"Rosencrantz and Guildenstern", prompt:"Why do Rosencrantz and Guildenstern agree to spy on Hamlet, and what does Hamlet’s treatment of them show about friendship and betrayal?",
  groups:[
   {label:"Loyalty or duty to the king", stems:["loyal","king","claudius","serve","duty","obey"]},
   {label:"Self-interest or reward", stems:["reward","gain","ambition","payment","favor","favour","self-interest","selfish","benefit","profit","bribe"]},
   {label:"Betrayal of boyhood friendship", stems:["friend","betray","childhood","school","boyhood","trust","deceive","false","spy"]},
   {label:"Their fate (England, execution)", stems:["fate","england","execute","death","dead","letter","sent","kill","die"]}],
  model:"Rosencrantz and Guildenstern obey Claudius and hope to gain royal favor, even if it means spying on their old friend. Hamlet quickly sees through them (“you would play upon me”) and sends them to their deaths in England. The episode shows how friendship is corrupted by Claudius’s court."},
 {cat:"D", title:"Yorick’s skull", prompt:"In the graveyard, Hamlet holds Yorick’s skull and thinks about Alexander and Caesar. What does this scene reveal about death and about how Hamlet’s thinking has changed?",
  groups:[
   {label:"Death is inevitable", stems:["death","mortal","dead","die","skull","inevitable","end","decay"]},
   {label:"Death makes everyone equal", stems:["equal","everyone","beggar","alexander","caesar","rich","poor","all","level","same","vanity","status","power"]},
   {label:"Hamlet’s acceptance or growth", stems:["accept","calm","change","matur","ready","resolved","readiness","fate","providence","peace"]},
   {label:"Contrast of laughter and decay", stems:["laughter","jest","jester","joy","humor","humour","childhood","memory","ironic","irony","contrast"]}],
  model:"Holding a skull that once belonged to a joking friend reminds Hamlet that death comes for everyone, from kings to beggars. His thoughts shift from fear and anger toward calm acceptance, and soon after he says “the readiness is all.”"},
 {cat:"D", title:"Playing the pipe", prompt:"Hamlet tells Guildenstern, “You would play upon me … do you think I am easier to be played on than a pipe?” Explain the extended metaphor and what it says about Hamlet.",
  groups:[
   {label:"Names the recorder, pipe or instrument", stems:["recorder","pipe","instrument","flute","play","music"]},
   {label:"Being manipulated or used", stems:["manipulat","control","use","used","pawn","tool","exploit","puppet","spy"]},
   {label:"His inner “mystery” or secrets", stems:["mystery","secret","heart","private","understand","inner","stops","sound"]},
   {label:"He refuses or resists", stems:["refus","resist","independen","identity","self","agency","pride","dignity","defiant","autonom","will not"]}],
  model:"Hamlet extends the image of a musical pipe: Guildenstern thinks he can learn Hamlet’s “stops” and play him like an instrument to draw out his secrets for Claudius. Hamlet says he is no one’s pipe, which shows his refusal to be manipulated and his pride in his inner self."},
 {cat:"D", title:"The mousetrap", prompt:"Hamlet says, “The play’s the thing / Wherein I’ll catch the conscience of the king.” Explain the metaphor and Hamlet’s plan.",
  groups:[
   {label:"Trap or catching imagery", stems:["trap","catch","snare","hunt","animal","mouse"]},
   {label:"Claudius’s guilty reaction", stems:["conscience","guilt","reaction","react","confess","reveal","expose","flinch","prove","test"]},
   {label:"A play that re-enacts the murder", stems:["play","theater","theatre","actor","murder","gonzago","performance","reenact","re-enact","stage","drama"]},
   {label:"Verifying the Ghost’s story", stems:["ghost","truth","verify","confirm","certain","doubt","sure"]}],
  model:"Hamlet will “catch” Claudius’s conscience as if trapping an animal. By staging a play that re-enacts the murder, he hopes Claudius’s reaction will reveal his guilt, which will confirm that the Ghost told the truth."},
 {cat:"D", title:"Patterns of poison and disease", prompt:"Throughout the play, images of poison, disease and rot recur. Choose some examples and explain what they suggest about Denmark and where the “infection” begins.",
  groups:[
   {label:"Names images of poison, disease or rot", stems:["poison","ear","rot","rotten","disease","sick","infect","canker","ulcer","plague","decay","corrupt","contaminat"]},
   {label:"Applies the images to the whole state", stems:["denmark","state","kingdom","court","country","whole","everyone","society","spread","political"]},
   {label:"Traces them to Claudius’s crime", stems:["claudius","murder","crime","sin","guilt","secret","hidden","brother","fratricide","usurp"]},
   {label:"Shows the spread or results (deaths)", stems:["tragedy","deaths","end","bodies","cup","fenc","laertes","gertrude","spread","kills","dies"]}],
  model:"Poison in the ear, an unweeded garden and “something rotten” all picture Denmark as sick. The infection starts with Claudius’s secret crime and spreads through the court until the final scene, where a poisoned sword and cup kill nearly everyone."},
 {cat:"O", title:"Ophelia and Gertrude", prompt:"Compare Ophelia and Gertrude as women in a court ruled by men. How are they alike and different, and what happens to each?",
  groups:[
   {label:"Both depend on or are controlled by men", stems:["father","husband","control","obey","men","patriarch","power","dependent","powerless","brother","king"]},
   {label:"Ophelia: young, innocent, obedient", stems:["innocent","pure","obedient","young","chaste","naive","naïve","obey"]},
   {label:"Gertrude: older, pragmatic, queen", stems:["remarr","pragmat","surviv","queen","status","strong-willed","political","older","mother","sensual"]},
   {label:"Both end tragically (death, madness, poison)", stems:["tragic","death","die","madness","drown","poison","victim","consequence","both"]}],
  model:"Both women are shaped by powerful men. Ophelia is young and obedient and is controlled by her father and brother, while Gertrude is an older queen who has survived by remarrying. Each is hurt by the men’s schemes, and both die: Ophelia drowns after madness, and Gertrude drinks poison meant for Hamlet."},
 {cat:"O", title:"Hamlet and Claudius as mirrors", prompt:"Hamlet and Claudius are enemies, but they also resemble each other. Explain how they are similar and how they differ.",
  groups:[
   {label:"Both kill (or plot to kill) relatives or rivals", stems:["revenge","murder","kill","polonius","fratricide","both","similar","plot"]},
   {label:"Both wrestle with guilt or conscience", stems:["guilt","conscience","soliloqu","pray","remorse","sin"]},
   {label:"Both deceive or perform", stems:["deceit","deceive","act","pretend","mask","disguise","feign","spy","antic","politic","lie"]},
   {label:"Differences in motive or morality", stems:["differ","unlike","whereas","intent","motive","justice","ambition","selfish","duty","moral"]}],
  model:"Both men kill, plot and pretend: Claudius hides murder behind a polite face and Hamlet hides his plan behind madness. Both speak private thoughts of guilt or doubt. But their motives differ: Claudius kills for crown and queen, while Hamlet is driven by duty, grief and a search for justice."}
];

const LAYOUT = "SFDFLO"+"FDLFOD"+"FLDFOF"+"DFLODF"+"LDFOFD"+"FOLDFE";
const ICON = {S:"🏰",F:"❝",D:"🕯️",O:"⚔️",L:"👻",E:"🗡️"};
const ROWS = ["Act I: The battlements","Act II: The antic disposition","Act III: The mousetrap","Act IV: Madness and exile","Act V: The graveyard","The duel"];
const ROWSUB = ["Ghosts, guards and a grieving prince","Hamlet puts on his mask","The play within the play","Plots, poison and a drowning","Skulls, swords and a final toast","One last answer"];
const LUCK = [
 {t:"The Ghost beckons: “Remember me.” Advance 2 spaces.", icon:"👻", move:2},
 {t:"Ophelia hands you rosemary, “that’s for remembrance.” +5 points.", icon:"🌹", pts:5},
 {t:"A poisoned cup! You lose 5 points.", icon:"🍷", pts:-5},
 {t:"Fortinbras’s army marches through Denmark. Advance 3 spaces.", icon:"⚔️", move:3},
 {t:"Polonius lectures you at great length. Retreat 2 spaces.", icon:"📜", move:-2},
 {t:"Horatio lends you his notes. +8 points.", icon:"🕯️", pts:8},
 {t:"Yorick’s skull makes you smile. +3 points.", icon:"💀", pts:3},
 {t:"A spy behind the arras overhears you. Lose 3 points.", icon:"👂", pts:-3},
 {t:"The gravedigger’s riddle: you solve it. +4 points.", icon:"⛏️", pts:4}
];
const TOKENS = ["🤴","💀","🌹","🗡️","👻","🕯️"];
const DICE = [{s:6,name:"Quick journey",d:"Roll 1 to 6. About 8 questions."},{s:4,name:"Standard",d:"Roll 1 to 4. About 12 questions."},{s:3,name:"Scholar’s pace",d:"Roll 1 to 3. About 16 questions."}];

/* ============== HELPERS ============== */
const $ = id => document.getElementById(id);
const sleep = ms => new Promise(r => setTimeout(r, ms));
const shuffle = a => { a = a.slice(); for(let i=a.length-1;i>0;i--){const j=Math.floor(Math.random()*(i+1));[a[i],a[j]]=[a[j],a[i]];} return a; };
const html = s => { const d=document.createElement('div'); d.innerHTML=s.trim(); return d.firstElementChild; };
const esc = s => { const d=document.createElement('div'); d.textContent=s; return d.innerHTML; };
function storeGet(k){ try{ return localStorage.getItem(k); }catch(e){ return null; } }
function storeSet(k,v){ try{ localStorage.setItem(k,v); }catch(e){} }

/* ============== STATE ============== */
let S = null;
let sel = {token:TOKENS[0], dice:1};

function newState(){
  return {
    name:$('pname').value.trim() || "Student", token:sel.token, sides:DICE[sel.dice].s,
    pos:0, score:0, streak:0, rolls:0, busy:false,
    decks:{F:shuffle(MC.map((_,i)=>i)), D:shuffle(SA.map((q,i)=>q.cat==='D'?i:-1).filter(i=>i>=0)), O:shuffle(SA.map((q,i)=>q.cat==='O'?i:-1).filter(i=>i>=0))},
    luck:shuffle(LUCK.map((_,i)=>i)),
    log:[], earned:0, possible:0
  };
}
function draw(type){
  if(!S.decks[type].length){
    S.decks[type] = type==='F' ? shuffle(MC.map((_,i)=>i)) : shuffle(SA.map((q,i)=>q.cat===type?i:-1).filter(i=>i>=0));
  }
  return S.decks[type].pop();
}

/* ============== START SCREEN ============== */
function initStart(){
  const tk = $('tokens');
  TOKENS.forEach(t => {
    const b = document.createElement('button'); b.className='tok'; b.type='button'; b.textContent=t;
    b.setAttribute('aria-label','Token '+t); b.setAttribute('aria-pressed', t===sel.token);
    b.onclick = () => { sel.token=t; [...tk.children].forEach(c=>c.setAttribute('aria-pressed', c===b)); };
    tk.appendChild(b);
  });
  const dc = $('dice');
  DICE.forEach((d,i) => {
    const b = document.createElement('button'); b.className='dopt'; b.type='button';
    b.innerHTML = `<b>${d.name}</b><small>${d.d}</small>`;
    b.setAttribute('aria-pressed', i===sel.dice);
    b.onclick = () => { sel.dice=i; [...dc.children].forEach(c=>c.setAttribute('aria-pressed', c===b)); };
    dc.appendChild(b);
  });
  const best = storeGet('elsinore_best');
  if(best) $('best').textContent = "Your best score on this device: " + best;
  $('begin').onclick = startGame;
}

function startGame(){
  S = newState();
  $('start').hidden = true; $('end').hidden = true; $('game').hidden = false;
  $('htk').textContent = S.token; $('hname').textContent = S.name;
  $('roll').textContent = "Roll the die (1–" + S.sides + ")";
  $('die').textContent = "?";
  msg("Roll to leave the battlements.");
  renderBoard(); updateHUD();
  window.scrollTo(0,0);
}

/* ============== BOARD ============== */
function renderBoard(){
  const b = $('board'); b.innerHTML = '';
  for(let dr=0; dr<6; dr++){
    const lr = 5-dr;
    for(let c=0;c<6;c++){
      const i = lr*6 + (lr%2===0 ? c : 5-c);
      const t = LAYOUT[i];
      const cell = document.createElement('div');
      cell.className = 'cell '+t+(i<S.pos?' past':'')+(i===S.pos?' here':'');
      cell.innerHTML = `<span class="n">${i===0?'':i+1}</span><span class="ic">${ICON[t]}</span>`;
      if(i===S.pos){ const k=document.createElement('span'); k.className='token'; k.textContent=S.token; cell.appendChild(k); }
      b.appendChild(cell);
    }
  }
  const row = Math.floor(S.pos/6);
  $('scene').innerHTML = `${ROWS[row]}<small>${ROWSUB[row]}</small>`;
}
function updateHUD(popId){
  $('sScore').firstElementChild.textContent = S.score;
  $('sStreak').firstElementChild.textContent = S.streak;
  $('sSpace').firstElementChild.textContent = S.pos+1;
  if(popId){ const e=$(popId); e.classList.remove('pop'); void e.offsetWidth; e.classList.add('pop'); }
}
function msg(t){ $('msg').textContent = t; }
function addPoints(n){
  S.score = Math.max(0, S.score + n);
  updateHUD('sScore');
}

/* ============== TURN FLOW ============== */
$('roll').onclick = takeTurn;
async function takeTurn(){
  if(S.busy) return;
  S.busy = true; $('roll').disabled = true;
  const die = $('die'); die.classList.add('rolling');
  for(let i=0;i<7;i++){ die.textContent = 1+Math.floor(Math.random()*S.sides); await sleep(90); }
  const r = 1+Math.floor(Math.random()*S.sides);
  die.classList.remove('rolling'); die.textContent = r;
  S.rolls++;
  msg("You rolled a "+r+".");
  await moveBy(r);
  const finished = await resolveSpace();
  if(!finished){ S.busy=false; $('roll').disabled=false; }
}
async function moveBy(n){
  const dir = n>0?1:-1;
  for(let i=0;i<Math.abs(n);i++){
    const next = S.pos+dir;
    if(next<0 || next>35) break;
    S.pos = next; renderBoard(); updateHUD(); await sleep(230);
  }
}
async function resolveSpace(){
  const t = LAYOUT[S.pos];
  if(t==='F'){ await playMC(); }
  else if(t==='D' || t==='O'){ await playSA(t,1,false); }
  else if(t==='L'){ await playLuck(); if(S.pos===35) return await resolveSpace(); const t2=LAYOUT[S.pos]; if(t2==='F'||t2==='D'||t2==='O'){ return await resolveSpace(); } }
  else if(t==='E'){ msg("The final duel! One last answer, worth double."); await sleep(500); await playSA(Math.random()<.5?'D':'O',2,true); endGame(); return true; }
  else { msg("Back at the battlements."); }
  return false;
}

/* ============== MODAL ============== */
function showModal(node){ $('sheet').innerHTML='';$('sheet').appendChild(node); $('modal').hidden=false; $('modal').scrollTop=0; }
function hideModal(){ $('modal').hidden=true; $('sheet').innerHTML=''; }

function streakCheck(full){
  if(full){
    S.streak++;
    if(S.streak%3===0){ addPoints(5); return " Streak bonus: +5!"; }
  } else S.streak = 0;
  updateHUD('sStreak');
  return "";
}

/* ---- multiple choice ---- */
function playMC(){
  return new Promise(res => {
    const q = MC[draw('F')];
    const opts = shuffle(q.options);
    const w = html(`<div>
      <span class="chip F">❝ Figurative language</span>
      <p class="attr">${esc(q.who)}, ${esc(q.loc)}</p>
      <blockquote>${esc(q.quote)}</blockquote>
      <p><b>Which figurative language device is at work here?</b></p>
      <div class="opts"></div><div class="fbwrap"></div></div>`);
    const box = w.querySelector('.opts');
    opts.forEach(o => {
      const b = document.createElement('button'); b.className='opt'; b.textContent=o;
      b.onclick = () => {
        const ok = o===q.options[0];
        [...box.children].forEach(c => { c.disabled=true; if(c.textContent===q.options[0]) c.classList.add('right'); });
        if(!ok) b.classList.add('wrong');
        S.possible += 10; let earned = 0;
        if(ok){ earned=10; S.earned+=10; addPoints(10); }
        const bonus = streakCheck(ok);
        S.log.push({kind:"Figurative language", title:q.device, prompt:q.quote.replace(/\n/g,' / '), answer:o, result:ok?"Correct":"Missed (answer: "+q.options[0]+")", pts:earned, device:q.device, ok});
        const fb = html(`<div class="fb"><h3>${ok?"Well spotted.":"Not quite."}</h3>
          <p class="pts ${ok?'':'zero'}">${ok?"+10 points."+bonus:"The answer is "+esc(q.options[0])+"."}</p>
          <p>${esc(q.why)}</p></div>`);
        const c = html(`<div class="cont"><button class="btn primary">Continue</button></div>`);
        c.firstElementChild.onclick = () => { hideModal(); msg(ok?"Correct! Roll again.":"Learn from it and roll again."); res(); };
        w.querySelector('.fbwrap').append(fb, c);
        c.firstElementChild.focus();
      };
      box.appendChild(b);
    });
    showModal(w);
  });
}

/* ---- short answer ---- */
function norm(s){ return " "+s.toLowerCase().replace(/[’‘]/g,"'").replace(/[^a-z0-9'\- ]/g,' ').replace(/\s+/g,' '); }
function stemHit(text, stem){
  const re = new RegExp("(^|[^a-z0-9])"+stem.replace(/[.*+?^${}()|[\]\\]/g,'\\$&'));
  return re.test(text);
}
function gradeSA(q, raw){
  const text = norm(raw);
  const hits = q.groups.map(g => g.stems.some(s => stemHit(text,s)));
  const n = hits.filter(Boolean).length;
  const need = 3;
  const base = n>=need ? 20 : n===need-1 ? 12 : n>=1 ? 5 : 0;
  const evidence = /["“”][^"“”]{8,}["“”]/.test(raw);
  return {hits, n, base, evidence};
}
function wordCount(s){ return (s.trim().match(/\S+/g)||[]).length; }

function playSA(cat, mult, isDuel){
  return new Promise(res => {
    const q = SA[draw(cat)];
    const label = isDuel ? "🗡️ The duel (double points)" : (cat==='D' ? "🕯️ Deep dive" : "⚔️ Foil face-off");
    const w = html(`<div>
      <span class="chip ${isDuel?'E':cat}">${label}</span>
      <h2 style="margin-bottom:8px">${esc(q.title)}</h2>
      <p>${esc(q.prompt)}</p>
      <textarea id="ans" placeholder="Write 2 to 4 sentences. Be specific: name characters, images and reasons."></textarea>
      <div class="wc"><span id="wcount">0 words (12 needed)</span><button class="btn primary" id="sub" disabled>Submit answer</button></div>
      <div class="fbwrap"></div></div>`);
    const ta = w.querySelector('#ans'), sub = w.querySelector('#sub'), wc = w.querySelector('#wcount');
    ta.oninput = () => { const n=wordCount(ta.value); wc.textContent = n+" words"+(n<12?" (12 needed)":""); sub.disabled = n<12; };
    sub.onclick = () => {
      const raw = ta.value; const g = gradeSA(q, raw);
      ta.disabled = true; sub.disabled = true; sub.hidden = true;
      let pts = (g.base + (g.evidence?3:0)) * mult;
      const full = g.base===20;
      S.possible += 20*mult; S.earned += Math.min(g.base*mult, 20*mult);
      addPoints(pts);
      const bonus = streakCheck(full);
      const verdict = full ? "Excellent." : g.base===12 ? "Solid start." : g.base===5 ? "A beginning." : "Try again next time.";
      S.log.push({kind:isDuel?"The duel":(cat==='D'?"Deep dive":"Foil face-off"), title:q.title, prompt:q.prompt, answer:raw, model:q.model, result:`${g.n} of ${q.groups.length} key ideas`, pts, ok:full});
      const list = q.groups.map((gr,i) => `<li class="${g.hits[i]?'y':'n'}"><span class="mk">${g.hits[i]?'✓':'○'}</span><span>${esc(gr.label)}${g.hits[i]?'':' <em style="color:var(--muted)">(not found)</em>'}</span></li>`).join('');
      const fb = html(`<div class="fb"><h3>${verdict}</h3>
        <p class="pts ${pts?'':'zero'}">+${pts} points${mult>1?' (doubled)':''}${g.evidence?' · includes a +'+(3*mult)+' evidence bonus':''}.${bonus}</p>
        <p style="margin:6px 0 0">You covered ${g.n} of ${q.groups.length} key ideas:</p>
        <ul class="chk">${list}</ul>
        <div class="model"><b>Model response</b>${esc(q.model)}</div></div>`);
      const c = html(`<div class="cont"><button class="btn primary">${isDuel?'See my results':'Continue'}</button></div>`);
      c.firstElementChild.onclick = () => { hideModal(); msg(full?"Great work. Roll again.":"Compare your answer with the model, then roll again."); res(); };
      w.querySelector('.fbwrap').append(fb, c);
      c.firstElementChild.focus();
    };
    showModal(w);
    setTimeout(()=>ta.focus(), 50);
  });
}

/* ---- luck ---- */
function playLuck(){
  return new Promise(res => {
    if(!S.luck.length) S.luck = shuffle(LUCK.map((_,i)=>i));
    const c = LUCK[S.luck.pop()];
    const w = html(`<div class="luck"><span class="chip D" style="color:var(--luck)">👻 Ghost’s fortune</span>
      <div class="big">${c.icon}</div><p style="font-size:1.15rem">${esc(c.t)}</p>
      <div class="cont" style="text-align:center"><button class="btn primary">Continue</button></div></div>`);
    w.querySelector('button').onclick = async () => {
      hideModal();
      if(c.pts) addPoints(c.pts);
      if(c.move){ await moveBy(c.move); }
      res();
    };
    showModal(w);
    w.querySelector('button').focus();
  });
}

/* ============== END ============== */
function endGame(){
  S.busy = true;
  const pct = S.possible ? Math.round(100*S.earned/S.possible) : 0;
  const rank = pct>=85 ? "Prince of Denmark" : pct>=65 ? "Scholar of Wittenberg" : pct>=40 ? "Courtier of Elsinore" : "Gravedigger’s Apprentice";
  $('game').hidden = true; $('end').hidden = false;
  $('endwho').textContent = S.token+" "+S.name+" reached the duel in "+S.rolls+" rolls.";
  $('rank').textContent = rank;
  $('bigscore').textContent = S.score+" points";
  const prevBest = parseInt(storeGet('elsinore_best')||"0",10);
  if(S.score>prevBest) storeSet('elsinore_best', String(S.score));

  const mcs = S.log.filter(l=>l.device), sas = S.log.filter(l=>!l.device);
  const mcOk = mcs.filter(l=>l.ok).length;
  const byDev = {}; mcs.forEach(l=>{ byDev[l.device]=byDev[l.device]||{ok:0,n:0}; byDev[l.device].n++; if(l.ok) byDev[l.device].ok++; });
  const devRows = Object.keys(byDev).map(d=>`<tr><td>${esc(d)}</td><td>${byDev[d].ok} of ${byDev[d].n}</td></tr>`).join('') || `<tr><td colspan="2">No multiple-choice questions this game.</td></tr>`;
  const saFull = sas.filter(l=>l.ok).length;
  $('endstats').innerHTML = `<p>Accuracy on questions asked: <b>${pct}%</b>. Figurative language: <b>${mcOk} of ${mcs.length}</b> correct. Written answers with full key ideas: <b>${saFull} of ${sas.length}</b>.</p>
    <table><thead><tr><th>Device</th><th>Correct</th></tr></thead><tbody>${devRows}</tbody></table>`;

  const j = $('journal'); j.innerHTML='';
  S.log.forEach(l => {
    const d = document.createElement('div'); d.className='j';
    let inner = `<p><b>${esc(l.kind)}: ${esc(l.title)}</b> · ${esc(l.result)} · ${l.pts} pts</p>`;
    if(l.device){ inner += `<p class="a" style="color:var(--muted)">${esc(l.prompt)}</p><p>Your answer: ${esc(l.answer)}</p>`; }
    else { inner += `<p class="a">${esc(l.answer)}</p><p style="color:var(--muted)"><b>Model:</b> ${esc(l.model)}</p>`; }
    d.innerHTML = inner; j.appendChild(d);
  });

  let t = `Hamlet: Elsinore board game\nStudent: ${S.name}\nScore: ${S.score} (${rank}); accuracy ${pct}% on questions asked\nRolls: ${S.rolls}\n\nFigurative language: ${mcOk}/${mcs.length} correct\n`;
  Object.keys(byDev).forEach(d => t += `  - ${d}: ${byDev[d].ok}/${byDev[d].n}\n`);
  t += `\nWritten answers (${sas.length}):\n`;
  sas.forEach((l,i) => t += `\n${i+1}. ${l.title} [${l.result}, ${l.pts} pts]\n   ${l.answer.replace(/\n/g,' ')}\n`);
  $('resultText').value = t;
  window.scrollTo(0,0);
}
$('copy').onclick = async () => {
  const ta = $('resultText');
  try{ await navigator.clipboard.writeText(ta.value); $('copied').textContent="Copied."; }
  catch(e){ ta.focus(); ta.select(); $('copied').textContent="Press Ctrl+C (or Cmd+C) to copy the selected text."; }
};
$('again').onclick = () => {$('end').hidden=true; $('start').hidden=false; const best=storeGet('elsinore_best'); if(best)$('best').textContent="Your best score on this device: "+best; window.scrollTo(0,0); };

initStart();
</script>
</body>
</html>

```
