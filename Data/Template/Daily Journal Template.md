#### properties
other_liquid:: 0
hydration:: 0
water:: 0
mood:: 0
sleep:: 8
####
```dataviewjs
const DAILY_FOLDER = "4 Archives/Journal/Daily Notes";
const DAILY_FORMAT = "YYYY-MM-DD";

function styleBar(el) {
  el.style.display = "flex";
  el.style.gap = "10px";
  el.style.alignItems = "center";
  el.style.flexWrap = "wrap";
}

function styleBtn(el) {
  el.style.padding = "8px 10px";
  el.style.borderRadius = "8px";
  el.style.background = "rgba(0,0,0,0.10)";
  el.style.border = "1px solid rgba(0,0,0,0.20)";
  el.style.cursor = "pointer";
  el.style.userSelect = "none";
  el.style.fontWeight = "700";
}

function getBaseDate() {
  // If you're currently viewing a daily note like:
  // 4 Archives/Journal/Daily Notes/2026-03-25.md
  // then use that date as the base.
  const f = app.workspace.getActiveFile();
  if (f) {
    const inFolder = f.path.startsWith(DAILY_FOLDER + "/");
    const baseName = f.basename; // without .md
    const m = moment(baseName, DAILY_FORMAT, true);
    if (inFolder && m.isValid()) return m.startOf("day");
  }
  return moment().startOf("day");
}

async function openDaily(dateMoment) {
  const filename = dateMoment.format(DAILY_FORMAT) + ".md";
  const path = `${DAILY_FOLDER}/${filename}`;

  let file = app.vault.getAbstractFileByPath(path);

  // Only create if missing
  if (!file) file = await app.vault.create(path, "");

  await app.workspace.getLeaf(false).openFile(file);
}

const container = this.container;
container.innerHTML = "";

const base = getBaseDate();

const bar = document.createElement("div");
styleBar(bar);

const prevBtn = document.createElement("div");
prevBtn.textContent = "← Prev day";
styleBtn(prevBtn);

const todayBtn = document.createElement("div");
todayBtn.textContent = "Today";
styleBtn(todayBtn);

const nextBtn = document.createElement("div");
nextBtn.textContent = "Next day →";
styleBtn(nextBtn);

const label = document.createElement("div");
label.style.fontWeight = "900";
label.style.opacity = "0.85";
label.textContent = base.format("YYYY-MM-DD");

bar.appendChild(prevBtn);
bar.appendChild(todayBtn);
bar.appendChild(nextBtn);
bar.appendChild(label);

container.appendChild(bar);

prevBtn.onclick = () => openDaily(moment(base).subtract(1, "day"));
todayBtn.onclick = () => openDaily(moment().startOf("day"));
nextBtn.onclick = () => openDaily(moment(base).add(1, "day"));
```

```dataviewjs
// === Mood Buttons for THIS note ===

// Mood options
const moods = [
    { value: 1, emoji: "😡" },
    { value: 2, emoji: "😕" },
    { value: 3, emoji: "😐" },
    { value: 4, emoji: "🙂" },
    { value: 5, emoji: "🤩" }
];

// === UI container ===
const container = this.container;
container.style.display = "flex";
container.style.gap = "12px";
container.style.marginBottom = "14px";

// === Button styling ===
function styleButton(btn) {
    btn.style.fontSize = "26px";          // bigger emoji
    btn.style.cursor = "pointer";
    btn.style.width = "48px";             // perfect square
    btn.style.height = "48px";
    btn.style.display = "flex";           // center emoji
    btn.style.alignItems = "center";
    btn.style.justifyContent = "center";
    btn.style.borderRadius = "10px";      // softer corners
    btn.style.background = "#1f1f1f";
    btn.style.border = "1px solid #00000040";
    btn.style.transition = "0.15s";
    btn.style.userSelect = "none";

    btn.onmouseover = () => btn.style.background = "#2a2a2a";
    btn.onmouseout  = () => btn.style.background = "#1f1f1f";
}

// === Write mood to THIS file ===
async function setMood(value) {
    const file = app.workspace.getActiveFile();
    if (!file) return;

    let content = await app.vault.read(file);

    // Replace existing mood:: X
    if (content.match(/mood::\s*\d/)) {
        content = content.replace(/mood::\s*\d/, `mood:: ${value}`);
    } else {
        // Insert at top
        content = `mood:: ${value}\n` + content;
    }

    await app.vault.modify(file, content);
    new Notice(`Mood set to ${value}`);
}

// === Render buttons ===
moods.forEach(m => {
    const btn = document.createElement("div");
    btn.textContent = m.emoji;
    styleButton(btn);
    btn.onclick = () => setMood(m.value);
    container.appendChild(btn);
});
```
```dataviewjs
// Sleep (minimal) — quick buttons + custom input (press Enter), writes inline `sleep::` to CURRENT note
const f=app.workspace.getActiveFile(); if(!f) return;
const K="sleep", MIN=0, MAX=15, PRE=[4,5,6,7,7.5,8,9,10];
const esc=s=>s.replace(/[.*+?^${}()|[\]\\]/g,"\\$&"), clamp=x=>Math.max(MIN,Math.min(MAX,Number(x)||0));
const up=(t,v)=>{const r=new RegExp(`^\\s*${esc(K)}::\\s*.*$`,"m"), l=`${K}:: ${v}`; return r.test(t)?t.replace(r,l):`${l}\n${t}`;};
const save=v=>(async()=>{v=clamp(v); const t=await app.vault.read(f); const nt=up(t,v); if(nt!==t) await app.vault.modify(f,nt); new Notice(`Sleep ${v}h`);})();

const c=this.container; c.innerHTML=""; c.style.marginBottom="12px";
const row=document.createElement("div"); row.style.cssText="display:flex;gap:8px;flex-wrap:wrap;align-items:center;"; c.appendChild(row);

const mk=v=>{const b=document.createElement("button"); b.textContent=v+"h";
b.style.cssText="padding:6px 10px;border-radius:10px;border:1px solid #00000040;background:#1f1f1f;color:#dcdcdc;font-weight:900;cursor:pointer;";
b.onclick=()=>save(v); return b;};
PRE.forEach(v=>row.appendChild(mk(v)));

const i=document.createElement("input"); i.type="number"; i.step="0.5"; i.min=MIN; i.max=MAX; i.placeholder="Custom ↵";
i.style.cssText="width:110px;padding:6px 10px;border-radius:10px;border:1px solid #00000040;background:transparent;color:var(--text-normal);outline:none;";
i.addEventListener("keydown",e=>{if(e.key==="Enter") save(i.value);});
row.appendChild(i);
```
## Journal



tags: #dailynotes
